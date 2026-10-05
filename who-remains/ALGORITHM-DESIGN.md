# Who remains v0.1 algorithm design

**Status:** Experimental design. This is not production cryptography, storage, or transport software.

## Design decision: Lake Maze

**Lake Maze** is the fast encrypted-data layout inside Who remains. “Skip across the lake” means a reader can locate and authenticate only the bins needed for a range, field, version, or playback window instead of downloading and decrypting the entire object.

Lake Maze is therefore a data layout and transfer algorithm, not a new cipher.

## Profile W: `who-remains/1`

| Layer | v0.1 choice | Reason |
|---|---|---|
| Logical data | Immutable object versions | Makes resume, caching, and rollback explicit. |
| Binning | Fixed 1 MiB bins, configurable by profile | Simple first benchmark; avoids prematurely committing to content-defined chunking. |
| Compression | Optional independent per-bin Zstandard | A skipped bin remains independently decompressible; compress before encryption. |
| Encryption | ChaCha20-Poly1305 per bin | Authenticated encryption detects tampering/bit flips; nonce uniqueness is mandatory.[3] |
| Bin ID | HMAC-SHA-256 with an object-scoped ID key | Prevents a global plaintext/equality index; never use a raw plaintext hash as a public ID. |
| Index | Two-level sparse manifest plus Merkle root | Fetch the top index, jump to relevant bins, then verify the selected path. |
| Manifest auth | Abracadabra grant plus hybrid signature | Identity and key access remain outside this data-plane format. |
| Recovery | Per-bin receipts and bounded retry | A lost bin does not force a complete object restart. |

The 1 MiB value is a starting benchmark parameter, not a universal optimum. The benchmark must compare smaller and larger bins under real payload and network profiles before changing the default.

## Object layout

```text
LakeMazeObject
├── header
├── version_manifest
│   ├── object_id
│   ├── version_id / parent_version_id
│   ├── key_epoch
│   ├── codec/profile IDs
│   ├── bin_count / logical_size / encoded_size
│   ├── sparse index: logical range → bin range
│   ├── Merkle root over authenticated bin IDs
│   └── Abracadabra signature/grant reference
└── bins[]
    ├── bin_index
    ├── logical_offset / logical_length
    ├── encoded_length
    ├── nonce
    ├── ciphertext + AEAD tag
    └── object-scoped bin ID
```

The manifest is authenticated before any bin is exposed. A partial fetch may prove that a bin is authentic, but it must not be advertised as a committed object version until the required manifest and application policy checks pass.

## Merkle rules

The tree is deterministic and domain-separated:

```text
leaf(bin_id)       = SHA-256(0x00 || len(bin_id) || bin_id)
parent(left,right)  = SHA-256(0x01 || left || right)
```

For an odd number of nodes, duplicate the final node at that level. The manifest stores the tree height and root. A reader accepts a bin only when its computed root equals the signed manifest root.

## Exact v0.1 pipeline

### Write

1. Canonicalize the application payload and metadata.
2. Create a new immutable `version_id` and record its optional parent.
3. Split the logical byte stream into fixed 1 MiB bins, with the final bin shorter.
4. Compress each bin independently when the profile allows it.
5. Derive an object key from an Abracadabra grant and current `key_epoch`.
6. Derive `bin_key = HKDF(object_key, "who-remains/v1/bin" || version_id || bin_index)`.
7. Derive a unique 96-bit nonce from a persisted object nonce prefix and `bin_index`; never reuse a nonce with the same key.
8. Build AAD from protocol/profile, object ID, version ID, parent ID, key epoch, bin index, logical length, and codec ID.
9. Encrypt the encoded bin with ChaCha20-Poly1305.
10. Compute `bin_id = HMAC-SHA-256(object_id_key, version_id || bin_index || ciphertext)`.
11. Build the sparse index and Merkle tree over `bin_id` values.
12. Sign the canonical manifest through Abracadabra and publish the manifest before or atomically with the bins.

### Read / “skip across the lake”

1. Fetch and authenticate the small top-level manifest.
2. Translate the requested logical range or field into required bin indexes using the sparse index.
3. Fetch only those bins, in parallel up to a bounded concurrency limit.
4. Verify each bin ID and Merkle path.
5. Verify the AEAD tag and associated data.
6. Decompress only the requested bins.
7. Return the requested range and a receipt; never mark missing bins as zeroes or silently substitute stale versions.

## Version delta algorithm

For a new version:

```text
new_version = parent_version + changed_bins + new_manifest
```

Unchanged-bin reuse is allowed only within the same object and security policy. Cross-user or global equality deduplication is disabled in v0.1 because it can disclose relationships between encrypted data. A changed bin gets a new `version_id`/`bin_index` derivation and a new AEAD nonce.

## Transfer protocol

Every bin request is idempotent:

```text
GET /object/{object_id}/version/{version_id}/bin/{bin_index}
  Authorization: Abracadabra grant
  If-Match: manifest_digest
  Range: optional encoded byte range

200  authenticated bin + receipt token
206  partial transport response, still requires full bin verification
404  authenticated manifest says bin is unavailable
409  manifest/version conflict; refetch manifest
410  tombstoned version
```

A receipt contains object ID, version ID, bin index, bin ID, manifest digest, transfer attempt ID, and authenticated acceptance time. The receiver commits the receipt only after verification. Retries use the same logical bin ID and do not create duplicate versions.

## Loss and bit-flip handling

AEAD is the cryptographic authority: a modified bin is rejected. For links with measurable accidental loss or corruption, add optional parity outside the ciphertext:

```text
compress bin → encrypt bin → group encrypted bins → parity/FEC → transport
transport → recover missing bytes/bins → verify bin ID + AEAD → decrypt
```

FEC may improve recovery; it does not authenticate data and must never bypass the AEAD or manifest checks. A quantum bit-flip channel would require a separate quantum error-correction protocol; Lake Maze is a classical encrypted-data system.

## Performance measurements

Agents must report measurements, not “fast” claims:

- manifest bytes / logical bytes;
- encrypted bytes / compressed bytes;
- time to first authenticated requested bin;
- range-read bytes versus full-object bytes;
- bins recovered after simulated loss rates;
- retry bytes and duplicate bytes;
- CPU, memory, and concurrency;
- version update bytes for unchanged and changed data;
- time to rebuild the sparse index and Merkle path.

Compare fixed bins against content-defined chunking only after the fixed-bin implementation is correct. Record the leakage, CPU, metadata, and cache trade-offs of each alternative.

## Required tests before integration

- Round-trip binary payloads from zero bytes through multi-gigabyte streamed fixtures.
- Range reads that skip bins before, between, and after the requested region.
- Manifest, Merkle path, bin ID, AAD, nonce, ciphertext, tag, and signature tampering.
- Interrupted upload/download at every bin boundary and resume without duplication.
- Lost, reordered, duplicated, stale, and wrong-version receipts.
- Parent-version reuse and changed-bin isolation.
- Compression on/off and incompressible input behavior.
- Simulated bit flips and packet loss with and without FEC.
- Fuzzing of manifests, indexes, range requests, and transfer state machines.

## Open algorithm decisions

- Confirm whether “binning” means fixed bins, content-defined chunks, bucketed indexes, or a combination.
- Confirm target payload classes: files, database records, media streams, or all three.
- Benchmark 256 KiB, 1 MiB, and 4 MiB bins before setting the default.
- Decide whether the sparse index is public, encrypted, or selectively disclosed.
- Decide which compression profiles are allowed and how codec changes affect version reuse.
- Define maximum version retention, tombstone lifetime, and garbage collection.
- Define the minimum number of bins required before a version becomes committed.

## Sources

[3] [RFC 8439 — ChaCha20 and Poly1305 for IETF Protocols](https://www.rfc-editor.org/rfc/rfc8439)
