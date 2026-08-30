# call-agent

[한국어](./README.ko.md) · **English**

> One-line summary: a single "delegation skill" (`call-agent`) that lets the AI CLI you're currently using (e.g. Claude Code) **automatically hand work off** to another AI CLI (Codex, Antigravity, Kiro, Claude Code, NotebookLM) when the other tool is better at it.

---

## 0. Glossary — read this first

A few terms used throughout this README:

| Term | What it means |
|---|---|
| **Agentic CLI** | An AI tool that runs in your terminal. When you tell it "do X," it reads files, edits code, and runs commands on its own. Examples: Claude Code, OpenAI Codex CLI, Google Antigravity (`agy`), AWS Kiro CLI (`kiro-cli`). |
| **Host** | The CLI you have open and are talking to right now. Usually Claude Code or Codex. |
| **Skill** | A folder containing a `SKILL.md` plus optional scripts. Claude Code / Codex auto-discover skills and load the relevant one based on what you ask. |
| **Router** | A skill whose `SKILL.md` does not do the work itself — it classifies your request and loads exactly one detailed reference file. `call-agent` is a router. |
| **Delegation** | Instead of doing the work itself, the host shells out to **another CLI** and returns the result. This is what call-agent enables. |
| **MCP** | Model Context Protocol — a shared protocol that lets different AI tools talk to the same external servers (files, DBs, web). Comes up in the `kiro` target's cross-registry feature. |
| **RAG** | Retrieval-Augmented Generation — the model retrieves source documents first and uses them as grounded evidence. NotebookLM is built around this. |

---

## 1. Why does this exist?

Several companies released their own agentic CLIs around the same time. They look similar, but **each is genuinely better at different things**:

- **Claude Code** — Strong at editing code, planning, and analyzing large codebases (1M-token context). Cannot generate images.
- **OpenAI Codex CLI** — Has image generation (`gpt-image-2`) and a polished `codex review` for PR-level reviews.
- **Google Antigravity (`agy`)** — Bakes Google Search grounding into the model's reasoning loop; direct access to scientific databases like gnomAD/UniProt.
- **AWS Kiro CLI (`kiro-cli`)** — Headless agentic terminal, natural-language → shell translation, second opinions from AWS Bedrock models.
- **NotebookLM** — Drop in PDFs / URLs / YouTube videos and get faithful RAG answers; can produce audio overviews.

The common annoyance:

> "I'm in the middle of working in Claude Code and I just need one diagram image. Now I have to open another terminal, launch Codex, copy context over… annoying."

**call-agent** removes that friction. Inside Claude Code you say "generate an image of …" and the `call-agent` skill fires, routes the request to Codex, generates the image, and reports the path back. You never left Claude Code.

---

## 2. One skill, intent-based routing

Earlier versions of this repo shipped **five** separate skills (`agy-call`, `codex-call`, …). They are now consolidated into **one** skill, `call-agent`, that does **intent-based routing** — the same single-skill-with-a-router pattern used by skills like `supergoal`.

`call-agent/SKILL.md` is a thin router:

1. It classifies your request against a small table (which target tool, which capability).
2. It loads exactly one detailed file — `reference/<target>/call.md` — and follows it.

Why one skill instead of five:

- **One description** competing for the model's attention at routing time, not five.
- **One place** to add a new target: drop a `reference/<name>/call.md` and add a router row.
- **Host-neutral.** The same skill installs into both Claude Code and Codex. Rule zero in the router: *never call yourself* — inside Claude Code it will not route to `claude`; inside Codex it will not route to `codex`.

### When does `call-agent` auto-fire?

It activates only under one of two conditions. This is intentionally narrow to avoid surprise billing and slow round-trips:

1. **You name the target tool explicitly** — "use agy to search …", "ask codex to …", "have NotebookLM read these PDFs", etc.
2. **You ask for something the host CLI cannot do natively** — e.g. inside Claude Code you ask for image generation → it routes to `codex` because Claude Code has no native image model.

Routine work (`review this code`, `plan this feature`) does **not** trigger delegation. The host is presumed capable.

---

## 3. The targets `call-agent` can route to

| Target | Borrows the powers of | Typical uses | Explicit trigger words |
|---|---|---|---|
| `codex` | OpenAI Codex | High-quality image generation (`gpt-image-2`), explicit `codex review` | "codex" |
| `agy` | Google Antigravity | Web-grounded search, image generation, scientific DBs (gnomAD/UniProt/PubMed), second opinion | "agy", "antigravity", "gemini cli" |
| `kiro` | AWS Kiro CLI (`kiro-cli`) | Natural-language → shell translation, MCP cross-registry, second opinion via AWS Bedrock | "kiro", "kiro-cli" |
| `claude` | Claude Code | 1M-context plan-mode planning, deep large-codebase review (for a non-Claude host) | "claude", "claude code" |
| `notebooklm` | Google NotebookLM | RAG over PDF/URL/YouTube corpora, audio overviews | "notebooklm", "nblm" |
| `gpt-pro` | ChatGPT Pro (web) | Deep-reasoning second opinion packaged as a sanitized bundle for manual paste into ChatGPT Pro (flat-fee subscription, no API cost); optional experimental live browser drive | "gpt-pro", "chatgpt pro", "gpt5 pro" |

Each target's exact invocation, flags, and wrapper scripts live in `skills/call-agent/reference/<target>/`. The router (`skills/call-agent/SKILL.md`) shows the full decision table.

### Model selection

Wrappers pass no hardcoded chat model wherever the peer CLI has its own default
(`codex`, `agy`); `kiro-chat.sh` takes an optional `--model` flag. The `claude`
wrappers default to `opus` and honor `CLAUDE_MODEL` (any alias — `sonnet`, `haiku`,
`fable` — or a full model name). Review prefers `CLAUDE_REVIEW_MODEL`, then
`CLAUDE_MODEL`, then `opus`. The shell preflight probe pins a cheap model;
override it with `CLAUDE_PROBE_MODEL` if the `haiku` alias ever changes.

---

## 4. Install

### 4-0. Quick install (global, one command)

Clone and link the skill into your coding agents' **global** skill dirs in one go:

```bash
git clone https://github.com/cskwork/call-agent && ./call-agent/install.sh
```

That symlinks `call-agent` into **both** `~/.claude/skills` and `~/.codex/skills` (whichever exist) — so any Claude Code or Codex session, in any project, auto-loads it. Open a fresh session and you're done.

**Prefer to let the agent install it?** Paste this into Claude Code or Codex:

> Clone https://github.com/cskwork/call-agent and run its `install.sh` to install the `call-agent` skill into my global `~/.claude/skills` and `~/.codex/skills`, then confirm the skill is discoverable.

Need per-target control (dry-run, uninstall, prerequisites)? See 4-1 → 4-3 below.

### 4-1. Per-target prerequisites

You only need to set up the targets **you actually plan to use**. Not all of them.

| Target | Required binary | Auth |
|---|---|---|
| `agy` | `agy` (Antigravity CLI) | `agy install`, then Google sign-in |
| `kiro` | `kiro-cli` (AWS Kiro CLI) | `kiro-cli login` |
| `codex` | `codex` | `codex login` (ChatGPT) or `OPENAI_API_KEY` env var |
| `notebooklm` | Python 3.10+, `notebooklm` CLI | `notebooklm login` (browser, one-time) |
| `claude` | `claude` (Claude Code) | `claude auth login` or `ANTHROPIC_API_KEY` env var |

### 4-2. Clone

```bash
# from GitHub
git clone https://github.com/cskwork/call-agent
# or from self-hosted Gitea
git clone https://gitea.agentic-worker.store/Donga-AX/cc-agent-call.git

cd call-agent
```

### 4-3. Run the installer

`install.sh` creates a **symlink** from this repo's `skills/call-agent` into the host CLI's skill directory. Because it is linked into **both** `~/.claude/skills` and `~/.codex/skills`, whichever CLI you run can reach the other targets. Since they're symlinks (not copies), a `git pull` here is immediately visible to the host.

```bash
./install.sh                # link call-agent into both host skill dirs
./install.sh --dry-run      # show what would be linked, don't actually link
./install.sh --uninstall    # remove the call-agent link (and any legacy *-call links)
```

After install, open a fresh session of your host CLI (Claude Code, etc.) and the skill is auto-loaded.

> Migrating from the old five-skill layout? `./install.sh --uninstall` removes the stale `agy-call` / `codex-call` / … symlinks too, then `./install.sh` links the single `call-agent`.

---

## 5. First run — 5-minute walkthrough

Scenario: **You're working in Claude Code and you need a hero illustration for a README.**

```text
> Generate an abstract illustration for the top of my README. 1:1 ratio.
```

Claude Code cannot generate images natively. The `call-agent` skill fires and:

1. Classifies "image generation" → routes to the `codex` target, loading `reference/codex/call.md`.
2. Reshapes your prompt for Codex and shells out to `codex`, which generates the image via `gpt-image-2`.
3. The resulting file path is returned and reported back in the Claude Code thread.

You never switched terminals.

Other examples:

- "Have agy search 'gpt-image-2 pricing'" → routes to `agy`, answered with Google Search grounding.
- "Bundle these 5 PDFs in NotebookLM and ask about 'the Section 3 conclusion'" → routes to `notebooklm`, builds the corpus and queries.

---

## 6. Why the trigger policy is conservative

The router fires delegation only on an explicit tool name or a real host-capability gap. The reasons:

- Calling an external CLI usually spends a separate token / credit balance.
- Outsourcing work the host already does well makes responses slower.
- Frequent automatic hand-offs make it hard for you to trace what your tool is actually doing right now.

So the keywords are deliberately narrow — delegation fires only when it clearly pays off, and never to the host CLI itself.

---

## 7. Tests

Sanity-check the install. The suite runs each target's smoke test in turn:

```bash
./tests/run-all.sh                 # L0 + L1: binary present, --help works, scripts parse
RUN_L2=1 ./tests/run-all.sh        # adds L2: real round-trip prompts (uses credits)
RUN_L3=1 ./tests/run-all.sh        # adds L3: core features (image gen, etc.) (uses credits)
RUN_L4=1 ./tests/run-all.sh        # adds L4: long-running async jobs (codex) (uses credits)
```

A target whose CLI isn't installed is reported as **SKIP**, not FAIL — so a
partial install (e.g. only `codex` + `claude`) still ends in `RESULT: OK`.
The suite only fails on a real error.

Per-target smoke tests:

```bash
./skills/call-agent/reference/agy/tests/smoke.sh
./skills/call-agent/reference/codex/tests/smoke.sh
# ... etc.
```

L2 and L3 hit real external models and may incur cost. For CI, leave only L0/L1 enabled.

---

## 8. FAQ

**Q. I don't use a host CLI — can't I just use the external CLI directly?**
Yes. This repo is only useful when you want to **stay inside the host CLI's flow** while occasionally calling out. For standalone use, just run the target CLI directly.

**Q. I don't want the skill auto-firing.**
Disable `call-agent` in the host CLI's settings, or simply run `./install.sh --uninstall` to drop the symlink. The repo itself can stay in place.

**Q. Can I add a brand-new CLI (e.g. some next-gen tool)?**
Yes, and it's now a two-step edit — no installer change needed:
1. Create `skills/call-agent/reference/<new-name>/call.md` (invocation details, preflight, scripts).
2. Add one row to the **Route** table in `skills/call-agent/SKILL.md`.

**Q. What about security?**
This repo stores no tokens. Each CLI authenticates the standard way for that tool (config file or env var).

---

## 9. License

MIT
