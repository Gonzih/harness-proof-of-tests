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

## 12. Soundness analysis

"Cryptographically sound" = every claim a verifier accepts reduces to either a
hardness assumption or a *named* trusted party, with no unexamined gaps.
Decompose the PoT claim into five sub-claims:

### Sound today, given strict implementation

1. **"About exactly this code."** Subject = git tree hash; the contract's
   *full closure* digested (contract file, image digest, lockfiles). Any
   dangling reference — an image by tag, a dep fetched at runtime — is a hole:
   the attested input must be fully determined by the hash. Hermeticity is
   what makes the input digest complete, not just a determinism trick.
2. **"Transcript untampered."** Canonical byte encoding (sorted,
   length-prefixed; hash bytes, never "the JSON") and Merkle **domain
   separation** — distinct leaf/node prefixes (CT's `0x00`/`0x01`), else
   second-preimage attacks can present an internal node as a fake leaf.
   Reduces to SHA-256.
3. **"Identity X made this claim."** DSSE signatures over PAE (kills
   payload-type confusion), Fulcio cert binds the ephemeral key to an OIDC
   identity. Reduction chain: ECDSA → Fulcio issuance honesty → GitHub OIDC
   honesty. Named trusted parties: Sigstore CA + GitHub-as-IdP — not
   removable, but watchable (monitors over Rekor).
4. **"Existed at time T, undeniable."** Verify the Rekor **inclusion proof
   against a signed tree head** (not just "an entry exists") and that signing
   happened inside the short-lived cert window. Remaining weakness:
   split-view; fix with witness-cosigned tree heads, optionally an
   OpenTimestamps anchor of the head into Bitcoin as a trust-nobody floor.

With 1–4 strict, this is sound, full stop: *"GitHub identity X asserted, at
time T, irrevocably and publicly, that tree A produces transcript root R under
contract C."* Forging it means breaking the hash/signature or corrupting
Fulcio/GitHub in a monitor-visible way.

### The unsound link, and the three known ways to close it

5. **"R is what tree A actually produces."** No signature can carry this —
   a signature proves key possession, never computation. Exactly three
   constructions exist:

   - **Trusted hardware (PoT-3)** — sound relative to a chip vendor, buildable
     now. TEE quote of a **reproducibly built** runner image; verdict-signing
     key generated *inside* the enclave, its pubkey in the quote's
     report-data; maintainer nonce in the quote (anti-replay). Reduces to:
     vendor silicon/microcode honest, no side channel. Check the quote's TCB/
     microcode versions against the vendor's current baseline, not just the
     signature — TEE assumptions degrade with each published attack.
   - **zk proofs (PoT-4)** — sound relative to math alone, no trusted party.
     Verifies in ms; proving 10^4–10^6x native, no syscalls. Not practical
     for real suites yet; the ladder's defined top.
   - **Verification games (≈PoT-2)** — bisection disputes + re-execution of
     one step; sound under an *economic* assumption (≥1 honest challenger,
     deterministic replay). Rigorous game theory, but deterrence, not
     cryptography.

   PoT's job is to keep these interchangeable behind the same attestation
   format, so a repo tightens its requirement with one line of policy.

## 13. Identity and standing (trust score)

The tier table in §7.1 assumed "reputation" exists. This section defines it.
**Standing** is a per-identity trust score that gates how much verification a
claim gets: high standing → low spot-check rate → cheap merges. It is the
protocol's memory.

### 13.1 One identity, two signature chains

A PoT identity is a durable public identity that both **authors code** and
**attests tests**. Two chains converge on it:

- **Authorship**: GPG- or SSH-signed commits. The signing key is published on
  the identity's GitHub account (verifiable via the public
  `github.com/<user>.gpg_keys` / `.keys` endpoints), so commit signatures are
  checkable against the account without any new infrastructure.
- **Attestation**: Sigstore keyless (OIDC → the same GitHub account) or the
  same GPG key in the DSSE envelope (offline-friendly fallback, paired with an
  OpenTimestamps stamp).

Both chains resolving to one account is what lets standing mean something:
the entity gaining trust for testing is the same entity whose code you're
merging. A signed commit proves *who wrote it*; a signed attestation proves
*who claims it's green*; standing prices *how much you believe them*.

Key lifecycle is inherited, not reinvented: GitHub key registration dates,
revocation, and rotation apply; a claim signed by a key not registered to the
account at signing time (per Rekor's timestamp) is invalid.

### 13.2 Standing is computed, never stored

Standing is a **pure deterministic function over public data**: the Rekor
log, git histories of participating repos, and published fraud proofs. There
is no standing server, no token, no oracle — any repo, bot, or contributor
recomputes any identity's standing from the same inputs and gets the same
number. (Trustix logic applied to reputation: don't trust the mapping, make
the mapping recomputable.)

Scoring events, by design intent:

| Event | Effect | Why |
|---|---|---|
| Attestation later **spot-checked and matched** | strong + | verified truth — the only heavyweight earner |
| Attestation **corroborated** by an independent identity | weak + | cheap corroboration, discounted for possible collusion |
| **Merged PR**: GPG-signed authorship + attestation + maintainer merge | + | maintainer acceptance is a costly, human signal |
| Correctly reported **FAILED on a canary** | + | proves the harness actually runs what it signs |
| Unchecked attestation | **zero** | claims alone must earn nothing, or spam farms standing |
| Spot-check divergence / canary rubber-stamp | **catastrophic −, effectively permanent** | the fraud proof is portable; every consumer can honor it |

Two dampeners keep it honest:

- **Repo weighting.** Standing earned in a repo is weighted by that repo's
  own standing (age, distinct-maintainer activity, whether *its* attestations
  survive checks). Recursive, PageRank-shaped — kills self-owned repo farms
  where an attacker merges their own attested PRs all day.
- **Independence discount.** Corroboration between identities that
  persistently co-attest the same trees is discounted toward zero; agreement
  only counts to the extent the agreeing parties look independent (different
  repos, different times, no mutual-corroboration cliques).

Plus **decay**: standing is a leaky bucket. A clean 2019 has little bearing on
2026; sustained honest activity is the only way to stay cheap to merge.

### 13.3 Consumption: standing → spot-check rate

The §7.1 tiers become thresholds over standing, set per-repo in the contract:

```toml
[policy.standing]
# score → sampling
tiers = [
  { min = 0.0,  spot_check = 1.00 },   # unknown: every claim re-run
  { min = 0.4,  spot_check = 0.25 },
  { min = 0.7,  spot_check = 0.05 },
  { min = 0.95, spot_check = 0.01 },   # long clean history
]
# which standing computations this repo honors, and at what weight
imports = [
  { scope = "self", weight = 1.0 },              # earned in this repo
  { scope = "global-rekor", weight = 0.5 },      # earned anywhere, discounted
]
```

Portability is **local policy, web-of-trust style**: a maintainer chooses
whether external standing counts and how much. Default posture: your own
repo's history at full weight, the global log at a discount, fraud proofs
honored from anywhere at full weight. Nobody is forced to trust anyone else's
math — but since the function is public and deterministic, honoring someone
else's computation is just choosing their inputs.

### 13.4 The harness axis (advisory today, load-bearing later)

Standing attaches to the human/account identity — the entity that controls
keys and answers for fraud. The harness (`runner.harness` in the predicate)
is self-reported and forgeable, so harness-level reputation can only be
**advisory**: an aggregate "attestations produced by Claude Code 2.x survive
spot-checks at rate r" statistic, usable as a weak prior for brand-new
identities, never as a substitute for checks.

The upgrade path: **harness vendor countersignatures**. If the harness vendor
(Anthropic, OpenAI, …) countersigns attestations produced by unmodified
builds of their harness — the App Attest pattern applied to coding agents —
harness identity becomes cryptographically load-bearing, and a new user
running a reputable harness could start above T0. That requires vendor
participation and integrity-attestation of the harness itself; the predicate
field and the DSSE envelope's multi-signature support already leave room for
it. **Gap: no harness vendor ships this today.**

### 13.5 Attacks on standing itself

| Attack | Countered by |
|---|---|
| Farm score on self-owned repos | repo weighting (recursive, maintainer-diversity term) |
| Collusion ring mutual corroboration | independence discount; corroboration is the weakest earner anyway |
| Build standing honestly, then burn it on one poisoned PR | standing only lowers *test* re-verification, never review (§9 last row); catastrophic slash makes the burned identity unrecoverable — the attack costs its whole history for one shot |
| Buy/steal a high-standing account | key rotation events and dormancy discontinuities lower standing; decay limits the loot |
| Sybil wash-trading spot-checks | earning requires *someone else's* verified compute (spot-checks, maintainer merges) — externally rate-limited by design |

## 14. Consensus: verification mining

Standing (§13) creates the currency. This section defines how it's mined —
and what kind of consensus PoT actually is.

### 14.1 Verification is permissionless work

Anyone can pick any claim from the public log, re-run its `(tree, contract)`
in the pinned container, and publish a **verification attestation**: same
subject, their own transcript root, their own signature, into the same log.

- **Match** → corroboration. The original claim gains confirmation weight;
  the verifier earns standing.
- **Divergence** → dispute. The Merkle trees bisect the disagreement to
  individual tests; targeted re-runs settle it. The loser — false claimant or
  false challenger — takes the catastrophic slash. Finding a real lie pays a
  **jackpot**: far more standing than a confirmation.

This is the cold-start solution: a zero-standing identity can't get cheap
merges yet, but it can mine — re-executing others' claims is useful work that
requires nothing but compute and honesty. Hash-power is replaced by **useful
test execution**; stake is replaced by **identity history**. Proof-of-useful-
work with no token: the reward is cheaper future merges.

### 14.2 The consensus rule

A claim is **confirmed at weight W** where W = Σ standing of *independent*
(§13.2 independence discount) verifiers whose roots match. Repos set the
threshold in policy:

```toml
[policy.consensus]
# merge eligibility: claimant standing OR confirmation weight
min_confirmation_weight = 1.5
# fresh trees in the merge queue: commit-reveal round among k verifiers
merge_queue_verifiers = 2
```

Not Nakamoto consensus: there is no global state machine, no fork choice, no
longest chain. Each claim independently accumulates confirmation weight —
rebuilder-network corroboration with standing-weighted votes. And it is
**advisory consensus**: it prices verification for the maintainer; it never
overrides them. The 51%-style attack (accumulate majority standing in a
repo's verifier set) therefore buys influence over a *recommendation* —
repo-local trust policy (§13.3 imports) decides whose standing counts, and
the maintainer keeps sovereignty.

### 14.3 The free-rider attack, and why mining stays honest

The central attack on any corroboration scheme: **mirror mining** — copy the
claimed root, sign "confirmed," burn zero compute. Three defenses, layered:

1. **Forced errors (Truebit's move).** The protocol continuously publishes
   bait: claims over real trees with deliberately wrong roots, signed by
   throwaway or maintainer identities. Confirming bait = proof of mirror
   mining = catastrophic slash. Bait is indistinguishable from real claims
   without actually running the tests — so the only safe mining strategy is
   the honest one.
2. **Jackpot asymmetry.** Catching a real divergence pays a multiple of what
   a confirmation pays. Actually running has positive expected value over
   copying even before slash risk.
3. **Commit-reveal in merge queues.** For fresh trees, k verifiers commit
   `H(root ‖ nonce)` before any root is revealed, then reveal. Nothing to
   copy.

### 14.4 What mining does NOT cover

Human review is not mechanically falsifiable — there is no root hash for "I
read this carefully." Review earns standing only through the maintainer-merge
event (§13.2), never through the mining path. Mining rewards exclusively
re-executable work. Keeping that line hard is what keeps the score honest:
every mined point traces to a computation someone else can repeat.

---

**Gap: quantitative deterrence.** (No data exists yet.) The right spot-check
rate per tier, the real base rate of fraudulent attestation, and the standing
function's constants (weights, decay half-life, independence metric) are
unknown until someone runs this. The MVP should log everything needed to tune
them.
