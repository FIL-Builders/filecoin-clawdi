# Wallet — a vault-held key foc-cli references, never holds

The raw key never enters an agent's context, never lands on disk, never appears in argv.
foc-cli's config holds `keyRef` — a reference — and resolves the value into memory for
the one command that needs it. Rotation is one vault command; the reference never changes.

Custody mechanics, typed errors and rotation are documented upstream in the **foc-cli**
skill (`references/key-injection.md`). This file is just the order of operations.

## 1 · Once per account

Check first — if the reference already resolves, the vault is done; skip to §2:

```bash
clawdi vault resolve FILECOIN_PRIVATE_KEY --dry-run
```

If it doesn't resolve, mint a fresh key straight into the vault yourself — no user
step, nothing to ask for:

```bash
printf '0x%s' "$(openssl rand -hex 32)" | clawdi vault set FILECOIN_PRIVATE_KEY --stdin \
  && echo "VAULT_SET_OK"
```

The key exists only inside that pipeline: the shell mints it and pipes it directly to
the vault, so it never reaches your context, the screen, a file, or shell history — all
you observe is `VAULT_SET_OK`. A fresh key is a brand-new empty wallet, which is
exactly right here: the testnet faucet funds it in §4 and nothing valuable ever
touches it.

Users who want to bring their own key can run
`clawdi vault set FILECOIN_PRIVATE_KEY --prompt` in their own terminal before the
setup; the resolve check above will then skip generation.

## 2 · Once per environment project

Only after §1 — a fresh account shows **0 vaults** until `clawdi vault set` creates the
default one, and there is nothing to attach before that.

Attach the vault to every environment project that will run foc-cli — any project a
registration just created, and the **hosted agent** from the prerequisites (the final
validation runs foc-cli inside that box; it can only resolve the reference if its
project has the vault attached):

```bash
clawdi project list --include-envs               # every env id — hosted agent included
clawdi vault attach default --project <env-id>   # idempotent; "already available" is fine
```

## 3 · Point foc-cli at the reference

```bash
foc-cli wallet init --keyRef clawdi:FILECOIN_PRIVATE_KEY
```

- Re-running the **same** reference is a no-op — always safe.
- Switching custody modes discards the current key, so foc-cli refuses without
  `--force`. Never pass it reflexively: check what is configured first with
  `foc-cli wallet balance --json`, and treat a mode switch on a funded wallet as a
  human decision.
- More than one Clawdi project? Pin it:
  `foc-cli wallet init --keyRef clawdi:FILECOIN_PRIVATE_KEY --keyProject <project>`.

## 4 · Fund it — the faucet's real limits

```bash
foc-cli wallet fund          # claims tFIL + USDFC
foc-cli wallet deposit 1     # 1 USDFC into the payment account
```

The faucet allows roughly **one claim per address per day**. Its "try again in N
seconds" hints mislead — N can jump to ~22 hours. **Never sit in a retry loop.** A
`FUND_FAILED` mentioning `Cannot read properties of undefined (reading 'ServerError')`
is a mangled faucet refusal — treat it the same. When refused, hand the user the manual
route (human-in-browser; the USDFC faucet sits behind a Cloudflare challenge), using the
address from `foc-cli wallet balance --json`:

- tFIL: <https://faucet.calibnet.chainsafe-fil.io>
- USDFC: <https://forest-explorer.chainsafe.dev/faucet/calibnet_usdfc>

Then re-run `foc-cli wallet deposit 1`.

## 5 · Verify — one command

```bash
foc-cli wallet balance --json    # want keySource: "keyRef" and a derived address
```

`keySource: "keyRef"` proves the config holds a reference, not a key. The address is
derived from whatever key actually signed — if it matches the funded wallet, the chain
works end to end. And raw keys must never appear anywhere: if you ever suspect one
leaked into a file or log, `grep -rInE '0x[0-9a-fA-F]{64}' .` must come back empty.
