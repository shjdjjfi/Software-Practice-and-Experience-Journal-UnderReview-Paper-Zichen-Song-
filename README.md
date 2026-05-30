# Proof-Carrying Source-Level Verification with Small-Kernel Certificate Checking

This repository provides an artifact for **proof-carrying source-level verification**:
a framework that separates *proof production* from *proof trust* for Rust
program verification.

The key idea is simple:

> A large verifier may discover a proof, but a small independent checker should
> decide whether the proof evidence is trustworthy.

Instead of treating the original verification engine as part of the trusted
computing base, this artifact emits explicit proof certificates and checks them
with a lightweight replay kernel. The result is an auditable verification
workflow in which proof evidence can be stored, inspected, compressed, corrupted
for testing, and independently rechecked.

---

## 1. Overview

Modern deductive verifiers often combine several responsibilities inside one
large engine: parsing source programs, generating verification conditions,
searching for proofs, applying proof rules, simplifying arithmetic, managing
branching proof states, and checking final proof closure. This monolithic design
is convenient for automation, but it also means that the final verification
result depends on a large trusted implementation.

This artifact explores a different design.

The verifier is treated as an **untrusted proof producer**. Its output is
converted into a structured proof certificate. The final accept/reject decision
is made by a small checker that replays the certificate without invoking the
original proof engine.

The framework supports:

- explicit JSON proof certificates;
- full and compressed certificate formats;
- independent small-kernel certificate checking;
- rule-schema and trace-shape replay;
- side-condition and substitution validation;
- branch-closure validation;
- replay digests for certificate integrity;
- negative tests for corrupted certificates;
- benchmark generation in CSV and Markdown formats.

The artifact targets source-level Rust verification, but the design is phrased
around a general proof-carrying verification workflow: proof producers may be
large, heuristic, and implementation-heavy, while proof checkers should be small,
auditable, and conservative.

---

## 2. Main Contributions

This artifact implements the following contributions.

### 2.1 Proof-Carrying Verification Workflow

The artifact introduces a proof-carrying workflow for source-level verification:

```text
Source program + specification
        ↓
verification engine as proof producer
        ↓
structured proof certificate
        ↓
certificate compression
        ↓
small independent checker
        ↓
accepted / rejected + replay statistics
```

The important design choice is that the verification engine is not trusted for
the final result. It is only trusted to produce candidate proof evidence.

### 2.2 Small-Kernel Certificate Checking

The trusted component is a small checker that validates certificate structure,
known rule schemas, replay state, branch metadata, closure evidence, and replay
digests. It does not call the original proof engine during checking.

This reduces the trusted boundary from a large verification stack to a compact
certificate checker and its explicit rule-schema interface.

### 2.3 Replayable Proof Certificates

The full certificate format records:

- certificate metadata;
- initial and final proof states;
- proof-step identifiers;
- rule names;
- parent-child proof structure;
- branch identifiers;
- side-condition attributes;
- substitutions;
- closure evidence;
- textual or opaque sequent snapshots;
- replay digests.

The certificate is designed to be inspectable and replayable, not merely a log
file.

### 2.4 Compression-Preserving Replay

The artifact supports compressed certificates. Compression is not treated as a
pure storage optimization: compressed macro steps are checked against the replay
discipline, and corrupted macro certificates are rejected.

This allows large proof traces to be reduced while preserving independent
checkability.

### 2.5 Negative Certificate Validation

The artifact includes negative tests that deliberately corrupt certificates.
These tests modify rule identifiers, substitutions, side conditions, initial
sequents, branch closure evidence, and compressed macro steps.

The expected behavior is rejection. These tests are included to demonstrate that
the checker is not an accept-all validator.

---

## 3. Architecture

The implementation is organized around four stages.

### Stage 1: Proof Production

A source-level verification engine is used to produce a proof trace. In this
artifact, existing readable proof traces are used as the initial proof evidence.

The proof producer may be large and complex. It may include symbolic execution,
proof search, rule application, simplification, and source-language front-end
processing. None of these components are part of the small trusted checker.

### Stage 2: Certificate Extraction

The extractor converts proof traces into full JSON certificates. Each proof step
is represented explicitly and linked to its parent, branch, rule family, and
available side-condition information.

When textual sequents are available in proof comments, they are recorded. When
the proof trace omits textual sequents, stable opaque/current sequent tokens are
used and the certificate is classified under conservative trace-shape replay.

### Stage 3: Certificate Compression

Full certificates can be compressed into macro certificates. Compression groups
replayable proof fragments while preserving enough structure for independent
checking.

The compressed representation records macro steps, replay digests, and the
metadata required to validate that compression did not invent proof evidence.

### Stage 4: Independent Checking

The checker reads either a full or compressed certificate and returns:

```text
accepted / rejected + statistics
```

Unsupported rules are rejected explicitly. The checker does not silently accept
unknown proof steps.

---

## 4. Trusted Computing Base

The trusted computing base consists of:

- the small certificate checker;
- the JSON certificate reader;
- the textual rule-schema whitelist;
- conservative side-condition checks;
- conservative substitution checks;
- digest validation;
- certificate-structure validation.

The trusted computing base does **not** include:

- the original proof-search engine;
- proof automation heuristics;
- large taclet/rule execution machinery;
- arithmetic simplification engines from the proof producer;
- source-language compiler internals used by the proof producer;
- proof-generation scripts;
- certificate compression as a trusted step.

The checker is deliberately conservative. If the certificate contains an
unsupported rule, malformed branch structure, invalid closure evidence, corrupted
substitution, inconsistent side condition, or broken macro step, the checker
rejects it.

---

## 5. Replay Modes

The artifact supports several replay modes.

### 5.1 Precise Replay

Precise replay validates:

- rule identifiers;
- step identifiers;
- parent-child dependencies;
- branch metadata;
- side-condition attributes;
- substitutions present in the proof trace;
- branch closure flags;
- current-goal chaining;
- replay digests.

This is the strongest replay mode used when sufficient proof-trace information is
available.

### 5.2 Schema-Level Textual Replay

When textual sequents are available, the checker performs schema-level replay for
supported proof-rule families, including:

- assignment and update rules;
- borrowing and reference rules;
- mutable-write rules;
- array rules;
- tuple rules;
- enum rules;
- loop-rule families;
- deterministic symbolic simplification;
- branch-closing rules.

The checker validates the rule family and the certificate-local evidence rather
than trusting the original proof engine.

### 5.3 Conservative Trace-Shape Replay

Some proof traces omit full textual sequents. In such cases, the checker uses
stable opaque/current sequent tokens and validates:

- known rule families;
- proof-tree shape;
- branch structure;
- step identifiers;
- closure evidence;
- replay digests.

This mode is conservative and explicitly reported as trace-shape validation. It
does not pretend to reconstruct information that the proof trace did not expose.

### 5.4 Conservative Arithmetic Replay

Arithmetic simplification rules are handled conservatively. The checker supports
rule families such as:

- `polySimp_*`;
- `polyDiv_*`;
- `inEqSimp_*`;
- literal simplification rules.

The artifact does not implement a full SMT solver. Richer SMT-backed arithmetic
certificate checking is future work.

---

## 6. Supported Proof Features

The current artifact supports the proof-rule families needed by the included
source-level verification examples. These include:

- assignment/update rules;
- variable substitution;
- shared borrowing;
- mutable borrowing;
- reference dereference;
- reference write;
- array-rule names when emitted by the proof producer;
- tuple-rule names when emitted by the proof producer;
- enum-rule names when emitted by the proof producer;
- loop-invariant rule names;
- deterministic symbolic simplification;
- arithmetic normalization using supported simplification prefixes;
- branch closure by explicit assumption reference;
- branch closure by syntactic closure marker.

Unsupported rules are rejected with an explicit `UnsupportedRule` result.

---

## 7. Repository Layout

A typical checkout contains the following components.

```text
.
├── checker/                       # small independent certificate checker
├── docs/
│   ├── certificate_format.md       # full and compressed JSON certificate format
│   ├── theory.md                   # replay and compression soundness argument
│   └── artifact_usage.md           # detailed artifact commands
├── examples/                       # source-level verification examples
├── proofs/                         # generated or existing proof traces
├── results/
│   ├── cert_benchmark.csv          # benchmark results
│   ├── cert_benchmark.md           # benchmark table
│   ├── cert_negative_tests.csv     # negative-test results
│   └── cert_negative_tests.md      # negative-test table
├── tests/                          # positive, negative, and compression tests
├── rustydl-cert                    # command-line entry point
├── Makefile                        # convenience targets
└── IMPLEMENTATION_SUMMARY.md       # implementation and limitation summary
```

Some directories may be generated after running the artifact commands.

---

## 8. Quick Start

### 8.1 Requirements

The artifact expects a Unix-like shell environment and Python 3.

If proof production is run from source-level verification inputs, the underlying
verification backend may additionally require Java, Rust, Cargo, and related
toolchain components. For checking existing certificates, the small checker only
needs the certificate files and Python environment.

### 8.2 Run Unit Tests

```bash
python3 -m unittest discover -s tests -p '*tests.py'
```

This runs positive replay tests, negative certificate tests, and compression
tests.

### 8.3 Generate a Full Certificate from an Existing Proof

```bash
./rustydl-cert from-proof examples/paper/example5.proof --out out/example5.full.json
```

### 8.4 Check a Full Certificate

```bash
./rustydl-cert check out/example5.full.json
```

Expected behavior:

```text
accepted + replay statistics
```

or, for corrupted/unsupported inputs:

```text
rejected + diagnostic reason
```

### 8.5 Run Verification and Emit a Certificate

```bash
./rustydl-cert verify examples/binary-search/binary-search.key --emit-cert out/binary.full.json
```

This command invokes proof production and emits a certificate for independent
checking.

### 8.6 Compress a Certificate

```bash
./rustydl-cert compress out/example5.full.json --out out/example5.compressed.json
```

### 8.7 Check a Compressed Certificate

```bash
./rustydl-cert check-compressed out/example5.compressed.json
```

---

## 9. Make Targets

The artifact provides convenience targets.

### 9.1 Verify One Example

```bash
make verify-cert EXAMPLE=example5
```

```bash
make verify-cert EXAMPLE=binary_search
```

### 9.2 Run Certificate Tests

```bash
make cert-tests
```

### 9.3 Run Benchmarks

```bash
make cert-benchmarks
```

This generates positive benchmark results and negative-test results.

---

## 10. Benchmark Outputs

Benchmark results are emitted in both CSV and Markdown formats.

```text
results/cert_benchmark.csv
results/cert_benchmark.md
```

The benchmark table reports certificate-checking statistics such as accepted
proofs, replay mode, trusted checker size, rule coverage, compressed certificate
behavior, and replay outcome.

The artifact also generates negative-test outputs:

```text
results/cert_negative_tests.csv
results/cert_negative_tests.md
```

Negative tests are expected to be rejected by the checker.

---

## 11. Negative Tests

Negative tests intentionally corrupt certificate components, including:

- rule identifiers;
- substitutions;
- side conditions;
- initial sequents;
- branch closure evidence;
- compressed macro steps.

The purpose is to validate that the checker enforces certificate integrity rather
than accepting arbitrary traces.

A successful negative-test run means that corrupted certificates are rejected
with explicit diagnostic categories.

---

## 12. Certificate Format

The full certificate format is JSON-based. It records proof-level metadata and a
sequence of proof steps.

A proof step may contain:

```json
{
  "id": "step-42",
  "rule": "assignment_update",
  "parent": "step-41",
  "branch": "main",
  "side_conditions": {},
  "substitutions": {},
  "before": "...",
  "after": "...",
  "digest": "..."
}
```

The compressed certificate format replaces selected proof fragments with macro
steps while preserving replay metadata and digest evidence.

The complete specification is provided in:

```text
docs/certificate_format.md
```

---

## 13. Soundness Argument

The artifact includes an FM-style soundness argument in:

```text
docs/theory.md
```

At a high level, the intended guarantee is:

> If the independent checker accepts a certificate, then the certificate
> corresponds to a replayable derivation under the supported rule schemas and
> replay discipline.

For compressed certificates, the intended guarantee is:

> If the checker accepts a compressed certificate, then the macro replay is valid
> with respect to the corresponding full replay structure.

The current implementation is deliberately conservative. Unsupported or
underspecified proof evidence is rejected rather than accepted silently.

---

## 14. Relationship to Existing Verification Engines

This artifact is not a replacement for a deductive verifier. It does not attempt
to improve proof search, automate more programs, or outperform existing
verification tools.

Instead, it adds a proof-carrying layer around source-level verification:

| Aspect | Conventional verifier-centered workflow | This artifact |
|---|---|---|
| Proof search | Performed by large verifier | Performed by large verifier |
| Final trust decision | Large verifier is trusted | Small checker is trusted |
| Proof evidence | Internal trace or proof file | Explicit JSON certificate |
| Rechecking | Requires verifier-specific machinery | Uses independent checker |
| Compression | Not central | Checked macro certificates |
| Negative testing | Optional | Built into artifact |
| Failure mode | Tool-specific failure | Explicit reject reason |

The verification backend is therefore used as a proof producer, not as the final
trusted authority.

---

## 15. Limitations

The artifact is intentionally conservative and has several limitations.

1. **Replay subset.**  
   The checker supports the rule families needed by the included examples. It is
   not a full implementation of every rule from the proof producer.

2. **Arithmetic checking.**  
   Arithmetic replay is conservative and rule-family based. The checker does not
   yet include a full SMT-backed arithmetic certificate checker.

3. **Opaque sequents.**  
   Some proof traces omit textual sequents. In these cases, the checker uses
   conservative trace-shape validation and reports this replay mode explicitly.

4. **Source-language coverage.**  
   The current examples cover a representative subset of source-level Rust
   verification patterns, but not the full Rust language.

5. **Certificate extraction.**  
   The current extractor is intentionally non-invasive. It parses readable proof
   traces rather than modifying the internals of the proof producer.

6. **Checker implementation.**  
   The checker is small by design. Its purpose is independent replay validation,
   not proof search or full reimplementation of a verification engine.

These limitations are design choices for an artifact focused on reducing trusted
proof-checking infrastructure.

---

## 16. Reproducibility Checklist

To reproduce the artifact results:

1. Install the required runtime environment.
2. Run the unit tests:

   ```bash
   python3 -m unittest discover -s tests -p '*tests.py'
   ```

3. Generate a full certificate:

   ```bash
   ./rustydl-cert from-proof examples/paper/example5.proof --out out/example5.full.json
   ```

4. Check the full certificate:

   ```bash
   ./rustydl-cert check out/example5.full.json
   ```

5. Compress the certificate:

   ```bash
   ./rustydl-cert compress out/example5.full.json --out out/example5.compressed.json
   ```

6. Check the compressed certificate:

   ```bash
   ./rustydl-cert check-compressed out/example5.compressed.json
   ```

7. Run benchmark generation:

   ```bash
   make cert-benchmarks
   ```

8. Inspect:

   ```text
   results/cert_benchmark.md
   results/cert_negative_tests.md
   ```

---

## 17. Expected Artifact Claims

A successful artifact run should demonstrate the following claims.

### Claim 1: Existing proof evidence can be converted into explicit certificates.

The artifact generates full JSON certificates from proof traces and records the
rule-level replay structure.

### Claim 2: Certificates can be checked independently.

The checker validates full certificates without invoking the original proof
engine.

### Claim 3: Certificate compression preserves replay checkability.

Compressed certificates are accepted only when macro replay is valid.

### Claim 4: Corrupted certificates are rejected.

Negative tests show that the checker rejects malformed rules, substitutions,
side conditions, branch evidence, initial sequents, and macro steps.

### Claim 5: The trusted boundary is reduced.

The final trust decision depends on a small checker rather than on the full proof
production engine.

---

## 18. Citation

If this artifact is used in academic work, cite the accompanying paper:

```bibtex
@article{proofcarrying_source_level_verification,
  title   = {Proof-Carrying Source-Level Verification with Small-Kernel Certificate Checking},
  author  = {Zichen Song, Weijia Li},
  journal = {Software: Practice and Experience},
  year    = {2026}
}
```

Please replace the placeholder metadata with the final bibliographic information
once available.

---

## 19. License and Acknowledgments

This artifact builds on existing source-level verification infrastructure and
proof traces. The certificate layer, checker, compression logic, tests, and
documentation implement a proof-carrying verification workflow on top of that
infrastructure.

Please consult the repository license files and third-party dependency notices
before redistribution.
