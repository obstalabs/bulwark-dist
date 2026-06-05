# Changelog

## [0.4.0] - 2026-06-05

- Remote enforcement: `bulwark ssh user@host --protect <paths> -- <agent>` runs
  an agent on a remote host with the gate on the *remote* kernel (SSH is only
  transport) and consent routed back to the local operator. A protected read is
  denied immediately (the kernel deadline is never held while you decide); a
  prompt surfaces locally and an `allow-session` reply lets the next read of
  that file through. Grants are scoped to identity, session, and policy version.
  Preview of the remote tier — SSH provides transport/auth today; a signed mTLS
  trust channel is the production hardening to come, and the `bulwark` binary
  must be present on the remote host.

## [0.3.0] - 2026-06-04

- Hardened mode: `bulwark run --hardened --allow '<glob>' -- <cmd>` enforces the
  allowlist as a kernel-level Landlock read floor instead of via the fanotify
  supervisor. Crash-safe — the restriction lives in the kernel on the agent
  itself, so a `SIGKILL`, crash, or power loss cannot widen access. `no_new_privs`
  also blocks escalation around the floor. Requires Landlock (Linux 5.13+).

## [0.2.0] - 2026-06-04

- CI / dispatch mode: `bulwark run --deny-all --allow '<glob>' -- <cmd>` runs an
  agent in default-deny allowlist mode — it may read only the granted paths plus
  a runtime base set, every other read is denied, no human in the loop. Inspect
  the base set with `bulwark base-set`.
- Graceful shutdown now fails closed: a `SIGTERM`/`SIGINT` while a read is held
  denies it before exiting. (A hard kill — `SIGKILL`, crash, power loss —
  remains an inherent fanotify limitation that releases the held read as
  allowed; a kernel-enforced floor for that case is in development.)
- Bind-mounted aliases of a protected file are now gated.

## [0.1.0] - 2026-06-03

First release — the Linux read-gate MVP.

- Kernel read gate: `bulwark run --protect <path> -- <cmd>` supervises a process
  tree and denies opens of protected files at the kernel (returns "permission
  denied") before any bytes are read.
- Inode-based protection that defeats symlink and rename tricks.
- Process-tree attribution; only opens by the supervised tree are judged.
- Off-band interactive consent (`--consent socket`) answered by `bulwark
  consent` — held at the kernel, decided by a human, the agent never sees the
  prompt. An agent cannot approve its own access.
- `Bulwark.toml` policy with a default protected-path profile, `allow`/`deny`
  mutation, and `audit` over the decision log.
- A signed-off receipt for every decision (process chain, inode, decision,
  source, reason — never file content).

Linux only; macOS (Endpoint Security) and remote consent are in development.
