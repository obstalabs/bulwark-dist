# Bulwark

> The kernel asks before the bytes leave the disk.

Bulwark is the consent layer for AI agents at the filesystem boundary. Launch an
agent under Bulwark; when any process in its tree tries to read a protected file,
the Linux kernel **pauses the read** and asks a human — out of band, over a
channel the agent cannot see or answer — before a single byte reaches the agent.

This repository holds **binary releases**. The source is private.

![platform](https://img.shields.io/badge/platform-Linux-blue)
![license](https://img.shields.io/badge/license-AGPL--3.0-green)

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

- **Linux.** Bulwark uses fanotify permission events; macOS support (via the
  Endpoint Security framework) is in development.
- **Root / `CAP_SYS_ADMIN`.** fanotify permission gating is a privileged
  operation.

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

See `bulwark --help` and the per-command help for policy files (`Bulwark.toml`),
profiles, `allow`/`deny`, `audit`, and `base-set`.

## Install

Download the archive for your architecture from the
[latest release](https://github.com/obstalabs/bulwark-dist/releases/latest),
verify the checksum, and place `bulwark` on your `PATH`:

```sh
tar xzf bulwark-<version>-<arch>-unknown-linux-gnu.tar.gz
sudo install -m 0755 bulwark /usr/local/bin/bulwark
bulwark --version
```

## License

Bulwark's open core is licensed under **AGPL-3.0-or-later**. The enforcement
product — fleet control plane, audit pipeline, macOS daemon, and integrations —
is commercial.

Copyright © 2026 Obsta Labs.
