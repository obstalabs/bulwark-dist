# Changelog

## [0.7.0] - 2026-06-06

- Agent-operable surface (ANCC): `bulwark init` writes a default `Bulwark.toml`;
  `bulwark doctor` reports whether a host can enforce (OS, root/`CAP_SYS_ADMIN`,
  kernel, Landlock) and exits non-zero when a required capability is missing;
  `--format json` added to `audit`, `check`, and `doctor` as the agent interface.
  Ships `docs/SKILL.md`, the machine-readable contract — documenting the clamp
  ratchet: an agent can tighten a clamp freely, but widening or removing one
  routes through off-band consent. An agent can clamp; it cannot un-clamp.

## [0.6.0] - 2026-06-05

- Remote productionization: `bulwark ssh` now answers the consent prompt on the
  local operator's machine (the relay runs locally, not on the remote host), and
  `--deploy <auto|never|scp|dist>` deploys the `bulwark` binary to a remote host
  that does not have it (existing binary, scp when arch-compatible, or the
  matching release verified by checksum). `--auto` still answers non-interactively.

## [0.5.0] - 2026-06-05

- Integrity circuit-breaker: each run records a clean-shutdown marker and the
  inode identity of every protected object. The next run enters tainted mode after
  an unclean restart (`SIGKILL`/crash) or object-identity drift — protected reads
  are denied and the allow-session cache is bypassed until an operator runs
  `bulwark reset`. Bounds the blast radius after an unclean recovery.

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
