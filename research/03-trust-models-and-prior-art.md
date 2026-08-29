# Research: Trust Models & Prior Art for Contributor-Side Test Proof

## Framing

The claim "I ran your test suite on my machine and it was green" is a claim
about **execution on hardware the verifier doesn't control**. Every prior-art
family attacks this from one of four angles:

- (A) **determinism + independent re-execution** (reproducible builds, Trustix)
- (B) **trusted infrastructure** (remote execution, TEEs, Sigstore/CI attestation)
- (C) **identity + accountability** (signatures, transparency logs, reputation)
- (D) **economics** (spot checks, stakes, challenge windows)

None achieves "cryptographic proof of local test execution" without one of:
trusted hardware, re-execution by someone, or economic deterrence. Cheap
verification levers are marked ▶.

## 1. Reproducible builds & rebuilder networks

reproducible-builds.org premise: make build output a *deterministic function*
of source inputs, so any third party can rebuild and compare bit-for-bit.
Verification is not a proof — it is **corroboration by independent
re-execution**. `rebuilderd` (kpcyrd, powers reproducible.archlinux.org)
dispatches rebuilds to workers, marks packages GOOD/BAD, uses diffoscope to
explain divergences. rebuilderd grew in-toto link attestation support;
`pacman-bintrans` logs package signatures in Rekor.

**Proves**: "N independent parties, given the same inputs, got the same hash."
Trust is *decentralized*, not eliminated.

**Transferable insight: determinism converts 'trust me' into 'check me'.** If
a test run is deterministic (pinned toolchain, hermetic env, seeded RNG, no
wall-clock/network), a signed transcript hash is *falsifiable* — any
spot-checker who re-runs and diverges has caught a lie. Without determinism, a
"proof of green" is unfalsifiable even in principle. ▶ Cheap verification =
re-run only a *sample*, like rebuilders sample the package universe.

- <https://reproducible-builds.org/tools/> · <https://github.com/kpcyrd/rebuilderd>
- <https://github.com/kpcyrd/pacman-bintrans> · <https://arxiv.org/pdf/2505.21642>

## 2. Trustix, Nix caches, Bazel remote cache — closest structural analog

**Trustix (Tweag, NLnet-funded).** Each independent builder maintains a
**Merkle-tree append-only log mapping build-input hash → build-output hash**,
signs log heads, publishes them. A client aggregates multiple builders' logs
and applies a **local trust policy** ("trust if ≥M of N builders agree";
weight by reputation). Non-agreement surfaces non-reproducibility or
compromise. Contrast with Nix status quo: cache.nixos.org is one ed25519 key =
single point of failure. Status: prototype, dormant (nix-community) — but the
best-articulated "trustless cache" design.

▶ **Direct adaptation**: treat *(repo-tree-hash + toolchain-hash +
test-command)* as build input and *(test transcript hash / pass-fail bitmap)*
as output. Multiple independent contributors running the same suite at the
same commit produce agreeing entries; maintainer "verification" collapses to
comparing hashes across submitters — near-zero cost — with re-run only on
disagreement.

**Bazel remote cache.** CAS is content-addressed (self-verifying); the
**Action Cache is input-addressed — NOT self-verifying**. Bazel issue #4276:
*"there is no way to verify the validity of an AC entry based on its key; you
have to trust that any data you get from the AC is correct."* Hence
`--remote_upload_local_results=true` from dev machines is unsafe; only trusted
CI may write. Bazel's resolution is remote execution on trusted infra.

**This is the problem in miniature, already adjudicated: "test passed" is an
action-cache entry — input-addressed, unverifiable from the value itself.**
Industry answer #1: trusted infra (pay for CI). Answer #2 (Trustix):
cross-check multiple untrusted executors. No answer #3 short of TEEs.

- <https://www.tweag.io/blog/2020-12-16-trustix-announcement/> · <https://github.com/tweag/trustix>
- <https://github.com/bazelbuild/bazel/issues/4276> · <https://bazel.build/remote/caching>

## 3. Git signing, signed pushes, Gerrit

Signed commits (GPG/SSH/gitsign) prove keyholder identity + content integrity
— **nothing about behavior**. Signed pushes add intent ("I intended these
commits at this ref"). **gitsign** does keyless signing via Fulcio/Rekor.
**gittuf** (OpenSSF) layers TUF-style policy over git refs in a signed
in-repo hash chain. **Gerrit's `Verified` label** is the canonical pattern: a
change needs `Verified +1`, castable only by a **trusted CI bot** — the
ecosystem institutionalized "the only entity allowed to say 'tests pass' is
infrastructure the project trusts."

Signing tech is the necessary **envelope** (non-repudiable claim, bound
identity, transparency-loggable) — and gives *accountability* (you know
exactly who lied), the input that makes economic schemes work.

- <https://people.kernel.org/monsieuricon/signed-git-pushes> · <https://github.com/sigstore/gitsign>
- <https://openssf.org/blog/2024/01/18/introducing-gittuf-a-security-layer-for-git-repositories/>

## 4. "AI agent ran the tests" — current landscape (2026)

Context: the AI-PR flood is a named crisis — curl killed its bug bounty
(Jan 2026, ~20% AI slop); Jazzband shut down citing AI spam; GitHub publicly
considering per-repo PR kill switches; OpenSSF has an open workstream
(ossf/wg-vulnerability-disclosures#178). **Demand for this mechanism is
documented and rising.**

- **Agent-signed evidence bundles** (agentmark.dev; harness patterns emitting
  diff + command log + transcript, signed as DSSE): proves *the harness
  operator's key vouches for this transcript*, NOT that commands executed as
  transcribed. Self-attestation with a nicer envelope.
- **Agent passports** (aport.io, agent-passport-standard, etc.):
  identity/authorization/reputation layers — none solves execution integrity.
- **Academic**: arXiv 2608.00801 (hardware-rooted attestation for AI-agent
  evidence: RATS + signed Action Evidence Packages, TPM-bound; attests
  platform state, not semantic test success); arXiv 2607.05397 (Proof of
  Execution for governed agent actions); arXiv 2605.21089 (evidence-driven
  trustworthy CI); arXiv 2607.26819 (agents routinely don't follow projects'
  contribution policies).
- **No shipped harness feature** in Claude Code / Codex / Devin produces
  third-party-verifiable *execution* proof.

- <https://www.theregister.com/software/2026/02/03/github-ponders-kill-switch-for-pull-requests-to-stop-ai-slop/4334869>
- <https://github.com/ossf/wg-vulnerability-disclosures/issues/178>
- <https://arxiv.org/html/2608.00801v1> · <https://arxiv.org/html/2607.05397>

## 5. Economic / game-theoretic mechanisms — where cheap verification lives

- **Truebit**: solvers post results with a deposit; disputes resolved by a
  **bisection verification game** narrowing to a single step re-executed by
  the arbiter. ▶ Dispute costs O(log n) + one step, not the whole job — analog:
  on challenge, re-run only the test shard whose transcript hashes diverge.
  Second trick: **forced errors** — deliberately inject wrong answers so
  diligent verifiers hit jackpots, solving the **verifier's dilemma** (if
  fraud is rare nobody checks, so fraud becomes safe). Analog: maintainer
  occasionally posts a known-failing canary to detect rubber-stampers.
- **Optimistic rollups**: accept immediately, finalize after a **challenge
  window**; security holds if ≥1 honest verifier exists; fraud slashes a bond.
  Literature is candid that the honest-verifier assumption is fragile without
  explicit incentives (arXiv 2505.24393, 2512.20864).
- **OROCHI / Efficient Server Audit Problem** (arXiv 1709.08501): verifier
  replays a whole request trace **~10x cheaper than original execution** using
  recorded nondeterminism from the untrusted party. ▶ Strongest academic
  support for cheap deterministic replay of a transcript.

**Workable composite** (every piece ships today; the composition doesn't):

1. Hermetic, deterministic test runner → canonical per-test transcript hashes.
2. Harness signs `(commit_hash, env_hash, per-test-hash-vector, exit_status)`
   as DSSE, keyless via Sigstore OIDC, logged to Rekor.
3. Maintainer **optimistically accepts** but re-runs a random x% of PRs and a
   random subset of tests within a PR.
4. Divergent hash = cryptographically attributable false claim → identity
   ban / reputation slash. GitHub identity + Rekor entry is the "stake" — no
   tokens needed; the scarce staked asset is the operator's account/reputation.
5. Disagreement between two independent contributors' logs on the same commit
   = free fraud detection (Trustix consensus).

What it can't prevent: an attacker who actually runs the tests green while
hiding malice elsewhere in the diff — this verifies *"tests ran and passed"*,
never *"code is good."*

- <https://people.cs.uchicago.edu/~teutsch/papers/truebit.pdf>
- <https://arxiv.org/pdf/1709.08501> · <https://arxiv.org/pdf/2401.17555>

## Bottom line

- Nothing existing lets a maintainer verify "tests passed on contributor
  hardware" from the artifact alone. Bazel's AC-poisoning issue is the precise
  negative result.
- Prior art converges on: **determinism → signed transcript hashes →
  transparency log → random spot-check + cross-submitter comparison →
  identity-based slashing.** Every component ships today; the composition for
  AI-agent PRs does not exist. Trustix is the nearest blueprint and is dormant
  — a genuine gap.
- ▶ Cheapest-verification toolkit, ranked: (1) cross-submitter log agreement
  (free), (2) transcript-hash spot check of a random test subset (seconds),
  (3) OROCHI-style batched replay (~10x cheaper), (4) Truebit bisection on
  dispute (one test), (5) full re-run only on challenge.
