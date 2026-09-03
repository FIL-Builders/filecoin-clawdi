# Persistent Agent Memory on Filecoin

Store agent context once. Carry it into any session, model, or framework.

## What this is

An agent working on a plan, a design, or a spec eventually runs low on context or the session ends. Without this, that work is gone, or the next agent (possibly a different framework, possibly a different machine) has to redo it from scratch.

This repo adds a skill that seals an agent's working context, stores it on Filecoin, and lets a different agent retrieve and verify that exact content later, so it can pick up the work instead of starting over. The content lives on Filecoin's sovereign storage, not inside any single company's platform, so it doesn't disappear if you stop using one particular tool.

Works today with Claude Code, Codex, OpenClaw, and Hermes running as connected agents on your machine, on Filecoin's Calibration testnet. A Clawdi Cloud Agent can find a receipt and fetch the bytes, but cannot unseal a context: its runtime has no CLI and no wallet.

## Prerequisites

1. A Clawdi account. Sign up at [clawdi.ai](https://clawdi.ai). A Cloud Agent (a hosted Hermes or OpenClaw, paid) is optional; setup only uses one, if it exists, for the cloud-side recall at the end.
2. One supported local agent to run the setup: Claude Code, Codex, OpenClaw, or Hermes, with Node.js 24 or newer.

Everything else (the Clawdi CLI, foc-cli, the wallet, funding) gets installed and configured by the setup step below. The only manual step in that process is approving the login in your browser: your agent starts it, gives you a link, and you paste back the address the browser lands on. To use your own Filecoin key instead of a freshly minted one, run `clawdi vault set FILECOIN_PRIVATE_KEY --prompt` first.

## Install

### Step 1: Install the skill and start setup

```bash
npx skills add https://github.com/FIL-Builders/filecoin-clawdi --skill filecoin-clawdi-setup -g -y
```

`-g` installs into `~/.agents/skills`, which Codex, OpenClaw, Hermes and most other agents read, and symlinks it for Claude Code, so one install covers every agent on the machine from any directory. Without it the skill lands in the current folder only. A "PromptScript does not support global skill installation" line at the end is harmless noise.

From any directory, open your agent and say:

```text
Set up Filecoin memory.
```

This installs the second skill too. Your agent then checks whether you're logged in to Clawdi. If you are, it goes straight to registering itself, creating a wallet, and funding it, skip to the result below. If not, proceed to Step 2. 

### Step 2: Create your Clawdi account and authorize

You never type a terminal command here. Your agent runs the login and hands you a link.

1. Your agent gives you an authorization URL. Open it, and create an account at [clawdi.ai](https://clawdi.ai) or sign in if you already have one.
2. Approve the authorization. The browser then lands on a `127.0.0.1` address that fails to load. That is expected, nothing is listening there.
3. Copy that full address from the browser's address bar and paste it back to your agent. It finishes the login from that address with `clawdi auth complete`.
4. The link is valid for 10 minutes. If it expired, ask your agent for a new one.

A Cloud Agent is not required. If you want one for the cloud-side recall at the end of setup, the dashboard offers it under New agent, then Deploy on Clawdi: pick Hermes or OpenClaw and review the monthly price. The `clawdi deploy` wizard does the same from the terminal.

With authorization done, your agent picks back up automatically: it registers itself with Clawdi, creates a wallet key inside Clawdi's vault (the raw key is never shown to you or the agent), and funds it on Filecoin's Calibration test network. It ends by pointing you at a second agent to recall the greeting: another agent on the same machine verifies the bytes with foc-cli; a Cloud Agent can only fetch them over HTTP.

## Working Example

Once setup is done, seal context on one agent:

```text
Plan the frontend for my dApp, then seal it and hand it off.
```

The agent finishes the plan, encrypts it, uploads the encrypted file to Filecoin, and leaves a small pointer in Clawdi's shared memory describing where it is and how to unlock it.

Open a different agent, a different framework, a different machine, doesn't matter as long as it's a connected agent logged in to the same Clawdi account and set up with this skill, and retrieve it:

```text
Pick up the frontend-design context and plan the backend against it.
```

It finds the pointer, downloads the file, checks it against the original fingerprint, decrypts it, and plans the backend against the actual frontend decisions, not a reconstruction of them. No key, file, or content passed directly between the two agents at any point.

## How it works

1. **Seal.** The sending agent encrypts the content with a key that lives in Clawdi's vault. The key is referenced, never shown, and never leaves the vault.
2. **Store.** The encrypted file is uploaded to Filecoin. The address returned is the hash of those exact bytes, so the address and the content are locked together.
3. **Point.** A pointer, under a kilobyte, is left in Clawdi's shared memory. It says where the content lives and how to unlock it. It is not the content itself.
4. **Open.** The receiving agent finds the pointer, downloads the file, and checks the bytes against the original hash before doing anything else. Only then does it decrypt, using the same vault reference. A mismatch stops the process, it never guesses or substitutes a near match.

A revised context is never edited in place. Sealing it again creates a new pointer, and the old one still resolves to what it always did. That's what makes the context immutable, not just stored.

Sealed content is private, only agents that can resolve the same vault reference can decrypt it, but the storage underneath is independently checkable: anyone can fetch the bytes by URL, a foc-cli download with a configured wallet checks them against the hash, and storage providers must reprove they're still holding the data on a recurring schedule, not just accept it once. Clawdi's shared memory never holds more than the pointer, the content itself always lives on Filecoin.

## The two skills in this repo

| Skill | Purpose |
| --- | --- |
| `filecoin-clawdi-context` | Seal a working context on one agent, open it on another. This is the one you use day to day. |
| `filecoin-clawdi-setup` | One-time wiring: registers your agents, creates the wallet, and stores a small public greeting on Filecoin, left unencrypted on purpose so any agent can look it up, fetch it, and with foc-cli verify it. It's a working proof that storage and retrieval function, not something you reach for once setup is done. |

## Repository layout

```text
skills/
├── filecoin-clawdi-context/      # seal a working context, open it on another agent
│   ├── SKILL.md
│   └── references/
│       ├── seal.md               # encrypt, upload, index the receipt
│       └── open.md               # find, verify, decrypt, use it
└── filecoin-clawdi-setup/        # one-time wiring, plus the verification demo
    ├── SKILL.md
    └── references/
        ├── wallet.md              # vault key reference, funding, verification
        └── greeting.md            # the public demo: upload, receipt, recall
```

## License

Apache-2.0 OR MIT, as declared in each skill's SKILL.md.
