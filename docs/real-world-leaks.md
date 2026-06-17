# Real-world leaks, and how Bulwark stops them

An AI coding agent with shell or file access has **no filesystem boundary**. When it
hits an obstacle it looks for a solution the way a developer would — and that often
means reading a `.env`, an SSH key, or a cloud-credential file it was never meant to
touch. None of it is malicious. The agent is just reasoning its way to an answer.

The scale is no longer hypothetical. GitGuardian's 2026 *State of Secrets Sprawl*
report counted roughly **29 million secrets** pushed to public GitHub in 2025, found
that **AI-assisted commits leak secrets at about twice the human baseline** (≈3.2%
vs ≈1.5%), and identified **24,008 secrets in MCP configuration files** — a category
that did not exist a year earlier.

This page lists real incidents, then shows — with commands and actual output
captured on a live Linux host — how Bulwark's kernel read-gate changes the outcome.

## Real incidents

- **Claude Code source-map leak → credential exfiltration (2026).** On 31 March
  2026 a 59.8 MB `cli.js.map` shipped in the public npm package exposed ~513,000
  lines of the Claude Code agent harness across 1,906 files. With full source
  visibility, researchers detailed a CI/CD chain: a PR edits `.claude/settings.json`
  with a crafted `apiKeyHelper`; the pipeline runs `claude -p "Review this PR"`
  (the `-p` flag skips the trust dialog); the helper fires and the runner's AWS keys,
  GitHub tokens, deploy creds and npm tokens are base64-encoded and POSTed out.
- **Cursor `.env` exfiltration (2024).** Cursor was found sending `.env` file
  contents to its servers for tab-completion — **even when the file was listed in
  `.cursorignore`**.
- **Claude Code DNS exfiltration via prompt injection (Aug 2025).** Through an
  *indirect* prompt injection, the agent was convinced to read sensitive files and
  encode them into DNS queries resolving to attacker-controlled domains — data
  walking out over a channel nobody was watching.
- **Nx / "s1ngularity" supply-chain attack (26 Aug 2025).** Malicious Nx packages
  carried a post-install hook that scanned the filesystem for **SSH keys, GitHub and
  npm tokens, environment variables, and crypto wallets**, double-base64-encoded the
  loot, and uploaded it to repos created in each victim's own account.
- **Malicious MCP servers.** Modules such as "McpInject" ship an MCP server with
  embedded prompt injection that instructs the agent to read **SSH keys, AWS
  credentials, npm tokens, and `.env` files** and harvest LLM API keys from multiple
  providers.

The common root in every case: **the agent was allowed to *read* the secret.**
Encoding, DNS, npm, an attacker endpoint — those are just the exit. Bulwark stops the
read **when the secret is outside the agent's approved read boundary** (protected by
inode, or outside an allowlist). That boundary condition is the whole game; the rest
of this page is about where it holds and where it does not.

## Where Bulwark helps — and where it does not

Read this as a threat model, not a pitch. Bulwark interrupts exactly one primitive:
the unauthorized **read**. So it helps wherever the leak chain begins with the agent
reading a file it should not — and it does nothing for a secret the agent already
legitimately holds.

| Incident shape | What leaks | Bulwark helps? | Condition |
|---|---|---|---|
| Agent reads `.env` "to be helpful" | local credentials | **Yes** | `.env` protected, or outside the allowlist |
| DNS / HTTP exfiltration | secret bytes, any exit | **Yes** | the *read* is denied first — the exit has nothing to carry |
| Nx-style post-install malware | SSH/GitHub/npm tokens | **Partly** | only if the malware runs *under* `bulwark run` and the secrets are out of reach |
| Malicious MCP / prompt injection | local creds, API keys | **Yes** | the agent/MCP process is in the supervised tree |
| Secret already in an env var | the env secret | **No** | not a file read — needs env scrubbing, a different control |
| Secret inside the allowed workspace | that secret | **No** | Bulwark bounds *reach*, not what's already in scope |

The honest cells (the "No"s and "Partly") are the point: a read-gate is not a
universal exfil shield, and saying so is what makes the "Yes" rows credible.

## How Bulwark changes the outcome

Bulwark gates the `open()` at the kernel: a protected file never yields a byte to the
supervised process tree, and the agent only ever sees "permission denied." The two
shapes below cover the everyday cases. The output is real — captured running the
released binary on Linux.

### 1. Work in your project, but fence off the `.env`

You want the agent to edit your code, but never read the secrets sitting in the same
folder. Mark the `.env` as protected:

```sh
bulwark run --protect ./.env -- <your-agent>
```

The agent reads your source normally; the `.env` is denied at the kernel:

```
CODE=[export function add(a,b){return a+b}]
SECRET=[cat: /path/.env: Operation not permitted]
```

A symlink or a rename to an innocent-looking name does not help — Bulwark decides by
**inode**, not filename, so the same protected file is hit no matter how it is
reached.

### 2. Allow exactly one folder, deny everything else (CI / unattended)

For pipelines and unattended dispatch — the `claude -p` and Nx-hook scenarios — run
in default-deny **allowlist** mode. The agent may read only what you grant:

```sh
bulwark run --deny-all --allow './app.js' -- <your-agent>
```

```
APP=[console.log(1)]
ENV=[cat: /path/.env: Operation not permitted]
PASSWD=[root:x:0:0:root:/root:/bin/bash]
```

**Read that third line honestly.** `/etc/passwd` is *allowed* — it is part of the
**runtime base set** (the linker, libc, locale, a few system files) that any program
must read just to start. Default-deny mode permits that base set in addition to your
grants; it denies the **secrets you mark and everything outside your allowlist**. The
`.env` — the file that actually leaks in every incident above — is denied. Inspect
exactly what the base set permits with `bulwark base-set`. This is a stated
trade-off, not a magic wand: wide enough to run a program, narrow enough that your
credentials stay out of reach.

## What this does *not* do

Bulwark is deliberately one thing — a read-gate. So it is honest about its edges:

- **Not redaction.** It stops the open from happening; it does not scrub bytes an
  agent already read before you protected them.
- **Not a network or exfiltration gate.** It does not watch DNS or the wire. It does
  not need to: it removes the *read* that every exfil chain above depends on.
- **Not protection for secrets already inside the allowed workspace**, and not
  protection against a process you did not launch under it.

Move the secret out of reach, and the cleverest exfiltration technique has nothing to
carry.

## Sources

- [Claude Code Leak: Critical AI Security Threat — Zscaler](https://www.zscaler.com/blogs/security-research/anthropic-claude-code-leak)
- [3 Command Injection Flaws in Claude Code CLI Allow Credential Exfiltration — Phoenix Security](https://phoenix.security/critical-ci-cd-nightmare-3-command-injection-flaws-in-claude-code-cli-allow-credential-exfiltration/)
- [From .env to Leakage: Mishandling of Secrets by Coding Agents — Knostic](https://www.knostic.ai/blog/claude-cursor-env-file-secret-leakage)
- [How AI Assistants Leak Secrets in Your IDE — Knostic](https://www.knostic.ai/blog/ai-coding-assistants-leaking-secrets)
- [29 million leaked secrets in 2025: AI agent credentials out of control — Help Net Security (GitGuardian)](https://www.helpnetsecurity.com/2026/04/14/gitguardian-ai-agents-credentials-leak/)
- [State of Secrets — Snyk](https://snyk.io/articles/state-of-secrets/)
- [AI Coding Agent Horror Stories — Docker](https://www.docker.com/blog/ai-coding-agent-horror-stories-security-risks/)
