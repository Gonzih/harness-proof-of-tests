# Research: Supply-Chain Attestation Ecosystem (2025–2026)

Context: designing a "proof that tests were run and passed" protocol for
contributor-side CI (an AI harness runs the maintainer's test suite on the
contributor's machine and submits verifiable evidence with the PR).

## 1. in-toto Attestation Framework

Three nested layers: an outer **DSSE envelope** (signing), a **Statement**
(binds predicate to artifacts), and a **Predicate** (the actual claim). The
Statement binds to artifacts purely by cryptographic digest — subjects are
matched by digest regardless of content type or name, so an attestation about
"commit abc123" travels independently of any registry or filename.

**Statement v1 format** (<https://github.com/in-toto/attestation/blob/main/spec/v1/statement.md>):

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [{"name": "<optional>", "digest": {"sha256": "<hex>"}}],
  "predicateType": "<URI>",
  "predicate": { }
}
```

**DSSE envelope** (<https://github.com/secure-systems-lab/dsse>):

```json
{"payload": "<b64(statement)>", "payloadType": "application/vnd.in-toto+json",
 "signatures": [{"keyid": "<optional>", "sig": "<b64>"}]}
```

Signatures are computed over PAE, not raw payload:
`PAE = "DSSEv1" + SP + LEN(payloadType) + SP + payloadType + SP + LEN(body) + SP + body`
— length-prefixing plus type inclusion prevents payload-type confusion and
append/prepend attacks.

**Vetted predicate types** (<https://github.com/in-toto/attestation/tree/main/spec/predicates>):
SLSA Provenance, SLSA VSA, Link, SPDX2/3, CycloneDX, SCAI, Release, Runtime
Traces, Reference, Vulns, Simple Verification Result — and, directly relevant,
**Test Result**.

**Test Result predicate** — `https://in-toto.io/attestation/test-result/v0.1`:

- `result` (required): enum `PASSED | WARNED | FAILED`
- `configuration` (required): list of ResourceDescriptors describing the test
  setup (e.g. digest of the CI workflow file)
- `url` (optional): link to logs
- `passedTests` / `warnedTests` / `failedTests` (optional): arrays of test names

The spec's own example is a GitHub Actions run attesting a git commit as
subject. This is exactly the "tests passed" claim schema — thin, but it exists
and is the canonical vocabulary to build on.

**Solves:** standardized, artifact-bound, signable claim format.
**Does NOT solve:** whether the claim is true. A test-result attestation is a
signed assertion "identity X says these tests passed against digest Y" —
nothing in the format proves execution occurred.

## 2. SLSA (slsa.dev)

SLSA v1.x is a track/level framework; the Build track (L0–L3) is the mature
one. Threat model: build-pipeline tampering (unauthorized source, compromised
build steps, forged provenance, post-build artifact substitution). Levels
(<https://slsa.dev/spec/v1.1/levels>):

- **L1**: provenance exists (may be unsigned) — catches mistakes, not attackers.
- **L2**: build runs on a **hosted build platform** that itself generates and
  **signs** the provenance; user build steps cannot forge it after the fact.
- **L3**: hardened builds — builds isolated from one another, and **signing key
  material is inaccessible to user-defined build steps**. Defeats insiders
  forging provenance from within a build.

**Provenance predicate** — `https://slsa.dev/provenance/v1`
(<https://slsa.dev/spec/v1.1/provenance>): `buildDefinition` (`buildType` URI,
`externalParameters` = tenant-controlled inputs verifiers must check,
`internalParameters`, `resolvedDependencies`) + `runDetails` (`builder.id` =
URI of the trusted platform, invocation metadata, `byproducts`). The trust
model explicitly separates signer from builder: a verifier trusts a
(signer, builder.id) pair; the builder is the entire trust base.

**Key transferable idea:** SLSA's L2/L3 distinction is exactly the "who is
trusted" axis for a test protocol — L2-style: the CI *platform* attests tests
ran (user can't forge post-hoc but controls workflow content); L3-style: the
test harness runs in a trusted, isolated context the developer can't inject
into, and the signing identity is unreachable from test code. SLSA does NOT
verify semantics — no SLSA "test track" exists as of v1.1.

## 3. Sigstore (cosign / Fulcio / Rekor)

**Fulcio** (<https://docs.sigstore.dev/certificate_authority/certificate-issuing-overview/>):
client obtains an OIDC token (Google/GitHub/Microsoft/CI workload identity),
generates an ephemeral keypair, proves key possession, and Fulcio issues a
**~10-minute X.509 cert** whose SAN is the OIDC subject — email, SPIFFE ID, or
**GitHub Actions workflow identity** (repo, workflow ref, trigger, commit SHA
embedded as extensions). "Keyless" = no long-lived keys; identity is the OIDC
account/workload.

**Rekor** (<https://docs.sigstore.dev/logging/overview/>): public, append-only
Merkle-tree transparency log. Each signing event uploads {signature, cert,
artifact hash}; Rekor returns a Signed Entry Timestamp and inclusion proof.
Verifiers check the signature was logged **while the short-lived cert was
valid** — the log substitutes for revocation/expiry semantics and makes signing
events publicly auditable and non-repudiable (you can monitor Rekor for
signatures issued in your identity's name).

**cosign**: `cosign attest --predicate ... --type ...` wraps any in-toto
predicate in DSSE, signs keylessly, logs to Rekor.

**Solves:** key management (none needed), strong binding of signature →
verified identity, public auditability, timestamping.
**Does NOT solve:** what the identity did. A signature proves *who* asserted
"tests passed", never *that* tests ran.

## 4. GitHub Artifact Attestations

`actions/attest-build-provenance` (and the generic `actions/attest`, which
accepts **arbitrary predicate-type + predicate** — usable for a test-result
predicate) runs inside a workflow with `id-token: write` +
`attestations: write`. Runner OIDC token → Sigstore cert → signs an in-toto
Statement → stores the Sigstore bundle in GitHub's attestation API.
Verification: `gh attestation verify <artifact> --owner ORG`. Public repos use
the Sigstore Public Good instance (public Rekor); private repos use GitHub's
internal Sigstore instance (no public transparency log).

**Runners:** attestation generation works on **both GitHub-hosted and
self-hosted runners** — the OIDC token is issued by GitHub Actions either way,
and the cert identifies the workflow, not the machine. But the SLSA claim
differs: GitHub-hosted runners give SLSA v1 Build **L2** by default; **L3** is
reachable via a **reusable workflow** (provenance then identifies the trusted
reusable workflow, which the calling repo can't tamper with). Self-hosted
runners produce identical-looking attestations, but the machine is
operator-controlled, so the "platform-signed, user-can't-tamper" premise is
weakened.

- <https://docs.github.com/en/actions/concepts/security/artifact-attestations>
- <https://github.com/actions/attest-build-provenance>
- <https://github.blog/enterprise-software/devsecops/enhance-build-security-and-reach-slsa-level-3-with-github-artifact-attestations/>
- <https://docs.github.com/actions/security-guides/using-artifact-attestations-and-reusable-workflows-to-achieve-slsa-v1-build-level-3>

**Solves:** turnkey identity-bound signing; cert cryptographically names the
exact workflow file, ref, and commit.
**Does NOT solve:** workflow content is developer-controlled — a workflow that
runs `echo ok` can attest anything; on self-hosted runners even the execution
substrate is untrusted. GitHub's docs say this explicitly: attestations "are
not a guarantee that an artifact is secure."

## 5. Test-Result-Specific Efforts (thin field — this is the gap)

- **in-toto `test-result` predicate v0.1** — the only standardized schema
  specifically for test outcomes. Minimal, little production adoption; emit via
  `cosign attest` or `actions/attest`.
- **Witness** (<https://github.com/in-toto/witness>, TestifySec → CNCF/in-toto)
  — the closest existing "prove this command ran" tool.
  `witness run -s test -- go test ./...` wraps the command; pluggable
  **attestors** capture evidence: `command-run` (cmd line, exit code,
  stdout/stderr; experimental ptrace-based process tracing), `git`,
  `github`/`gitlab`/`jenkins`/`aws-codebuild` (CI environment identity),
  `environment`, `material`/`product` (file hashes before/after), `sarif`,
  `sbom`, `secretscan`, `lockfiles`, `network-trace`. Test outcome inferred
  from exit code + products. Signs via keys, Sigstore keyless, or SPIFFE/SPIRE;
  RFC3161 timestamps; stores in **Archivista**; verifies with an OPA/Rego
  policy engine ("step `test` must have a command-run attestation, signed by
  identity X, exit code 0"). Strongest prior art: it attests the *execution*,
  not just the *claim*.
- **FRSCA** (<https://github.com/buildsec/frsca>, OpenSSF) — reference secure
  software factory on Tekton + Tekton Chains (signs SLSA provenance via cosign
  with SPIRE/Vault keys). Build provenance, not test results; not production
  ready; Chains admits it can't yet prove a TaskRun wasn't modified outside
  Tekton.
- **SLSA VSA (Verification Summary Attestation)** —
  <https://slsa.dev/verification_summary>: a second-order predicate where a
  *verifier* attests "I checked artifact X against policy P, result PASSED."
  The pattern for a downstream "tests were verified to have passed" claim.
  in-toto's **SCAI** predicate similarly supports evidence-backed attribute
  assertions ("attribute: unit-tests-passed, evidence: <link>").
- **Chainguard** — Enforce does attestation-based policy enforcement (admission
  control on provenance/SBOM/vuln attestations); no test-result product.
  (<https://www.chainguard.dev/unchained/the-role-of-attestations-in-a-secure-software-supply-chain>)

## Synthesis: the trust ladder for "proof of tests"

The ecosystem cleanly solves layers 1–3 and leaves layer 4 open:

1. **Format** (solved): DSSE + in-toto Statement + test-result predicate,
   subject = digest of the exact code tested (commit/tree hash, not branch name).
2. **Identity** (solved): Sigstore keyless — signature provably from a specific
   GitHub account or CI workflow at a specific ref/commit.
3. **Non-repudiation / timing** (solved): Rekor public log — the claim existed
   at time T and can't be silently retracted.
4. **Faithful execution** (open): nothing above proves tests actually ran.
   Existing mitigations map to SLSA's ladder: (a) trust the identity (weakest);
   (b) trust the platform + inspectable workflow definition (L2-style);
   (c) put test execution + signing inside a trusted, isolated harness the
   developer can't influence — reusable workflow (L3-style), Witness-wrapped
   execution with process/network tracing, or ultimately a TEE. Witness is the
   only tool actively targeting (c) for arbitrary commands today; a
   "proof-of-tests" protocol is essentially a hardened, test-aware instance of
   that pattern.
