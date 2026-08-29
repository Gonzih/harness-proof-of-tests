# Contributing

This is a protocol concept looking for adversaries and builders. Site:
<https://pot.maksim.sh>. Spec: [`CONCEPT.md`](CONCEPT.md). Research grounding:
[`research/`](research/).

## Most valuable contributions right now

1. **Break the threat model.** Open an issue describing an attack the design
   misses — especially around flaky tests, Sybil identities, canary detection,
   or the spot-check economics. Adversarial review > code at this stage.
2. **`pot` CLI MVP** — `run` / `verify` / `spot-check` as a single static
   binary over cosign + Rekor. Spec in `CONCEPT.md §10`.
3. **Transcript canonicalization** per test framework (pytest, cargo test,
   go test, jest, …): canonical byte encoding of per-test results, Merkle
   layout with domain separation (`CONCEPT.md §12.2`).
4. **Harness integrations** — hooks/skills for Claude Code, Codex, etc.:
   "repo has `.pot/` → run `pot run` before opening the PR."
5. **Deterrence modeling** — spot-check rates per reputation tier vs. fraud
   base rates. Simulation or math, both welcome.
6. **PoT-3 runner** — reproducible TEE runner image (TDX / Nitro) with
   in-enclave verdict signing.

## Ground rules

- Issues for design arguments; PRs for spec changes and code.
- Honesty about limits is the house style: this protocol makes lying
  irrational, not impossible — proposals claiming more must say which trusted
  party or hardness assumption they lean on.
- AI-assisted contributions welcome — that's the point — but you (a human
  identity) sign what you submit.
