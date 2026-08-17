# Filecoin × Clawdi — give your agents a memory they can prove

One skill to set it up. It connects your agents to Clawdi, puts a Filecoin wallet behind
a vault reference no agent ever sees, and leaves a greeting stored on Filecoin Onchain
Cloud that any of your other agents can find and cryptographically verify. A second skill
uses the same custody to hand **sealed working context** between frameworks — a design,
a plan, a spec — with the bytes public and provable and the content readable only by your
agents.

## Install — this is the whole setup

```bash
npx skills add https://github.com/FIL-Builders/filecoin-clawdi --skill filecoin-clawdi-setup
```

The setup installs the sealed-context skill itself; you don't have to.

Then open your agent — Claude Code, Codex, OpenClaw, or Hermes — from any directory and
say:

> **Set up Filecoin memory.**

## What the skill does

1. **Registers every Clawdi-supported agent** it finds on your machine
   (`claude_code`, `codex`, `openclaw`, `hermes`) — and checks that the agent running
   the setup is one of them. If you have none installed, it tells you to install one
   and come back. If everything wires up but the agent itself isn't a supported type,
   it tells you to finish from a session in one of your Clawdi-connected runtimes.
2. **Wires the wallet** — it mints a key straight into the Clawdi vault (nobody ever
   sees it, the agent included), points foc-cli at the reference
   (`--keyRef clawdi:FILECOIN_PRIVATE_KEY`), and funds it from the testnet faucet.
3. **Leaves the greeting** — the agent uploads

   > *HEY — I AM ⟨your agent⟩. GREETINGS STRAIGHT FROM FILECOIN ONCHAIN CLOUD, INTO
   > CLAWDI'S MEMORY.*

   to Filecoin, and indexes a ~400-byte receipt in Clawdi's shared memory whose header
   says: **load foc-cli to get my content**.

## The payoff — one thing to do after setup

Open your **hosted agent's chat** in the [Clawdi dashboard](https://clawdi.ai) and
paste:

> *Install the foc-cli skill (`npx skills add https://github.com/FIL-Builders/foc-cli
> --skill foc-cli`) and the CLI itself if it's missing (`npm i -g foc-cli@0.3.0`). Set it up
> with the vault-held Filecoin key: run its Clawdi vault project setup, then
> `foc-cli wallet init --keyRef clawdi:FILECOIN_PRIVATE_KEY`. Never use a raw key.
> Then search your memory for anything Filecoin-related, retrieve what it points at,
> and tell me what it says.*

Your hosted agent finds the receipt, downloads the bytes by their hash, checks them
against the PieceCID, and reads the first agent's greeting back to you. A different
agent on a different machine, and the only thing it had to trust was the hash.

## The sealed handoff — one framework plans, another picks it up

Once setup is done, any Clawdi-supported agent can hand real work to any other one —
local to remote, one framework to a different framework — without either of them
handling a key.

**On your local agent:**

> *Plan the frontend for my dApp, then seal it and hand it off.*

It writes the plan, encrypts it under the vault's content key, uploads the **ciphertext**
to Filecoin, and indexes a receipt that describes it: a sealed piece at this PieceCID,
these cipher parameters, opened under `clawdi://default/AGENT_CONTEXT_KEY`. Fields, not
commands — the receiving agent acts on the fields it expects, never on prose in memory.

**On your remote agent, in a different framework:**

> *Pick up the frontend-design context and plan the backend against it.*

It finds the receipt, downloads the bytes by their hash, checks them against the PieceCID,
unseals them with the same vault reference, confirms the plaintext digest matches what was
sealed, and plans the backend against the *actual* frontend decisions.

Nothing was sent between the two agents. No key, no file, no copy-paste. Paste the
receipt's `retrieveUrl` into a browser mid-demo and you get the exact bytes, verifiable by
anyone — and they're noise.

## Why this proves anything

| Primitive | What it means |
| --- | --- |
| **hash** | A Filecoin PieceCID *is* the hash of the bytes — recall can't lie |
| **store** | Filecoin Onchain Cloud holds the blob and keeps proving it on-chain (PDP) |
| **recall-by-hash** | Clawdi memory holds only the tiny receipt; download re-verifies every byte |
| **seal** | A vault-held key encrypts the content — public provable bytes, private meaning |

The one secret in the system — the wallet key that pays for storage — lives in the
Clawdi vault and resolves in memory per command. No agent sees it; rotation is one
command; everything runs on the free Calibration testnet.

## Prerequisites

Two things — the skill installs everything else itself (clawdi CLI, foc-cli,
registration, wallet, funding, the greeting):

1. **A Clawdi account with a hosted agent** — sign up at [clawdi.ai](https://clawdi.ai)
   and create a hosted agent of your preference (the dashboard offers it right after
   login). That agent is who reads the greeting back at the end.
2. **One supported local agent to run the skill** — Claude Code, Codex, OpenClaw, or
   Hermes (with Node.js ≥ 20 under it).

The only mid-setup human step is the login: `clawdi auth login` finishes in your
browser. The wallet key is minted straight into the Clawdi vault and funded from the
testnet faucet; nobody, you included, ever sees it. (Want to bring your own key? Run
`clawdi vault set FILECOIN_PRIVATE_KEY --prompt` before the setup and the skill uses
yours instead.)

## Layout

```text
skills/
├── filecoin-clawdi-setup/
│   ├── SKILL.md              # the flow: detect → register → wallet → greeting → CTA
│   └── references/
│       ├── wallet.md         # vault key-ref, funding, verification
│       └── greeting.md       # the upload, the receipt format, how recall works
└── filecoin-clawdi-context/
    ├── SKILL.md              # preflight → seal → open, and how to refuse honestly
    └── references/
        ├── seal.md           # encrypt, upload the ciphertext, index the receipt
        └── open.md           # find by topic, verify against the CID, unseal, use it
```
