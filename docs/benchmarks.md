# Benchmarks

Single-threaded CPU performance and SNOMED CT matching accuracy on Australian GP clinical notes.

## At a glance

| | SNOExtract |
|---|---|
| Throughput        | 30–60 notes/sec sustained (server); <100 ms per call (one-shot CLI) |
| Peak RAM          | 118 MB |
| Disk footprint    | <300 MB |
| Hardware          | Single-threaded, x86_64 CPU, no GPU |
| Disorder F1       | **74.6** |
| Substance / Medication F1 | **84.7** |

## Environment

ASUS Zenbook 14 · Intel Core i7-13700H · 16 GB RAM · Ubuntu 24.04 LTS · single-threaded, CPU-only · profiled with `/usr/bin/time -v` · benchmark date 2026-03-15.

Corpus: 25 Australian GP notes from SynGP500 (held out from training), 2,448 ground-truth annotations — Azure pre-annotated, human doctor reviewed, SNOMED CT-AU coded.

## Throughput and resource usage

| System | Wall time (25 notes) | Peak RAM | Disk | Notes/sec |
|---|---:|---:|---:|---:|
| **SNOExtract**       | **0.4 s** | **118 MB** | **<300 MB** | **62.5** |
| MedCAT (Full SNOMED) | 17.2 s    | 3,783 MB   | 1,625 MB     | 1.5 |
| QuickUMLS            | 12.1 s    | 2,659 MB   | 35,000 MB    | 2.1 |
| GPT-5.2 (cloud API)  | 1,493 s   | —          | —            | 0.017 |

## Matching accuracy by semantic type

| Semantic Type | Ground truth | F1 (×100) |
|---|---:|---:|
| **Substance** — medications, active ingredients | 282 | **84.7** |
| **Regime / therapy**                            |  21 |   82.1   |
| **Disorder** — diagnoses, conditions            | 459 | **74.6** |
| Finding — symptoms, clinical observations       | 790 |   69.6   |
| Procedure                                       | 299 |   64.0   |

**Substance** and **Disorder** are strong categories — the two most clinically actionable types for medication reconciliation, problem-list maintenance, and diagnosis-driven analytics.

---