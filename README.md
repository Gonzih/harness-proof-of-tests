# harness-proof-of-tests

**Site: <https://pot.maksim.sh>** · Contributions wanted — see
[CONTRIBUTING.md](CONTRIBUTING.md), issues and PRs open.

**Problem:** you maintain an open-source project. AI harnesses (and humans)
send you PRs. You don't want to pay for CI or re-run their tests — you want a
verifiable proof, attached to the PR, that *your* test suite was run against
*that exact code* and was green, on *their* compute.

**Answer in one line:** you cannot cryptographically prove execution on a
contributor's laptop with today's hardware — but you can make a false "tests
passed" claim falsifiable, publicly attributable, and economically irrational,
which in practice is what CI trust already rests on.

## Contents

- [`CONCEPT.md`](CONCEPT.md) — the PoT protocol: test contract, Merkle test
  transcripts, DSSE/in-toto attestations signed keylessly via Sigstore and
  logged to Rekor, millisecond maintainer-side verification, and an
  enforcement layer (tiered random spot-checks, Trustix-style cross-submitter
  agreement, Truebit-style canaries). Trust ladder PoT-0 … PoT-4.
- [`research/01-attestation-ecosystem.md`](research/01-attestation-ecosystem.md)
  — in-toto (incl. the existing `test-result` predicate), DSSE, SLSA levels,
  Sigstore (Fulcio/Rekor/cosign), GitHub artifact attestations, Witness.
  What each solves; why none proves execution.
- [`research/02-trusted-execution-and-verifiable-compute.md`](research/02-trusted-execution-and-verifiable-compute.md)
  — TEEs (SGX/TDX/SEV-SNP/Nitro/Apple), verifiable-CI projects, zkVMs
  (RISC Zero/SP1) with honest overhead numbers, TPMs, proof-of-work,
  OpenTimestamps, Truebit. Why consumer hardware can't attest workloads.
- [`research/03-trust-models-and-prior-art.md`](research/03-trust-models-and-prior-art.md)
  — reproducible builds & rebuilder networks, Trustix (the closest analog),
  Bazel's action-cache trust verdict, git signing/Gerrit, the 2026 AI-PR
  crisis, and the game-theoretic toolkit for cheap verification.

## The design in 30 seconds

1. Repo commits a **test contract** (`.pot/contract.toml`): pinned command,
   pinned runner image digest, determinism level. It lives inside the tree
   being attested, so it can't be swapped.
2. Contributor's harness runs the contract hermetically, builds a **Merkle
   transcript** of per-test results, wraps it in an **in-toto/DSSE
   attestation** (subject = git tree hash), signs **keylessly via Sigstore**
   with the contributor's GitHub identity, logs to **Rekor**.
3. Maintainer runs `pot verify`: signature, identity, subject==PR head,
   contract digest, freshness, all-green. Milliseconds, zero CI.
4. Truth is enforced, not assumed: **random spot-checks** (re-run a random
   Merkle shard, rate scaled by reputation tier), **cross-submitter log
   agreement** (free corroboration), and **canary branches** with
   known-failing tests that expose rubber-stampers. A caught lie is a
   portable, signed, transparency-logged fraud proof — identity burned.

Honest agents pay ~nothing (they already ran the tests to write the patch);
cheaters fight a system built to eventually catch and permanently attribute
them. That gradient is the protocol.
