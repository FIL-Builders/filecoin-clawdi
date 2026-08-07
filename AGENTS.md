# filecoin-clawdi — agent operating rules

One skill ([skills/filecoin-clawdi-setup](skills/filecoin-clawdi-setup/SKILL.md)) turns
Clawdi + Filecoin Onchain Cloud into provable agent memory. Three primitives:
**hash** (PieceCID = hash of the bytes), **store** (Filecoin warm storage + PDP proof),
**recall-by-hash** (Clawdi memory holds tiny receipts; download validates bytes against
the CID).

## Rules

1. **Storage/payments: foc-cli ONLY** — never the raw Synapse SDK. Reference:
   `foc-cli --help` or the upstream skill
   (`npx skills add https://github.com/FIL-Builders/foc-cli --skill foc-cli`), whose
   `references/key-injection.md` is the authority on key custody.
2. **Calibration testnet (chain 314159) only.** Mainnet (`--chain 314`) or any
   fund-moving operation on it requires explicit human confirmation — never chain them
   autonomously.
3. **Keys are references.** foc-cli holds `wallet init --keyRef
   clawdi:FILECOIN_PRIVATE_KEY` and resolves it per command — nothing at rest. Never
   ask for, print, or handle a raw private key. Never pass `--force` to `wallet init`
   reflexively — it discards whatever key is configured, which may be the only copy.
4. **Heavy bytes go to Filecoin; Clawdi memory gets ONLY the small receipt.** Keep
   shared memory small and curated.
5. **A download that validates against the CID is the only accepted proof of storage.**
   If recall can't find an exact topic receipt, say so — never substitute a near match
   silently.
6. **Never write secrets** (keys, tokens) into files, memory records, or chat. The
   `clawdi://` reference string is safe; the value it resolves to never is.
