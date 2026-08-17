# Seal — encrypt the context, store the ciphertext, index the receipt

The agent that holds the context runs this. It can be any Clawdi-supported agent type
(`claude_code`, `codex`, `openclaw`, `hermes`), local or hosted — the flow is identical.

## 1 · Write it

Write the context to a file in a **temp directory**, never the user's project tree —
plaintext that lands in a repo gets committed. Markdown, whatever length the work
actually is; the heavy bytes are the point of Filecoin.

```bash
CTX=$(mktemp -d)/frontend-design.md   # topic slug as the filename — keeps §3 honest
```

Pick a **topic slug** now: lowercase, hyphenated, the thing another agent would search
for (`frontend-design`, `api-contract`, `data-model`). It goes in the filename, the
receipt, and the search on the other side. One topic, one seal.

## 2 · Seal it

```bash
clawdi read clawdi://default/AGENT_CONTEXT_KEY \
  | openssl enc -aes-256-cbc -pbkdf2 -iter 600000 -salt -pass stdin \
      -in "$CTX" -out "$CTX.enc"
```

The key exists only in that pipe. It is never in argv (so never in `ps`), never on disk,
never in your context.

Record the plaintext digest — it lets the opening agent prove it unsealed exactly what
you sealed, and it is the only integrity check that survives the decryption boundary:

```bash
shasum -a 256 "$CTX" | cut -d' ' -f1     # → sha256 for the receipt
```

## 3 · Store the ciphertext

```bash
foc-cli upload "$CTX.enc" --copies 1 --json   # → pieceCid, dataSetId, size, retrieveUrl
```

Upload the `.enc` file. Confirm a `pieceCid` came back **before touching memory** — never
index a failed upload. `UPLOAD_FAILED` does not always mean nothing was stored: check
`foc-cli dataset details` before retrying, or you pay twice for the same bytes.

## 4 · Index the receipt

One `clawdi memory add`, filled from the upload's JSON. The header — everything before
the second `|` — is what another agent sees first in search results, so it says what this
record is: a sealed receipt, bytes on Filecoin, opened by the filecoin-clawdi-context
skill. A description, never a command — the opener acts on the fields (`pieceCid`,
`enc`, `keyRef`, `sha256`), not on prose:

```bash
clawdi memory add "FILECOIN-MEMORY context | kind: sealed receipt — ciphertext stored on \
Filecoin Onchain Cloud, retrievable by pieceCid with foc-cli, opened by the \
filecoin-clawdi-context skill under keyRef | \
from: <agent_type> | topic: <slug> | tags: filecoin-memory,receipt,sealed,topic:<slug>,\
chain:314159,status:stored | pieceCid: <pieceCid> | dataSetId: <id> | size: <n> bytes | \
created: <YYYY-MM-DD> | retrieveUrl: <url> | enc: aes-256-cbc/pbkdf2/600000 | \
keyRef: clawdi://default/AGENT_CONTEXT_KEY | sha256: <plaintext digest> | \
summary: <one plaintext line — a label, not a leak> | \
verify: download validates bytes against the CID" --category context
```

Four fields earn their place beyond the plain receipt:

| Field | Why the opening agent needs it |
| --- | --- |
| `enc:` | The exact parameters. It decrypts with these, never with guessed defaults. |
| `keyRef:` | Names the key this receipt was sealed under, so the opener resolves that one — not whatever is current. |
| `sha256:` | Proof the unsealed bytes are exactly what you sealed. |
| `summary:` | Lets it judge relevance before spending a download. **Plaintext, in shared memory** — write a label ("frontend design plan, 8 sections, routing + state"), never the substance. |

## 5 · Report

Tell the user the topic slug and the PieceCID, and that the other agent needs nothing
but its own Clawdi session — no key, no file, no copy-paste of the content. If they want
to see it for themselves, the `retrieveUrl` returns the exact bytes to anyone, and they
are noise without the vault.
