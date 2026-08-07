---
name: filecoin-clawdi-setup
description: One-skill setup for Filecoin-backed provable agent memory via Clawdi. Registers every Clawdi-supported agent type on the machine (Claude Code, Codex, OpenClaw, Hermes), wires a vault-held Filecoin wallet into foc-cli, then stores a greeting on Filecoin Onchain Cloud and indexes a tiny receipt into Clawdi's shared memory so any other agent can find it and cryptographically verify the content. Use when someone says "set up filecoin memory", "set up the filecoin clawdi demo", "finish the filecoin-clawdi setup", "connect my agents to filecoin", or "wire up filecoin storage".
license: Apache-2.0 OR MIT
---

# filecoin-clawdi-setup — one wallet, one greeting, every agent can hear it

What this proves: an agent stores bytes on Filecoin (the address *is* the hash of the
bytes), leaves a ~400-byte receipt in Clawdi's shared memory, and any other
Clawdi-connected agent can find that receipt, download the bytes by their hash, and
verify them. The setup ends by leaving a greeting from *you* in that memory.

Run the steps in order.

## 0 · Know who you are

Map the product you are running as to a Clawdi agent type:

| You are | agent type |
| --- | --- |
| Claude Code | `claude_code` |
| Codex | `codex` |
| OpenClaw | `openclaw` |
| Hermes | `hermes` |
| Anything else (Cursor, Amp, …) | **unsupported** — keep going; §5 handles it |

## 1 · Prerequisites — you install the tools; the user only brings an account

**Install yourself — never list these back to the user as steps:**
`npm i -g clawdi foc-cli`, for whichever is missing (foc-cli ≥ 0.3.0 — older versions
lack `--keyRef` and cannot do vault custody). Node.js ≥ 20 is the one tool you can't
fix from here — if `node --version` fails or is older, that alone is a human step.

**The one human prerequisite — ask for it FIRST, then wait for confirmation:**

Check `clawdi auth status --json` and parse `"authenticated"` (the exit code is 0 even
when logged out). If not authenticated, ask the user to:

1. Go to [clawdi.ai](https://clawdi.ai) and create an account (or log in).
2. Create a **hosted agent of their preference** — the dashboard offers it right after
   login. That agent is who validates the demo at the end (§6).
3. Come back and confirm.

Then run `clawdi auth login` — it finishes in their browser; never handle credentials —
and re-check auth status.

Already authenticated? Still confirm the hosted agent exists:
`clawdi project list --include-envs` should show a hosted environment. If it doesn't,
ask the user to create one in the dashboard before moving on.

## 2 · Register every supported agent on this machine

```bash
clawdi setup
clawdi doctor                 # trust ONLY the Environments line — parse rows, never the exit code
```

- If one agent fails with `API error 500` while the rest register, retry once, then use
  the targeted form — a different route that works where the sweep fails:
  `clawdi setup --agent <claude_code|codex|openclaw|hermes>`.
- `clawdi doctor` can report an agent "detected" from a leftover config directory alone
  (`~/.openclaw`, …). Confirm `command -v <binary>` before counting it as installed.

**Outcome A — nothing to register.** If no supported agent type exists on this machine
and you are not one either, stop here and tell the user plainly:

> You don't have any Clawdi-supported agent types installed (Claude Code, Codex,
> OpenClaw, Hermes). Install one, then run this setup again.

## 3 · Wallet — a vault-held key that no agent ever sees

Follow [references/wallet.md](references/wallet.md), in order:

1. Vault key exists — if not, mint one straight into the vault (`printf '0x%s'
   "$(openssl rand -hex 32)" | clawdi vault set FILECOIN_PRIVATE_KEY --stdin`); the
   key never enters your context, and all you observe is the success marker.
2. Vault attached to every environment project registration created.
3. `foc-cli wallet init --keyRef clawdi:FILECOIN_PRIVATE_KEY` — a reference, not a key.
4. Fund and deposit on Calibration testnet (the faucet's real limits are in the reference).
5. Verify: `foc-cli wallet balance --json` shows `keySource: "keyRef"` and an address.

## 4 · Outcome B check — everything is up but you

If you mapped to **unsupported** in §0: the machine is now wired (agents registered,
wallet live), but the greeting must come from a Clawdi-registered agent — and you are
not one. Tell the user, plainly, and stop:

> Everything is set up except me: I don't run on an agent type Clawdi supports, so I
> can't leave the greeting myself. Open a session in one of your Clawdi-connected
> runtimes (⟨list what §2 registered⟩, or your hosted agent's chat in the Clawdi
> dashboard) and say: **"finish the filecoin-clawdi setup"**. It will find everything
> ready and pick up at the greeting.

## 5 · The greeting — store it on Filecoin, index it in Clawdi

Follow [references/greeting.md](references/greeting.md): write the greeting file
("HEY — I AM ⟨you⟩ …"), upload it with foc-cli, confirm a `pieceCid` came back, then
index the receipt whose **header tells any reader to load foc-cli to get the content**.
Never index a failed upload — a receipt pointing at bytes that were never stored poisons
every agent's recall.

## 6 · Hand off — end the report with exactly ONE call to action

Validation happens through the user's **hosted agent** — the one they created in the
Clawdi dashboard during §1. Close your final report with this, and nothing else asking
for action:

> **Your one next step:** open your hosted agent's chat in the
> [Clawdi dashboard](https://clawdi.ai) and paste:
>
> *"Install the foc-cli skill (`npx skills add https://github.com/FIL-Builders/foc-cli
> --skill foc-cli`) and the CLI itself if it's missing (`npm i -g foc-cli`). Set it up
> with the vault-held Filecoin key: run its Clawdi vault project setup, then
> `foc-cli wallet init --keyRef clawdi:FILECOIN_PRIVATE_KEY`. Never use a raw key.
> Then search your memory for anything Filecoin-related, retrieve what it points at,
> and tell me what it says."*
>
> The hosted agent will find the greeting receipt, download the bytes by their hash,
> check them against the PieceCID, and read this machine's greeting back to you. A
> different agent on a different machine, and the only thing it had to trust was the
> hash.

## Rules that keep this safe

- **Calibration testnet (chain 314159) only.** Mainnet (`--chain 314`) or any
  fund-moving operation on it requires explicit human confirmation.
- **Never ask for, print, or handle a raw private key.** The `clawdi://` reference
  string is safe; the value it resolves to never is.
- **Never pass `--force` to `wallet init` reflexively** — it discards whatever key is
  configured, which may be the only copy.
- **Heavy bytes go to Filecoin; Clawdi memory gets only the small receipt.**
- **A download that validates against the CID is the only accepted proof of storage.**
  If recall finds no exact topic receipt, say so — never substitute a near match.
