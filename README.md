# Bulwark

> The kernel asks before the bytes leave the disk.

Bulwark is the consent layer for AI agents at the filesystem boundary. Launch an
agent under Bulwark; when any process in its tree tries to read a protected file,
the Linux kernel **pauses the read** and asks a human — out of band, over a
channel the agent cannot see or answer — before a single byte reaches the agent.

This repository holds **binary releases**. The source is private.

![platform](https://img.shields.io/badge/platform-Linux%20%26%20macOS-blue)
![license](https://img.shields.io/badge/license-AGPL--3.0-green)
[![ANCC](https://img.shields.io/badge/ANCC-compliant-brightgreen)](https://ancc.dev)

## What Bulwark is

When an agent reaches for a sensitive file — an SSH key, a cloud credential, a
`.env`, anything you marked protected — Bulwark pauses the open at the kernel,
asks you to allow or deny, and records proof of the decision. The agent only
ever sees the result: the file opens, or it gets "permission denied." It never
sees the prompt, the list of protected paths, or that a human was asked.

- **Decides by inode, not filename** — a symlink or a rename with an innocent
  name still hits the same protected file, so the trick doesn't work.
- **Off-band consent you can't trick** — the approval travels over a channel the
  supervised process has no handle on, and an agent is structurally refused from
  approving its own access.
- **A receipt for every decision** — who tried to open what, the process chain
  that led there, allow or deny, and why.

## What Bulwark is NOT

- **Not redaction.** It stops the open from happening; it does not scrub bytes
  already read.
- **Not an authority or approval system for actions.** It governs file reads,
  not what an agent is allowed to *do*.
- **Not a network or exfiltration gate.** It does not watch the wire.
- **Not protection for secrets already inside the allowed workspace**, and not
  protection against a process you didn't launch under it.

It does one thing: gate the read at the kernel.

## Requirements

- **Linux and macOS.** Linux uses the fanotify gate; macOS uses the same local
  gate via Apple Endpoint Security. macOS builds are Obsta-signed and notarized,
  installable with `brew install obstalabs/tap/bulwark` or the direct `.pkg`
  (below).
- **Root** — Linux needs `CAP_SYS_ADMIN` for the fanotify gate (Landlock for
  `--hardened`); macOS needs root plus Full Disk Access for the Endpoint Security
  gate. The macOS kernel gate is a signed component installed alongside the CLI;
  run `bulwark doctor` to check your setup.

## Quick start

```sh
# Deny the supervised command any read of files under ~/.ssh, by inode.
sudo bulwark run \
  --protect ~/.ssh \
  --receipts /tmp/bulwark-receipts.jsonl \
  -- bash -c 'cat ~/.ssh/id_ed25519'        # -> cat: Permission denied
```

With off-band consent, the read is held and you decide:

```sh
# In one terminal — the agent is paused at the kernel waiting for your call:
sudo bulwark run --consent socket --protect ~/.ssh -- claude

# In another — answer the request:
bulwark consent          # shows Process / Path / Reason, then: allow / deny
```

## CI / dispatch — give an agent exactly one path

For pipelines and unattended dispatch, run in **default-deny allowlist mode**:
the agent may read only the paths you grant plus a runtime base set, and every
other read is denied — no human in the loop.

```sh
# Dispatch a triage agent to a production host with read access to the LOGS
# only — never the data directory, never credentials.
sudo bulwark run --deny-all \
  --allow '/var/log/clickhouse-server/**' \
  -- triage-agent --investigate "query timeouts on shard 3"
```

The agent reads the logs and runs normally; it is denied the data directory,
credentials, `/etc/shadow`, and anything else. A program needs to read its
linker, libc, and a few system files just to start, so allowlist mode permits a
**runtime base set** in addition to your grants — inspect exactly what that
allows with `bulwark base-set`. This is a stated trade-off, not a magic wand:
wide enough to run a program, narrow enough that sensitive material stays out of
reach.

### Crash-safe: hardened mode

The same allowlist, enforced as a **kernel-level Landlock floor** instead of via
the supervisor:

```sh
sudo bulwark run --hardened \
  --allow '/var/log/clickhouse-server/**' \
  -- triage-agent --investigate "query timeouts"
```

Hardened mode applies the restriction to the agent process in the kernel and
then becomes the agent — there is no supervisor. The floor cannot be widened by
killing anything: even a `SIGKILL`, a crash, or power loss leaves the agent
denied. (`no_new_privs` also blocks escalation around the floor.) Requires
Landlock (Linux 5.13+). Use this for unattended dispatch where the worst case
includes the supervisor dying at the wrong moment.

### When an agent pulls the lever: the clamp is a ratchet

In a multi-agent fleet, the thing applying the restraint is itself an agent — an
orchestrator dispatching a sub-agent onto a sensitive host wraps it in a Bulwark
read-gate at dispatch. That is why Bulwark is built to be operated by an agent,
not only a human. But handing an agent the lever is only safe if the lever turns
**one way**:

- **Tightening** a clamp — adding a `--protect`, narrowing an allowlist — is
  agent-initiated and free. Adding restriction is always safe.
- **Widening or removing** a clamp routes through the off-band consent path
  (`--consent socket`), where a human answers over a channel the supervised tree
  cannot see or forge. **An agent can clamp; it cannot un-clamp.**

This asymmetry is the whole reason it is safe to let one agent bound another. A
restraint tool whose restraint an agent can quietly loosen is not a restraint.
The principle is general: *agent-native compliance is about who pulls the lever,
not who gets restricted* — and a lever an agent may pull must be a ratchet.

## Integrity circuit-breaker

Each run records its integrity context — a clean-shutdown marker and the inode
identity of every protected file. The next run enters **tainted mode** if the
prior run ended uncleanly (a `SIGKILL`, crash, OOM, or power loss left no
clean-shutdown marker) or if a protected path now resolves to a different inode.

A tainted run denies protected reads by default, and in interactive mode it
re-asks for every protected open instead of trusting a cached approval — no
pre-taint grant survives. The taint is sticky and persists across restarts until
an operator reviews the audit record and acknowledges it:

```sh
bulwark reset   # clears the taint after review
```

This bounds the blast radius after an unclean recovery. A held read at the exact
instant of a hard kill is governed by the kernel's documented behaviour (see
below); hardened mode is the crash-safe answer for that case.

## Remote: let an agent SSH into a server, gated

When an agent runs on a remote host, the read happens on the *remote* kernel — a
local guard sees only encrypted SSH traffic. So Bulwark runs the gate on the
remote machine and routes consent back to you:

```sh
bulwark ssh user@prod-host \
  --protect /etc --protect /var/lib/postgresql \
  -- claude
```

Enforcement is on the remote kernel; SSH is only transport. A protected read is
denied immediately (the kernel deadline cannot be held while you think), and the
prompt — host, path, process chain — surfaces **on your machine**, where you
answer it: the operator loop runs locally, not on the remote host. Your
`allow-session` answer travels back over a separate control channel and lets the
next read of that file through; the agent on the remote host never sees the
prompt and cannot forge it. Grants are scoped to the requester identity, the
session, and the policy version, so they cannot over-authorize. For unattended
dispatch, `--auto <verdict>` answers every prompt the same way (CI).

If the remote host does not have `bulwark`, it is deployed for you:

```sh
bulwark ssh user@prod-host --deploy auto \
  --protect /etc -- claude
```

`--deploy auto` (the default) uses an existing remote binary, else copies the
local one when it is arch-compatible, else downloads the matching release and
verifies its checksum before running. Use `--deploy never` to require an existing
binary, or `scp`/`dist` to force a path.

This is a preview of the remote tier: SSH provides transport and authentication
today; a signed, time-bounded trust channel (mutual TLS) is the production
hardening to come.

See `bulwark --help` and the per-command help for policy files (`Bulwark.toml`),
profiles, `allow`/`deny`, `audit`, and `base-set`.

## Install

### Homebrew (Linux & macOS)

```sh
brew install obstalabs/tap/bulwark
```

On macOS the formula installs the notarized CLI. The kernel gate is a separate
signed Endpoint Security component — run `bulwark doctor` for setup.

### macOS — direct notarized package

For an offline-trusted install (signed, notarized, and stapled), download the
`.pkg` for your architecture from the
[latest release](https://github.com/obstalabs/bulwark-dist/releases/latest):

```sh
# Apple Silicon: bulwark-<version>-aarch64-apple-darwin.pkg
# Intel:         bulwark-<version>-x86_64-apple-darwin.pkg
sudo installer -pkg bulwark-<version>-<arch>-apple-darwin.pkg -target /
bulwark --version
```

Or via Homebrew cask: `brew install --cask obstalabs/tap/bulwark`.

### Linux — release tarball

Download the archive for your architecture from the
[latest release](https://github.com/obstalabs/bulwark-dist/releases/latest),
verify the checksum, and place `bulwark` on your `PATH`:

```sh
tar xzf bulwark-<version>-<arch>-unknown-linux-gnu.tar.gz
sudo install -m 0755 bulwark /usr/local/bin/bulwark
bulwark --version
```

## Editions

**Local enforcement is open. Managed trust is paid.** The line is architectural:
if it runs entirely on your own machines, it is open source; if it depends on
managed trust, identity, fleet policy, or audit infrastructure, it is the
commercial tier.

**Bulwark Core** — AGPL-3.0-or-later, self-managed local enforcement:
the read gate, local off-band consent, the CI allowlist (`--deny-all`), the
crash-safe Landlock floor (`--hardened`), and the peer `bulwark ssh` mechanism
when you own both ends. Local functionality is never gated by a license check.

**Bulwark Pro / Fleet** — the commercial tier, managed trust for teams: the
remote trust channel (mutual-TLS, signed-grant authority), fleet policy
distribution, a centralized tamper-evident audit pipeline, team approval flows,
an operator cockpit, and an SLA on the consent channel. `bulwark ssh` is the open
mechanism; the managed trust around it is the product.

Copyright © 2026 Obsta Labs.
