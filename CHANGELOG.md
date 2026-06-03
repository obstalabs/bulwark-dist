# Changelog

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
