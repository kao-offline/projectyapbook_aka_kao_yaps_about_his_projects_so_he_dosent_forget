# Who remains — agent-ready overview

**Status:** Concept only. No repository or implementation exists yet.

**Detailed algorithm design:** [ALGORITHM-DESIGN.md](./ALGORITHM-DESIGN.md)

## One-sentence purpose

Who remains is the encrypted data-plane format and transfer system for Kao's apps: it versions, packs, encrypts, verifies, resumes, and manages data while reducing transfer latency, loss impact, and unnecessary bytes.

## Boundary

Who remains owns:

- canonical object and version manifests;
- chunking/binning and packing strategy;
- compression-before-encryption policy;
- authenticated encrypted payload layout;
- integrity, receipts, retries, resumability, and partial recovery;
- retention, version references, and garbage-collection metadata;
- transport-efficient APIs for apps such as stamp.dev.

Who remains does **not** own user login, device identity, permission decisions, or password handling. Abracadabra supplies identity, grants, and the key-encryption boundary.

## Relationship with Abracadabra

```text
Abracadabra: who may access this object and which key epoch is allowed
        │ scoped grant + wrapped object key
        ▼
Who remains: how the encrypted object is represented, versioned, moved, resumed
        │ chunks / bins / manifest / receipts
        ▼
Storage, peers, WebDAV, Git/object backends, or app-specific transport
```

The first implementation should keep these interfaces separate so either system can be tested or replaced independently.

## Candidate object pipeline

The exact format is an open design decision. A safe candidate pipeline is:

1. Canonicalize the logical payload and metadata.
2. Optionally compress before encryption; never compress ciphertext.
3. Split into fixed-size chunks for the first implementation.
4. Encrypt each chunk with authenticated encryption and fresh nonces.
5. Bind each chunk to object ID, version, chunk index, algorithm, and protocol version as associated data.
6. Build a manifest containing chunk references, lengths, ordering, version ancestry, and integrity commitments.
7. Sign or authenticate the manifest using keys/grants from Abracadabra.
8. Transfer missing chunks independently with resumable receipts and bounded retries.
9. Commit the version only after the manifest and required chunks pass verification.

The term **binning** needs a precise definition before implementation: fixed-size bins, content-defined chunks, bucketed manifests, or another grouping strategy. Do not silently choose a design that leaks plaintext similarity or makes encryption unsafe.

## Data model

- **Object:** stable logical identity for a file, record, or application dataset.
- **Version:** immutable snapshot with parent/version ancestry and creation metadata.
- **Chunk:** independently authenticated encrypted payload with index, length, nonce, and key epoch reference.
- **Bin:** transport/storage grouping of chunks; semantics must be specified before use.
- **Manifest:** authenticated description of a version and its chunks; contains no plaintext secrets.
- **Receipt:** signed or authenticated acknowledgement of accepted chunk content and version.
- **Tombstone:** authenticated deletion or supersession marker that prevents ambiguous resurrection.
- **Retention policy:** rules for keeping latest, pinned, recoverable, or expired versions.

## First vertical slice

Start local and measurable:

1. encode a small object into a versioned manifest;
2. use fixed-size chunks and authenticated encryption;
3. support upload, interruption, resume, and duplicate-chunk idempotency;
4. verify chunk and manifest integrity before commit;
5. reconstruct the exact original bytes;
6. create a second version that reuses unchanged encrypted chunks only when the security policy permits it;
7. delete or expire a version through an authenticated tombstone;
8. measure bytes sent, time-to-first-byte, resume cost, and loss recovery.

## Efficiency strategy

- Begin with fixed-size chunks because they are easier to test and benchmark.
- Add content-defined chunking only if measured version deltas justify its complexity.
- Compress before encryption and record the codec/version in authenticated metadata.
- Keep manifests small, canonical, and independently fetchable.
- Use parallel chunk transfer with bounded concurrency rather than unbounded fan-out.
- Make every upload and receipt idempotent so retries do not duplicate logical data.
- Support range/resume semantics and verify every accepted chunk.
- Separate metadata needed for routing from metadata that should remain private.
- Avoid cross-user plaintext deduplication; encrypted equality can leak relationships unless deliberately scoped and keyed.

## Threat and failure model

Plan for tampering, truncated chunks, replayed receipts, missing chunks, duplicated chunks, stale manifests, partial network loss, malicious storage, version rollback, compromised client devices, and metadata leakage. Integrity failures must be explicit and fail closed; a partial object must never appear committed as a complete version.

## Agent workstreams

- **Format/spec agent:** canonical manifest, chunk header, versioning, error codes, compatibility rules.
- **Crypto/encoding agent:** integrate Abracadabra grants and audited encryption library without duplicating crypto logic.
- **Chunking agent:** fixed-size implementation, then benchmark-driven binning/content-defined experiments.
- **Transfer agent:** resumable upload/download, receipts, retry budgets, bounded concurrency, cancellation.
- **Storage adapter agent:** local filesystem first; later adapters for object storage, WebDAV, and app-specific backends.
- **Versioning agent:** ancestry, retention, tombstones, garbage collection, conflict behavior.
- **Benchmark agent:** latency, bytes transferred, interruption recovery, loss rates, CPU/memory, and manifest overhead.
- **Security-review agent:** leakage analysis, replay/tamper tests, metadata review, dependency audit.

## Verification gates

- Round-trip tests across random payload sizes and binary data.
- Tamper, reorder, truncate, duplicate, wrong-key, stale-manifest, and replay tests.
- Interrupted-transfer and resume tests at every chunk boundary.
- Property tests for canonical encoding and idempotent receipts.
- Benchmarks comparing fixed-size, content-defined, and alternative binning strategies.
- Explicit accounting of plaintext, compressed, encrypted, manifest, retry, and redundant bytes.
- Compatibility fixtures so older versions fail clearly rather than being misread.
- No claim of loss minimization without a reproducible network-loss benchmark.

## Open decisions

- What exactly does “binning” mean for this project?
- What are target object sizes, network conditions, and acceptable recovery times?
- Is compression mandatory, optional, or selected per content type?
- Is encrypted chunk reuse allowed across versions, users, or only one object?
- What is the maximum retained version count and garbage-collection policy?
- Are versions immutable snapshots, append-only logs, or both?
- How are concurrent edits merged or rejected?
- Which storage backends are required for the first real app integration?

## Non-goals for v1

- A new encryption primitive.
- Global deduplication that exposes plaintext relationships.
- Unbounded peer-to-peer replication.
- Silent conflict resolution for concurrent writes.
- Production performance claims before benchmark fixtures exist.
