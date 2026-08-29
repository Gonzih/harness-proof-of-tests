# PoT: Harness-Signed Proof of Testing

A protocol for open-source maintainers to accept contributor-side CI: the
contributor's AI harness runs the maintainer's test suite on the contributor's
compute and submits a verifiable claim that the tests ran and were green. The
maintainer verifies in milliseconds, pays for zero CI, and retains the ability
to catch and permanently attribute lies.

Status: concept. Research grounding in `research/`.

---

## 1. What is actually provable (honest foundation)

Lens: cryptographic trust analysis. Three findings from the research constrain
every design:

1. **A signature proves who claimed, never what ran.** DSSE + Sigstore +
   Rekor give you a non-repudiable, identity-bound, timestamped claim. Nothing
   in any signature scheme proves the test process executed
   (research/01).
2. **No consumer hardware can attest arbitrary workloads.** SGX is gone from
   consumer chips, Apple exposes nothing, TPMs can't see userspace behavior.
   Hardware proof exists only on rented server TEEs. zk-proving a real test
   suite is 10^4–10^6x overhead and architecturally blocked on syscalls
   (research/02).
3. **The claim "tests passed" is an input-addressed cache entry** — Bazel
   adjudicated this exact problem and concluded it is unverifiable from the
   value alone (research/03, Bazel #4276).

Therefore PoT does not chase "cryptographic proof of execution" on the
contributor's laptop — that is impossible today. Instead it makes lying:

- **falsifiable** — determinism means any re-run that diverges catches the lie
- **attributable** — a lie is a signed, transparency-logged artifact tied to a
  durable identity
- **irrational** — random spot-checks + cross-submitter comparison + permanent
  reputation loss make expected cost of lying exceed cost of just running the
  tests (which the harness had to do anyway to write working code)

This is the reproducible-builds / optimistic-rollup trust model applied to
test execution. It is strictly stronger than what maintainers accept today
(an unverifiable "CI is green" screenshot or nothing at all), and the
enforcement math is favorable precisely because honest behavior is *cheaper*
than cheating for an AI harness that must run tests anyway to iterate.

## 2. Actors

- **Maintainer** — owns the repo, defines the test contract, verifies claims.
- **Contributor harness** — AI coding agent (Claude Code, etc.) on the
  contributor's machine. Runs tests, produces and signs attestations.
- **Verifier** — a tiny stateless check (CLI, pre-merge bot, or a
  seconds-long GitHub Action). Verifying a signature + policy is ~free; the
  point is to never run the *suite* on maintainer compute.
- **Transparency log** — Rekor (public Sigstore instance). Append-only record
  of every claim ever made.
- **Spot-checkers** — anyone who re-executes a claimed run: the maintainer
  occasionally, other contributors' harnesses routinely (§7).

## 3. The test contract (`.pot/contract.toml`)

The maintainer commits a machine-readable contract to the repo. This is the
root of the protocol: it pins *what* must run so the attestation can pin
*that it was this and nothing else*.

```toml
[contract]
version = 1
# Command executed inside the runner container. The harness may not vary it.
command = "cargo test --locked --offline -- -Z unstable-options --format json"

[environment]
# OCI image digest — the hermetic runner. Pinned by digest, not tag.
image = "ghcr.io/acme/proj-test-runner@sha256:9f2a…"
# Network is disabled during the run; deps must be vendored/locked.
network = false
# Env vars allowed through (everything else stripped).
env_allowlist = ["RUST_BACKTRACE"]

[determinism]
# What goes into the transcript hash. Start status-only; tighten over time.
#   status  — per-test pass/fail/skip only (tolerates timing/output noise)
#   output  — + canonicalized per-test output (paths/timestamps scrubbed)
level = "status"
# Tests excluded from the transcript because they are known-flaky.
quarantine = ["net::test_dns_timeout"]

[policy]
# Claims older than this (relative to PR head push) are stale.
max_age_hours = 48
# Minimum determinism level accepted.
min_level = "status"
```

Key property: **the contract is part of the tree being tested.** The
attestation's subject is the git tree hash, which includes the contract, so a
contributor cannot substitute a weaker contract without changing the digest
under review.

## 4. The run: what the harness does

1. Check out PR head, note `commit`, `tree` hashes.
2. Pull the pinned runner image, verify its digest.
3. Execute `contract.command` inside the container: no network, stripped env,
   fixed locale/TZ/seed (`SOURCE_DATE_EPOCH`-style discipline).
4. Parse the structured test output into a canonical **transcript**:

```
transcript := sorted list of (test_id, status[, canonical_output_hash])
transcript_root := merkle_root(transcript entries)
```

Merkle-izing per-test entries is what makes *partial* re-verification work: a
spot-checker can re-run one shard and check its leaves against the root
without re-running the suite.

5. Produce the attestation (§5), sign it, upload to Rekor, attach to the PR.

For an AI harness this is near-zero marginal cost: it already runs the tests
to know its patch works. The protocol only adds "run the *final* iteration
inside the pinned container and sign the result."

## 5. The attestation

Standard in-toto Statement in a DSSE envelope — deliberately boring, so
existing tooling (`cosign`, `gh attestation`, Rekor) works unmodified.

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    { "name": "git-tree", "digest": { "gitTree": "<tree-sha>" } },
    { "name": "git-commit", "digest": { "gitCommit": "<head-sha>" } }
  ],
  "predicateType": "https://pot.dev/attestation/test-run/v1",
  "predicate": {
    "result": "PASSED",
    "contract": {
      "path": ".pot/contract.toml",
      "digest": { "sha256": "<contract-hash>" },
      "command": "cargo test --locked …",
      "image": "ghcr.io/acme/proj-test-runner@sha256:9f2a…"
    },
    "transcript": {
      "determinismLevel": "status",
      "root": "<merkle-root>",
      "counts": { "passed": 412, "failed": 0, "skipped": 3, "quarantined": 1 },
      "url": "<optional: full transcript blob, content-addressed>"
    },
    "runner": {
      "harness": "claude-code/2.x",
      "os": "darwin-arm64",
      "startedAt": "…", "finishedAt": "…"
    }
  }
}
```

Extends the existing in-toto `test-result/v0.1` predicate (result +
configuration + test lists) with the two fields that make verification cheap:
the **contract digest** (binds the claim to exactly what the maintainer
demanded) and the **transcript Merkle root** (enables partial replay).

**Identity/signing**, in order of preference:

1. **Sigstore keyless**: harness triggers an OIDC flow against the
   contributor's GitHub identity → Fulcio ephemeral cert → signature + cert
   logged to Rekor. Claim is provably "GitHub user X, at time T" with zero key
   management. This is the default.
2. SSH/GPG key already registered on the contributor's GitHub account
   (verifiable via `https://github.com/<user>.keys`), plus an OpenTimestamps
   stamp for the timestamp. Offline-friendly fallback.

Rekor gives every claim a public, append-only existence: a contributor can
never deny having made a claim, and anyone can enumerate all claims an
identity has ever signed — the raw material for reputation (§8).

## 6. Verification (the maintainer's side — milliseconds, no CI)

`pot verify <PR>` — runnable as a CLI on the maintainer's laptop, a merge-queue
bot, or a free-tier Action that finishes in seconds:

```
1. signature      DSSE verifies; Fulcio cert chain OK; Rekor inclusion proof OK
2. identity       signer identity passes repo policy (e.g. any GitHub account;
                  or allowlist; or minimum reputation tier)
3. subject        attestation subject tree/commit == PR head
4. contract       predicate.contract.digest == hash of .pot/contract.toml
                  in that same tree; command/image match verbatim
5. freshness      Rekor timestamp within policy.max_age_hours of head push
6. result         result == PASSED, failed == 0, quarantine list ⊆ contract's
```

All six checks are pure computation over the PR + attestation + public log.
**No test execution.** A green `pot verify` badge means: *a durable identity
staked its name, in public, on this exact tree passing this exact contract.*

## 7. Enforcement: making the claim trustworthy

Verification (§6) proves the claim is well-formed and attributable. Three
mechanisms, layered, make it *probably true* — this is where the protocol
actually lives:

### 7.1 Random spot-checks (optimistic verification)

The maintainer re-executes a fraction of claims — full suite, or a random
Merkle shard (re-run 10 of 412 tests, verify those leaves against the signed
root). Sampling rate scales with the submitter's reputation:

| Tier | Identity | Spot-check rate |
|------|----------|-----------------|
| T0   | new / unknown | 100% (claim treated as advisory only) |
| T1   | n clean verified runs | ~25%, random shard |
| T2   | long clean history | ~5%, random shard |
| T3   | maintainer / trusted org | on suspicion only |

A T0 contributor's first PRs cost the maintainer one run each — the same as
today. The savings compound: every clean spot-check buys future cheap merges.
Expected maintainer compute converges toward `rate × suite_cost`, a tunable
knob instead of a fixed bill.

**Catching a lie is terminal**: the diverging transcript + the signed claim +
the Rekor entry form a portable, self-contained proof of fraud. Publish it;
ban the identity; other repos consuming the same log can honor the ban.
The "stake" being slashed is the GitHub identity's standing — scarce enough to
deter, no tokens required.

### 7.2 Cross-submitter agreement (Trustix model — free verification)

Every attestation for `(tree, contract)` lands in the same public log. When
independent identities attest the same input:

- **agreement** = corroboration at zero maintainer cost (this is rebuilder
  -network logic applied to tests)
- **divergence** = automatic, free fraud/flakiness alarm → triggers a targeted
  spot-check, Truebit-style: bisect to the disagreeing test, re-run only that

Deliberate pattern for AI-PR workflows: the maintainer's merge-queue bot asks
one *other* recent contributor's harness to re-attest the head-of-queue tree.
Contributors' harnesses spot-check each other; the maintainer's compute stays
at zero. (Submitting a PR can even require bundling K re-attestations of other
open claims — verification as pay-it-forward.)

### 7.3 Canaries (the verifier's dilemma, solved Truebit-style)

Periodically the maintainer pushes a branch with a deliberately failing test
and requests attestation. Any identity that returns `PASSED` for it is
rubber-stamping without running — instant, unambiguous, publishable proof of
fraud. Cheap to operate, devastating to game: the attester can't distinguish
canaries from real work without actually running the tests, which is the
desired behavior anyway.

### 7.4 Why the economics work for AI harnesses specifically

A human might rationally skip a 40-minute suite. An AI harness *must* run
tests to iterate — the green run exists as a byproduct of doing the work. So
for honest agents the protocol costs ~one extra containerized run + one
signature, while cheating requires deliberately engineering a false
attestation against a system designed to eventually catch and permanently
attribute it. Honest is the cheap path. Protocols win when the incentive
gradient points the right way, not when lying is impossible.

## 8. Trust levels (SLSA-style ladder)

Named levels so maintainers can state requirements in one line
("this repo requires PoT-2"):

- **PoT-0 — Claimed.** Well-formed signed attestation, verifiable identity,
  Rekor-logged. Proves attribution only. (Already far better than a PR that
  says "tests pass 👍".)
- **PoT-1 — Contracted.** PoT-0 + hermetic pinned-image run + contract-digest
  binding + Merkle transcript. Claim is now *falsifiable*: any re-run can
  check it.
- **PoT-2 — Checked.** PoT-1 + submitter is under an active enforcement regime
  (spot-check tier system, canaries, cross-submitter log). Claim is *probably
  true*, with quantifiable assurance = f(sampling rate, history).
- **PoT-3 — Attested execution.** Run executed on infrastructure that proves
  execution: TEE CVM (Intel TDX / AMD SEV-SNP, e.g. Phala dstack or a cloud
  CVM — contributor-paid, ~cents/run) with a reproducible runner image whose
  quote + in-enclave-signed verdict chain to hardware vendors; or a trusted
  platform runner. Cryptographic proof of execution; no longer "the
  contributor's laptop," but still not the maintainer's bill.
- **PoT-4 — Proven (future).** zkVM proof of the test binary's execution.
  Architecturally blocked today (research/02); the level exists so the ladder
  has a defined top as proving costs fall.

## 9. Threat model summary

| Attack | Countered by |
|---|---|
| Fabricate "green" without running | falsifiability (PoT-1) + spot-checks + canaries; permanent attributable proof of fraud |
| Run a weakened suite / wrong command | contract digest is inside the attested tree; command & image matched verbatim |
| Attest an old/different tree | subject = tree hash checked against PR head; Rekor freshness window |
| Tamper with runner image | image pinned by digest in the contract |
| Cherry-pick a lucky flaky run | quarantine list is maintainer-controlled; status-level determinism; divergence on non-quarantined test = alarm |
| Sybil fresh identities to dodge reputation | T0 = 100% spot-check; new identities earn nothing until checked |
| Rubber-stamp other claims (7.2) | canaries hit cross-checkers too |
| Backdate / deny a claim | Rekor append-only log; OpenTimestamps fallback |
| **Malicious code with genuinely green tests** | **out of scope — PoT proves "tests ran and passed", never "code is safe". Review still exists.** |

## 10. MVP plan

Smallest thing that works end-to-end, all off-the-shelf:

1. **`pot` CLI** (single static binary):
   - `pot run` — read contract, run container, build transcript, emit
     statement, sign via `cosign sign-blob`-style keyless flow, upload to
     Rekor, print a compact "proof block" to paste in the PR description
     (or attach via `gh attestation`/artifact API).
   - `pot verify <pr>` — the six checks from §6.
   - `pot spot-check <pr> [--shard n/k]` — partial or full replay + leaf
     comparison; on divergence, emit a portable fraud proof bundle.
2. **Contract**: `.pot/contract.toml` as in §3; start at `level = "status"` —
   pass/fail vectors are already deterministic for most suites, no hermeticity
   heroics needed on day one.
3. **GitHub integration**: a ~30-line Action running `pot verify` (seconds,
   free tier) posting the badge; merge-queue rule requires it. Canary cron.
4. **Harness integration**: a Claude Code skill / hook — "before opening a PR
   against a repo with `.pot/`, run `pot run` and attach the proof." The
   protocol is harness-agnostic; anything that can exec a CLI can comply.
5. Later: reputation index over Rekor entries; cross-submitter re-attestation
   bot; PoT-3 runner image for Phala dstack / Nitro.

## 11. Design non-goals

- Not judging code quality or safety — only "the contract ran green".
- No blockchain beyond existing public logs (Rekor; OpenTimestamps as an
  optional stamp). PoW proves energy, not correctness — excluded on the
  research findings. No tokens: reputation-as-stake is sufficient and free.
- No requirement for exotic hardware at PoT-0..2; the upgrade path to
  hardware proof (PoT-3) is contributor-opt-in.

---

**Gap: quantitative deterrence.** (No data exists yet.) The right spot-check
rate per tier and the real base rate of fraudulent attestation are unknown
until someone runs this. The MVP should log everything needed to tune them.
