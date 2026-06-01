# ADR 0001: Reproducibility strategy for NorthStar models

## Context
Currently, the ML team at NorthStar Logistics has no centralized model registry or data versioning system, resulting in models being manually uploaded to S3 with fragile, non-deterministic naming conventions like `eta_v2_FINAL.onnx`. There is no programmatic trace linking these compiled production binaries back to the exact code commits, database states, or random seed parameters that generated them, making debugging and reliable model rolling back impossible.

## Decision
To guarantee absolute functional reproducibility across all current and future deployments, we will enforce strict pinning across four operational layers using the following concrete tooling and conventions:

* **Layer 1: Environment:** We will pin all dependency trees. Every model project must use `Poetry` to generate an explicit lockfile (`poetry.lock`). For training execution, this environment will be bundled into a Docker container image. The base runtime must use an exact image digest string (e.g., `python:3.11-slim@sha256:...`) rather than semantic tags.
* **Layer 2: Data:** We will adopt `DVC` (Data Version Control) backed by our existing S3 storage. Raw query results for training sets will be outputted to immutable CSV/Parquet artifacts, tracked via content-addressed `.dvc` pointer files containing strict MD5 hashes, and checked into Git alongside the source code.
* **Layer 3: Code:** Every model training execution must be initiated via an automated GitHub Actions runner or a local wrapper script that enforces a clean Git state. The exact 40-character `Git Commit SHA` will be automatically injected into the model's metadata and logged to an `MLflow` experiment tracking server upon run completion.
* **Layer 4: Randomness:** We will enforce a programmatic mandate across all training entrypoints. Scripts must explicitly configure seeds for all underlying pseudo-random number generators (`random.seed(42)`, `numpy.random.seed(42)`, and framework-specific options like `torch.manual_seed(42)`). If distributed execution or data shuffling is leveraged, the data partition worker ranks must be deterministically assigned using fixed index logic.

## Alternatives rejected
* **Relying on S3 Timestamps & Manual Folder Versioning:** We rejected the practice of organizing data by `/year/month/day/` prefixes in S3. Upstream backfills, historical data rectifications, or accidental overwrites can quietly mutate past data snapshots, invalidating historical training continuity.
* **Using Unlocked `requirements.txt` Files:** We rejected using loose package requirement specs. Simple semantic version shifting in transient dependencies can break runtime compilation or introduce subtle differences in mathematical calculation layers between training and production instances.

## Consequences
* **Operational Friction:** Engineers can no longer run "quick fixes" locally and push them directly to S3. All changes must be cleanly committed to Git, and training data must pass through the `DVC` registration pipeline.
* **Storage Overhead:** Storing immutable, content-addressed dataset snapshots via DVC in S3 will scale our storage requirements linearly as retraining intervals occur (every 2 weeks for ETA, monthly for Routing).
* **Auditable Security:** The business gains a comprehensive audit trail. We can pinpoint the exact system state for any model version if performance anomalies manifest or if leadership requests a performance audit.

## Revisit if
We will revisit this architectural strategy if our data volume scales into multi-terabyte limits where file-based snapshotting with DVC introduces major local disk bottlenecks, forcing a transition toward enterprise data lakehouse versioning systems like `Delta Lake` or `LakeFS`.