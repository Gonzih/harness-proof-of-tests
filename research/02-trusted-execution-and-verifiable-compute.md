# Research: Trusted Execution & Verifiable Computation (2025–2026)

Use case anchor: an OSS contributor's AI harness runs the maintainer's test
suite on the contributor's own machine and must prove to the maintainer that it
actually ran and passed.

## 1. TEEs

**Intel SGX** — Process-level enclave; remote attestation quote (DCAP/ECDSA)
proves "this exact enclave measurement (MRENCLAVE = hash of loaded code+data)
is running on genuine Intel silicon", signed by keys chained to Intel. Fatal
for this use case: **SGX was removed from consumer Core CPUs at 11th/12th
gen** and now lives only on Xeon server parts
([Intel statement](https://www.intel.com/content/www/us/en/support/articles/000089326/software/intel-security-products.html)).
A contributor's laptop almost certainly cannot produce an SGX quote. An
enclave also can't run an arbitrary test suite directly — needs a LibOS like
Gramine. Dev effort: high; hardware availability on contributor machines:
near zero.

**Intel TDX** — VM-level TEE: the quote attests the measurement of the whole
guest VM image, chained to Intel
([SNPGuard paper](https://arxiv.org/pdf/2406.01186)). Proves "this VM image
with hash H booted on genuine TDX hardware"; everything inside inherits trust
from what the image contains. Xeon-only (Sapphire Rapids+, mostly cloud:
Azure/GCP/Phala). If you're willing to run tests in a cloud CVM instead of on
the contributor's machine, this is the current-best practical TEE — but then
it's rented attested compute, not "the contributor's machine."

**AMD SEV-SNP** — Same shape: signed attestation report ties the launch
measurement of the guest to AMD's cert chain via the AMD Key Distribution
Service ([trust analysis](https://arxiv.org/pdf/2405.01030)). EPYC server
chips only; consumer Ryzen has no SEV. Open tooling still immature (SNPGuard
exists because provisioning+attesting a CVM was painful). Same verdict as TDX.

**AWS Nitro Enclaves** — Hypervisor-signed attestation document with PCR0
(enclave image hash), PCR1 (kernel/bootstrap), PCR2 (application), PCR8
(signing cert)
([AWS docs](https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave-concepts.html),
[Trail of Bits](https://blog.trailofbits.com/2024/02/16/a-few-notes-on-aws-nitro-enclaves-images-and-attestation/)).
Combined with reproducible enclave-image builds, a verifier can check "this
exact code ran in an enclave on AWS"
([AWS blog](https://aws.amazon.com/blogs/web3/verify-enclave-counterparties-with-reproducible-builds-and-cryptographic-attestation-using-aws-nitro-enclaves/)).
Root of trust is AWS itself. Real packaging work (EIF images, no persistent
disk, vsock-only networking), and by definition runs on EC2.

**Apple Silicon / macOS** — Every M-series Mac has a Secure Enclave, but Apple
exposes **no general-purpose code attestation**. App Attest/DeviceCheck attest
app identity (not computation results); Managed Device Attestation (macOS 14+)
attests device posture to an MDM, not workloads
([Apple security guide](https://support.apple.com/guide/security/attestation-process-security-sec97eb9e2f2/web)).
**Verdict: nothing usable for third-party computation attestation on macOS.**

**Cross-cutting reality**: the machines contributors actually own (consumer
Intel/AMD, Apple Silicon) have **no TEE that can attest an arbitrary workload
to a third party**. TEE attestation in 2025–2026 is a server/cloud capability.
Even where hardware exists, the quote proves "measured image H launched" — you
must additionally make the test-runner image reproducible and have it sign its
verdict from inside, or the measurement proves nothing about the test result.

## 2. Confidential-computing CI / verifiable-CI projects

- **GitHub Artifact Attestations / SLSA provenance** (GA since May 2024):
  trust root = GitHub's infrastructure, not hardware. Solves the problem *if
  you trust GitHub's runners* — but that's "run CI on trusted infra," the
  exact thing being avoided. Cannot attest anything on a contributor's machine.
- **in-toto / Witness**
  ([witness-run-action](https://github.com/testifysec/witness-run-action)):
  signed attestations of CI steps. On an untrusted machine this is only "the
  contributor signed a claim" — no hardware root; the contributor can sign
  lies. TPM/enclave-aware roadmap exists but the local-laptop story is
  trust-me signatures.
- **Edgeless Systems**: Constellation (attested confidential K8s) in
  maintenance mode; development moved to **Contrast** (workload-level
  attestation with Confidential Containers)
  ([blog](https://www.edgeless.systems/blog/from-constellation-to-contrast)).
  Cloud CVM infrastructure.
- **Phala / dstack**: dstack (donated to the Linux Foundation 2025) gives a
  container workflow on Intel TDX + NVIDIA GPU TEE with dual remote
  attestation and a public verification chain
  ([overview](https://docs.phala.com/dstack/overview),
  [paper](https://arxiv.org/html/2509.11555v1)). Currently the lowest-friction
  way to run "a container whose execution is hardware-attested." Again: rented
  attested cloud.
- **Enarx**: effectively dormant (repos moved to enarx-archive after Profian's
  shutdown). **Gramine** maintained but inherits SGX's
  dead-on-consumer-hardware problem. **Oasis** (Sapphire/ROFL) attests
  workloads on its own SGX/TDX node network.
- **Gap: no project found that attests CI runs on arbitrary contributor-owned
  hardware.** Everything real either trusts a platform (GitHub) or requires
  server TEEs.

## 3. zkVMs / verifiable computation

Mechanism: compile the workload to RISC-V (RISC Zero, SP1) or WASM (zkWASM),
execute in a proving VM, emit a succinct proof that "program with image-ID H,
on input X, produced output Y / exited 0." Verification is milliseconds and
needs no trust in the prover's hardware — the only technology in this brief
whose proof actually means "the computation was performed correctly," with no
trusted party at all.

Could you prove "this test binary exited 0"? In principle yes — the
theoretically perfect fit. In practice, mostly no:

- Overheads: proving typically **~4–6 orders of magnitude over native** for
  general code; measured case studies show grotesque ratios (59s proof for a
  15µs function — [study](https://arxiv.org/pdf/2508.17518));
  "hundreds of thousands to millions of times slower" is the honest figure
  ([Jolt/a16z](https://a16zcrypto.com/posts/article/building-jolt/),
  [vApps paper](https://arxiv.org/pdf/2504.14809)). A 60s test suite is
  plausibly days-to-months of GPU proving.
- Frontier numbers are specialized: SP1 Hypercube proves ~99.7% of Ethereum
  blocks in <12s — on **16 RTX 5090s (~$100k)**, for workloads dominated by
  cryptographic precompiles ([SP1 Hypercube](https://blog.succinct.xyz/sp1-hypercube/)).
  A pytest run is not an Ethereum block: heavy syscalls, filesystem, threads,
  subprocesses, network — **none of which zkVMs support**. Porting a real test
  environment into a no_std-ish RISC-V guest is a rewrite, not a port.
- **Honest verdict**: feasible today only for small, hermetic,
  pure-computation test kernels compiled to the zkVM target; infeasible for
  "run the maintainer's real test suite." Improving ~an order of magnitude
  every 1–2 years, but the syscall/IO gap is architectural.

## 4. TPM-based attestation

Measured boot extends hashes of firmware → bootloader → kernel into PCRs; a
TPM **quote** is an AIK-signed snapshot of PCR values, chained to the TPM's
Endorsement Key
([MS docs](https://learn.microsoft.com/en-us/azure/attestation/tpm-attestation-concepts)).
Linux IMA can additionally extend measurements of executed files into PCR10.

**Proves**: "a machine with this TPM booted this measured software stack" and,
with IMA, "these file hashes were executed at some point."
**Cannot prove**: that a given userspace process ran with given inputs, what
it did at runtime, its output, or exit code. IMA measures load-time file
hashes, not behavior
([pitfalls](https://sigma-star.at/blog/2026/01/tpm-on-embedded-systems-pitfalls-and-caveats/)).

Practicality: TPMs are the one attestation root actually present on
contributor machines (fTPM on all modern x86; Apple's Secure Enclave is not a
TPM). But making a TPM quote imply "tests passed" requires contributors to
boot a specific measured OS image — "reboot into my appliance to submit a PR"
is socially dead on arrival. As a weak signal it raises forgery effort;
contributor with physical access and root can still defeat it.

## 5. Proof-of-work and blockchain angles

- **PoW proves energy spent, nothing else.** A hashcash stamp over a test log
  prices spam; zero correctness content.
- **OpenTimestamps / blockchain anchoring** proves *existence-at-time*: hash
  committed (via calendar-server Merkle aggregation) into a Bitcoin block
  header, verifiable forever, free ([opentimestamps.org](https://opentimestamps.org/)).
  Useful as cheap tamper-evidence ("this exact log existed before time T, not
  retro-edited"); proves nothing about how the log was produced.
- **Truebit-style verification games**: solver commits to computation + stake;
  challenger plays an interactive bisection game narrowing to a single step
  re-executed by on-chain judges; fraud loses the stake
  ([Truebit paper](https://people.cs.uchicago.edu/~teutsch/papers/truebit.pdf);
  Truebit Verify open beta 2025). **Gensyn's Verde** applies this to ML
  training with bitwise-reproducible operators so disputes localize to one op
  ([Verde](https://arxiv.org/pdf/2502.19405)). Key insight: optimistic
  verification only works when someone can re-execute the disputed computation
  **deterministically** — requires hermetic env, pinned deps, no flaky tests.
  Degenerate case: if the maintainer can re-execute tests to adjudicate, they
  could have just run the tests. Optimistic schemes buy **amortization**
  (spot-check a fraction, slash liars) — pays off across many
  contributors/runs, not for one PR.
- "Proof of useful work" as a consensus primitive remains research-grade; no
  deployed system proves arbitrary computation correctness via PoW.

## Bottom line

**Gap: no mechanism in 2025–2026 proves an arbitrary test suite executed
correctly on unmodified contributor-owned hardware.** Option space, ranked by
realism:

1. **Optimistic verification**: accept + randomly re-verify (spot-check) +
   revoke trust on mismatch. Cheapest true verification is re-execution;
   amortize it.
2. **Rented attested compute**: run the suite in a TDX/SEV-SNP CVM (Phala
   dstack, cloud CVMs) or Nitro Enclave with a reproducible runner image;
   attestation quote + in-enclave-signed verdict is genuinely strong. Defeats
   "local" but preserves "not the maintainer's cost."
3. **Tamper-evidence, not proof**: signed structured test logs +
   OpenTimestamps anchoring + optional TPM quote. Raises forgery effort;
   proves nothing to a determined adversary.
4. **zk proof of exit-0**: correct in principle, infeasible for real suites
   (10^4–10^6x overhead, no syscalls); revisit in a few years.
5. Apple Silicon offers nothing; SGX is gone from consumer chips; TPMs can't
   see userspace behavior.
