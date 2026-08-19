# Persistent Agent Memory on Filecoin

Store agent context once. Carry it into any session, model, or framework.

## What this is

An agent working on a plan, a design, or a spec eventually runs low on context or the session ends. Without this, that work is gone, or the next agent (possibly a different framework, possibly a different machine) has to redo it from scratch.

This repo adds a skill that seals an agent's working context, stores it on Filecoin, and lets a different agent retrieve and verify that exact content later, so it can pick up the work instead of starting over. The content lives on Filecoin's sovereign storage, not inside any single company's platform, so it doesn't disappear if you stop using one particular tool.

Works today with Claude Code, Codex, OpenClaw, and Hermes.

## Prerequisites

1. A Clawdi account with a hosted agent. Sign up at [clawdi.ai](https://clawdi.ai) and create a hosted agent, the dashboard offers this right after login.
2. One supported local agent to run the setup: Claude Code, Codex, OpenClaw, or Hermes, with Node.js 20 or newer.

Everything else (the Clawdi CLI, foc-cli, the wallet, funding) gets installed and configured by the setup step below. The only manual step in that process is logging in, `clawdi auth login` opens your browser. To use your own Filecoin key instead of a freshly minted one, run `clawdi vault set FILECOIN_PRIVATE_KEY --prompt` first.

## Install

### Step 1: Install the skill and start setup

```bash
npx skills add https://github.com/FIL-Builders/filecoin-clawdi --skill filecoin-clawdi-setup
```

From any directory, open your agent and say:

```text
Set up Filecoin memory.
```

This installs both skills in the repo. Your agent then checks two separate things: whether you're authenticated with Clawdi, and whether a hosted agent already exists in your account. Neither implies the other, you can be logged in with no hosted agent yet, or have a hosted agent from a previous session but a new login to do. If both check out, it goes straight to registering itself, creating a wallet, and funding it, skip to the result below, if either is missing proceed to Step 2. 

### Step 2: Create your Clawdi account and authorize

1. Visit [clawdi.ai](https://clawdi.ai) and create an account, or log in if you already have one.
2. Create a hosted agent. The dashboard offers this right after login, pick Hermes or OpenClaw, and a free or paid tier, whichever fits what you need.
3. Once that's done, your agent runs `clawdi auth login` and gives you a URL.
4. Open that URL in your browser and complete the authorization.
5. If the page fails to load after you authorize (common when running in a remote or sandboxed setup), copy the full URL from your browser's address bar anyway and paste it back to your agent. It can complete the login from that URL even though the page itself shows an error.

With authorization done, your agent picks back up automatically: it registers itself with Clawdi, creates a wallet key inside Clawdi's vault (the raw key is never shown to you or the agent), and funds it on Filecoin's free test network.

## Working Example

Once setup is done, seal context on one agent:

```text
Plan the frontend for my dApp, then seal it and hand it off.
```

The agent finishes the plan, encrypts it, uploads the encrypted file to Filecoin, and leaves a small pointer in Clawdi's shared memory describing where it is and how to unlock it.

Open a different agent, a different framework, a different machine, doesn't matter, and retrieve it:

```text
Pick up the frontend-design context and plan the backend against it.
```

It finds the pointer, downloads the file, checks it against the original fingerprint, decrypts it, and plans the backend against the actual frontend decisions, not a reconstruction of them. No key, file, or content passed directly between the two agents at any point.

## How it works

1. **Seal.** The sending agent encrypts the content with a key that lives in Clawdi's vault. The key is referenced, never shown, and never leaves the vault.
2. **Store.** The encrypted file is uploaded to Filecoin. The address returned is the hash of those exact bytes, so the address and the content are locked together.
3. **Point.** A pointer, under 500 bytes, is left in Clawdi's shared memory. It says where the content lives and how to unlock it. It is not the content itself.
4. **Open.** The receiving agent finds the pointer, downloads the file, and checks the bytes against the original hash before doing anything else. Only then does it decrypt, using the same vault reference. A mismatch stops the process, it never guesses or substitutes a near match.

A revised context is never edited in place. Sealing it again creates a new pointer, and the old one still resolves to what it always did. That's what makes the context immutable, not just stored.

Sealed content is private, only agents that can resolve the same vault reference can decrypt it, but the storage underneath is independently checkable by anyone: storage providers must reprove they're still holding the data on a recurring schedule, not just accept it once. Clawdi's shared memory never holds more than the pointer, the content itself always lives on Filecoin.

## The two skills in this repo

| Skill | Purpose |
| --- | --- |
| `filecoin-clawdi-context` | Seal a working context on one agent, open it on another. This is the one you use day to day. |
| `filecoin-clawdi-setup` | One-time wiring: registers your agents, creates the wallet, and stores a small public greeting on Filecoin, left unencrypted on purpose so anyone can look it up and verify it directly. It's a working proof that storage and retrieval function, not something you reach for once setup is done. |

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
