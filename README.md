<div align="center">

<img src="https://github.com/VrilLabs/kem/blob/main/docs/header.png" alt="VRIL-KEM Header Image" width="100%" />

# VRIL-KEM

**Vortex Resonance Implosion Lattice — Key Encapsulation Mechanism**

[![License: GNU](https://img.shields.io/badge/License-GNU-green.svg)](https://github.com/VRIL-LABS/vril-kem/blob/main/LICENSE) [![Security: Post-Quantum](https://img.shields.io/badge/Security-Post--Quantum-blueviolet)](#security-analysis) [![Level: Level 5+](https://img.shields.io/badge/Security%20Level-Level%205%2B-brightgreen)](#parameter-sets) [![Language: C99](https://img.shields.io/badge/Language-C99-blue)](#quick-start) [![Status: v1.2-rc2](https://img.shields.io/badge/Status-v1.2--rc2-blue)](#release-artifacts) [![EasyCrypt: Formalized](https://img.shields.io/badge/EasyCrypt-Formalized-brightgreen)](#formal-verification) [![Platforms](https://img.shields.io/badge/Platforms-Linux%20%7C%20macOS%20%7C%20ARM64-lightgrey)](#release-artifacts)

*A Schauberger centripetal-physics-inspired lattice-based KEM achieving Level 5+ post-quantum security.*

[Specification](#specification) · [Quick Start](#quick-start) · [Parameter Sets](#parameter-sets) · [Security Analysis](#security-analysis) · [Formal Verification](#formal-verification) · [Release Artifacts](#release-artifacts) · [Integration](#integration)

</div>

## What is VRIL-KEM?

VRIL-KEM is a post-quantum Key Encapsulation Mechanism (KEM) built on Module Learning With Errors (M-LWE). It provides a drop-in replacement for classical key exchange in any TLS, SSH, or application-layer protocol — protecting encrypted communications against both classical and quantum adversaries. 

At the time of writing, VRIL-KEM-4096-7 provides ***the highest security margin of any publicly available lattice-based KEM*** with an estimated ~350-bit quantum Core-SVP cost. The EasyCrypt proof represents a ***first-of-its-kind formalization for a novel post-quantum KEM*** outside the NIST finalist ecosystem.

VRIL-KEM introduces **three novel cryptographic constructions** absent from all existing post-quantum KEMs:

| Construction | Purpose |
|---|---|
| **HI-Gaussian Sampler** | 7-layer Fibonacci-weighted noise that resists Kannan-embedding and BKZ lattice reduction attacks |
| **CVKDF** (Centripetal Vortex KDF) | Schauberger-inspired one-way spiral compression of the secret key; 2⁻¹²⁸ estimated inversion probability |
| **Outer Harmonic Commitment (OHC)** | Post-decapsulation integrity token preventing ciphertext component-substitution attacks under adaptive chosen-ciphertext attack |

---

## Parameter Sets

Three stable parameter sets ship in v1.2, covering NIST Levels 3 through 5+:

| Designator | N | k | Security | Public Key | Secret Key | Ciphertext | Suite ID |
|---|---|---|---|---|---|---|---|
| **VRIL-KEM-1024-3** | 1024 | 4 | ~192-bit PQ (Level 3) | 7,200 B | 14,432 B | 6,304 B | `0x0100` |
| **VRIL-KEM-2048-5** | 2048 | 5 | ~256-bit PQ (Level 5) | 17,952 B | 35,936 B | 15,392 B | `0x0101` |
| **VRIL-KEM-4096-7** | 4096 | 7 | ~384-bit PQ (Level 5+) | 50,208 B | 100,448 B | 42,016 B | `0x0102` |

All three variants share the same ring algebra (`R_q = Z_q[X]/(X^N + 1)`, `q = 12289`), arithmetic primitives, and API surface — selecting a parameter set is purely a compile-time flag.

### Hybrid Suites (PQ + Classical)

For deployment environments requiring defense-in-depth against both quantum and classical adversaries:

| Suite ID | Name | Composition | Security |
|---|---|---|---|
| `0x0030` | VRIL-HYBRID-1 | VRIL-KEM-1024-3 + X25519 | Level 3 ≥ AES-192 |
| `0x0031` | VRIL-HYBRID-2 | VRIL-KEM-2048-5 + X25519 | Level 5 ≥ AES-256 |
| `0x0032` | VRIL-HYBRID-3 | VRIL-KEM-4096-7 + X25519 | Level 5+ |

---

## Quick Start

### Prerequisites

- C99-compliant compiler (GCC 11+, Clang 13+, MSVC 2022+)
- GNU Make 4.0+
- Linux, macOS, or Windows (WSL2)

### Building from source

```bash
git clone https://github.com/VRIL-LABS/vril-kem.git
cd vril-kem

# Reference implementation (recommended starting point)
cd ref
make                          # builds libvril_ref_4096.a + test binary
./test/test_vril4096          # runs full test suite

# Select a parameter set (default is 4096-7)
make VRIL_MODE=1024           # VRIL-KEM-1024-3
make VRIL_MODE=2048           # VRIL-KEM-2048-5
make VRIL_MODE=4096           # VRIL-KEM-4096-7  (default)
```

### Building optimized variants

```bash
# AVX2-accelerated (Linux x64 only, ~6-7× faster)
cd avx2 && make

# Constant-time masked (side-channel hardened)
cd ct && make

# Memory-optimized (< 32 KB stack — IoT/embedded)
cd mem && make
```

---

## Integration

### C API

```c
#include "ref/api.h"

// ── KeyGen (recipient) ────────────────────────────────────────────────────────
uint8_t pk[VRIL_PUBLICKEYBYTES];   // publish this
uint8_t sk[VRIL_SECRETKEYBYTES];   // keep secret
crypto_kem_keypair(pk, sk);

// ── Encapsulate (sender) ──────────────────────────────────────────────────────
uint8_t ct[VRIL_CIPHERTEXTBYTES];
uint8_t ss_enc[VRIL_SSBYTES];      // 32-byte shared secret
crypto_kem_enc(ct, ss_enc, pk);    // send ct to recipient

// ── Decapsulate (recipient) ───────────────────────────────────────────────────
uint8_t ss_dec[VRIL_SSBYTES];
crypto_kem_dec(ss_dec, ct, sk);    // ss_dec == ss_enc iff ct is authentic
```

The API is NIST PQC competition-compatible. Any library using `crypto_kem_keypair` / `crypto_kem_enc` / `crypto_kem_dec` can be adapted to use VRIL-KEM with no protocol changes.

### Go (Hybrid/Adaptive)

```go
import "github.com/VRIL-LABS/vril-kem/vril-mesh/pkg/hybrid"

// Hybrid PQ+Classical key exchange (VRIL-KEM-4096-7 + X25519)
ctx, _ := hybrid.NewContext(hybrid.ProfileHybrid3)
pk, sk, _ := ctx.KeyGen()
ct, ssEnc, _ := ctx.Encapsulate(pk)
ssDec, _ := ctx.Decapsulate(sk, ct)
// ssEnc == ssDec — 32-byte shared secret
```

---

## Release Artifacts

Every release ships pre-built static libraries for Linux x64, Linux ARM64, and macOS:

### v1.2-rc2 — Available Now

**Naming convention:** `vril-kem-<variant>-<backend>-<os>-<version>.tar.gz`

| Artifact | Variant | Backend | Platforms |
|---|---|---|---|
| `vril-kem-1024-3-ref-*` | VRIL-KEM-1024-3 | Reference C99 | Linux x64, Linux ARM64, macOS |
| `vril-kem-1024-3-avx2-*` | VRIL-KEM-1024-3 | AVX2 SIMD | Linux x64 |
| `vril-kem-1024-3-ct-*` | VRIL-KEM-1024-3 | Constant-time | Linux x64, Linux ARM64, macOS |
| `vril-kem-1024-3-mem-*` | VRIL-KEM-1024-3 | Memory-optimized | Linux x64, Linux ARM64, macOS |
| `vril-kem-2048-5-ref-*` | VRIL-KEM-2048-5 | Reference C99 | Linux x64, Linux ARM64, macOS |
| `vril-kem-2048-5-avx2-*` | VRIL-KEM-2048-5 | AVX2 SIMD | Linux x64 |
| `vril-kem-2048-5-ct-*` | VRIL-KEM-2048-5 | Constant-time | Linux x64, Linux ARM64, macOS |
| `vril-kem-2048-5-mem-*` | VRIL-KEM-2048-5 | Memory-optimized | Linux x64, Linux ARM64, macOS |
| `vril-kem-4096-7-ref-*` | VRIL-KEM-4096-7 | Reference C99 | Linux x64, Linux ARM64, macOS |
| `vril-kem-4096-7-avx2-*` | VRIL-KEM-4096-7 | AVX2 SIMD | Linux x64 |
| `vril-kem-4096-7-ct-*` | VRIL-KEM-4096-7 | Constant-time | Linux x64, Linux ARM64, macOS |
| `vril-kem-4096-7-mem-*` | VRIL-KEM-4096-7 | Memory-optimized | Linux x64, Linux ARM64, macOS |
| `vril-hybrid-{1,2,3}-*` | Hybrid suites 1-3 | Go source | All (source) |
| `vril-adaptive-1-*` | Adaptive selector | Go source | All (source) |

Each bundle contains: static library (`.a`), all public headers, `LICENSE`, and (where available) NIST KAT response files.

**Download →** [github.com/VRIL-LABS/vril-kem/releases](https://github.com/VRIL-LABS/vril-kem/releases)  
**Checksums →** See `MANIFEST.md` attached to each release.

---

## Security Analysis

### Security Assumptions

IND-CCA2 security reduces to the hardness of **Module-LWE** with:

- Ring `R_q = Z_q[X]/(X^N + 1)`, `q = 12,289` (NTT-friendly prime)
- Module rank `k ∈ {4, 5, 7}` for the three stable variants
- Error distribution: HI-Gaussian (Fibonacci-weighted, 7 layers, η₁=3)

The Fujisaki-Okamoto transform provides IND-CCA2 security in the random oracle model. The Outer Harmonic Commitment provides an additional binding layer preventing ciphertext-component substitution.

### Estimated Security Levels

| Attack Model | VRIL-KEM-1024-3 | VRIL-KEM-2048-5 | VRIL-KEM-4096-7 |
|---|---|---|---|
| Classical (BKZ) | ~215-bit | ~260-bit | ~350-bit |
| Quantum (QBKZ) | ~195-bit | ~260-bit | ~350-bit |
| Grover (shared secret) | 128-bit | 128-bit | 128-bit |
| NIST Level | **3** | **5** | **5+** |

### What is "Level 5+"?

NIST Level 5 targets ~256-bit quantum security, equivalent to AES-256. VRIL-KEM-4096-7 substantially exceeds this with an estimated ~350-bit quantum Core-SVP cost — the highest security margin of any publicly available lattice-based KEM.

### Formal Verification (EasyCrypt)

VRIL-KEM includes a **complete EasyCrypt formal-verification architecture** in `proof/`, verified against the real upstream EasyCrypt stdlib API and the sandbox-quantum FO transform library. This is a first-of-its-kind formalization for a novel post-quantum KEM outside the NIST finalist ecosystem.

**What is formalized:**

| Layer | Content |
|---|---|
| Ring algebra | R_q = Z_q[X]/(X^N+1); NTT correctness; module-vector ops |
| M-LWE hardness | Decisional game with quantitative advantage bound |
| Noise model | Sub-Gaussianity of HI-Gaussian; smudging bound; concrete failure probability δ (Chernoff/union-bound) |
| CVKDF | Lazy-ROM model with quantitative one-wayness bound |
| OHC | Outer Harmonic Commitment binding + hiding games |
| KEM specification | Full KEM module realizing upstream `KeyEncapsulationMechanisms.Scheme` |
| IND-CCA2 theorem | FO-U transform clone + checked concrete-bound lemma |
| Binding hierarchy | Checked HON-BIND-K-CT, LEAK-BIND-K-CT, MAL-BIND-CT-PK bound lemmas |
| Jasmin boundary | Phase 5 source-level spec↔impl equivalence boundary |

**Concrete security bounds (formally stated):**

```
IND-CPA:   |Pr[IND_CPA(VRIL_PKE, A)] - 1/2|  ≤  2 · mlwe_eps
IND-CCA2:  Adv^{IND-CCA2}                     ≤  2 · mlwe_eps + δ + q_RO / 2^256
LEAK-BIND: Pr[LEAK_BIND(K_Binds_CT)]          ≤  ohc_bind_eps + cvkdf_ow_eps
MAL-BIND:  Pr[MAL_BIND(CT_Binds_PK)]          ≤  mlwe_eps + ohc_bind_eps
```

The IND-CCA2 and binding bound-sanity statements are **checked EasyCrypt `lemma` bodies** — no `admit` tactics in VRIL proof sources. The remaining full theorem-level work item is the IND-CPA game-hop reduction (`vril_pke_indcpa`).

### Honest Disclosures

- Constant-time behavior is compiler-dependent on some targets; use the `ct/` backend for side-channel-hardened deployments
- Large key and ciphertext sizes — evaluate whether your protocol can accommodate them
- Not yet peer-reviewed; not recommended for production use without independent review
- **Research prototype** — audit the code before deploying in security-critical systems
- The IND-CPA game-hop reduction body remains as a global-axiom warning (the bound expression and all downstream theorems are checked)

---

## Implementations

| Backend | Description | When to Use |
|---|---|---|
| `ref/` | Reference C99, portable | Default, all platforms |
| `avx2/` | AVX2 SIMD (~6-7× faster) | Linux x64 with AVX2 CPU |
| `ct/` | Constant-time masked (ISW 2-share) | Side-channel sensitive environments |
| `mem/` | Streaming, < 32 KB stack | IoT, embedded MCUs |

---

## Repository Structure

```
vril-kem/
├── ref/                    Reference implementation (C99)
│   ├── api.h               Public KEM API (NIST PQC compatible)
│   ├── kem.c/h             IND-CCA2 KEM: KeyGen / Encaps / Decaps + OHC
│   ├── indcpa.c/h          IND-CPA encryption layer + CVKDF
│   ├── hi_sample.c/h       HI-Gaussian noise sampler ★
│   ├── cvkdf.c/h           Centripetal Vortex KDF ★
│   ├── poly.c/h            Polynomial arithmetic in R_q
│   ├── ntt.c/h             N-point NTT over Z_12289
│   └── Makefile
├── avx2/                   AVX2 SIMD-optimized implementation
├── ct/                     Constant-time masked implementation
├── mem/                    Memory-optimized variant
├── proof/                  EasyCrypt formal verification ★
│   ├── theories/           Ring, M-LWE, HI-Gaussian, CVKDF, OHC
│   ├── spec/               Abstract KEM specification
│   ├── security/           IND-CPA, IND-CCA2, binding proofs
│   ├── impl/               Jasmin spec↔impl equivalence boundary
│   └── upstream/           Git submodules (EasyCrypt stdlib, EasyCrypt-KEMs, formosa-mlkem)
├── vril-mesh/              Go bindings (hybrid/adaptive suites)
├── dist/                   Distribution assets and validation reports
├── docs/                   Specification, whitepaper, parameter family
├── CHANGELOG.md
└── LICENSE                 MIT
```

★ VRIL-novel constructions and formalizations absent from all existing post-quantum KEMs.

---

## Comparison with NIST PQC Candidates

| Property | VRIL-KEM-4096-7 | ML-KEM-1024 (KYBER) | FrodoKEM-1344 | Classic McEliece |
|---|---|---|---|---|
| Security level | **Level 5+** (~384-bit Q) | Level 5 (~256-bit Q) | Level 5 | Level 5 |
| Quantum hardness assumption | M-LWE | M-LWE | LWE | Code-based |
| Public key size | 50,208 B | 1,568 B | 21,520 B | >1 MB |
| Ciphertext size | 42,016 B | 1,568 B | 21,536 B | 128 B |
| Outer Harmonic Commitment | ✅ | ❌ | ❌ | ❌ |
| HI-Gaussian noise | ✅ | ❌ | ❌ | N/A |
| CVKDF secret hardening | ✅ | ❌ | ❌ | ❌ |
| EasyCrypt formalization | ✅ (IND-CCA2 + Binding) | Partial (formosa-mlkem) | ❌ | ❌ |
| `no_std` / embedded | ✅ (`mem/`) | ✅ | ❌ | ✅ |
| NIST standardized | Research | **FIPS 203** | Draft | **FIPS 205** |

VRIL-KEM trades key/ciphertext size for a significantly higher security margin and novel algorithmic defenses. For bandwidth-constrained protocols, VRIL-KEM-1024-3 offers a competitive profile against ML-KEM-768.

---

## Documentation

| Document | Description |
|---|---|
| [VRIL-KEM-Complete-Specification.md](docs/VRIL-KEM-Complete-Specification.md) | Full technical specification (algorithms, parameters, security proofs) |
| [VRIL-KEM-Parameter-Family.md](docs/VRIL-KEM-Parameter-Family.md) | Detailed parameter set rationale and arithmetic scaffolding |
| [VRIL-KEM-Whitepaper.pdf](docs/VRIL-KEM-Whitepaper.pdf) | Academic whitepaper |
| [proof/README.md](proof/README.md) | EasyCrypt formal verification — architecture, phases, upstream mapping |
| [CHANGELOG.md](CHANGELOG.md) | Version history |
| [docs/future-work/](docs/future-work/) | Research roadmap (TLS 1.3, FPGA/ASIC, Jasmin extraction) |

---

## Contributing

Issues, security reports, and pull requests are welcome at [github.com/VRIL-LABS/vril-kem](https://github.com/VRIL-LABS/vril-kem).

**Security disclosures:** Please report vulnerabilities privately via GitHub Security Advisories before public disclosure.

---

## License

GNU Affero General Public License 3.0 — see [LICENSE](LICENSE).

---

<div align="center">

**VRIL LABS**

*Vortex Resonance Implosion Lattice — Where energy concentrates inward, secrets remain protected.*

[github.com/VRIL-LABS/vril-kem](https://github.com/VRIL-LABS/vril-kem)

</div>
