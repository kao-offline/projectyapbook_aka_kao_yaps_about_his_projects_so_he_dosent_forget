# Abracadabra v0.1 algorithm design

**Status:** Experimental design. This is not production cryptography and has not been independently reviewed.

## Design decision

Abracadabra is not a new cipher. It is a versioned protocol that composes:

1. post-quantum key establishment;
2. authenticated encryption for application payloads;
3. hybrid signatures for identity and anti-spoofing;
4. explicit replay, epoch, and device-state checks.

NIST FIPS 203 defines ML-KEM as a key-encapsulation mechanism for establishing shared secrets, with ML-KEM-512, ML-KEM-768, and ML-KEM-1024 parameter sets.[1] NIST FIPS 204 defines ML-DSA for signatures that authenticate the signer and detect unauthorized modification.[2] The first profile below uses ML-KEM-768 and ML-DSA-65 as candidates; the exact library/API must be pinned to final FIPS-compatible implementations before coding.

## Profile A: `abracadabra/1`

| Function | Candidate construction | Rule |
|---|---|---|
| Key establishment | Hybrid X25519 + ML-KEM-768 | Require both shared-secret contributions; combine only through the defined KDF. |
| Key derivation | HKDF-SHA-384 with domain-separated labels | Never reuse a traffic key for signing, IDs, or recovery. |
| Payload AEAD | ChaCha20-Poly1305 | 32-byte key, 96-bit unique nonce, 16-byte tag. RFC 8439 defines the AEAD construction and requires nonce uniqueness under a key.[3] |
| Identity signature | Hybrid Ed25519 + ML-DSA-65 | v0 accepts a message only when both signatures verify. |
| Password verifier | Argon2id from a maintained library | Passwords never enter the payload encryption path. |
| Encoding | Canonical CBOR or a specified binary encoding | Reject duplicate fields, unknown critical fields, non-canonical integers, and ambiguous encodings. |

The hybrid choice is deliberately conservative: a session is accepted only when the classical and post-quantum components both succeed. If a future review chooses a different combiner, it becomes a new protocol version rather than an in-place change.

## Session handshake

For a first implementation, the authenticated handshake is:

1. The initiator sends a canonical `ClientHello` containing protocol profile, random nonce, ephemeral X25519 public key, intended app/audience, and the responder's ML-KEM-768 key ID.
2. The initiator encapsulates to the responder's ML-KEM-768 public key and sends the KEM ciphertext plus its identity key IDs.
3. The responder generates its ephemeral X25519 key, decapsulates the ML-KEM ciphertext, and returns a canonical `ServerHello` containing its X25519 public key, identity key IDs, and expiry.
4. Both sides compute `transcript_hash = SHA-384(ClientHello || ServerHello)` over the exact canonical bytes.
5. Both sides verify the peer's Ed25519 and ML-DSA-65 signatures over `"abracadabra/v1/handshake" || transcript_hash`.
6. Both sides compute `Z_classical = X25519(local_ephemeral_private, peer_ephemeral_public)` and `Z_pq = ML-KEM.Decaps(local_private, kem_ciphertext)`.
7. Set `session_id = first_16_bytes(SHA-256("abracadabra/v1/session-id" || transcript_hash))`. The session secret is `PRK = HKDF-Extract(transcript_hash, len(Z_classical) || Z_classical || len(Z_pq) || Z_pq)` and `K_master = HKDF-Expand(PRK, "abracadabra/v1/session" || app_id || session_id, 64)`.
8. Split `K_master` into the two direction roots; derive `K_tx` and `K_rx` with the labels shown below.

If either signature, KEM decapsulation, transcript, audience, or algorithm-profile check fails, the session is discarded without exposing which sub-check failed to an unauthenticated peer.

## Key hierarchy

```text
root/recovery key
  └── application key-encryption key (app_id, key_epoch)
        └── object key (object_id, object_epoch)
              └── traffic key (direction, session_id, sequence window)
```

Every derivation includes a literal domain label and all relevant context:

```text
K_app  = HKDF(root_secret,  "abracadabra/v1/app"  || app_id || key_epoch)
K_obj  = HKDF(K_app,       "abracadabra/v1/object" || object_id || object_epoch)
K_tx   = HKDF(K_obj,       "abracadabra/v1/tx" || session_id || direction)
K_rx   = HKDF(K_obj,       "abracadabra/v1/rx" || session_id || direction)
```

`HKDF` above means the selected library's extract-and-expand API, not a custom hash construction. IDs and labels are length-delimited before concatenation.

## Packet algorithm

### Sender

1. Confirm the Abracadabra grant is valid for `app_id`, `object_id`, action, key epoch, and expiry.
2. Load the persisted per-key send sequence. Refuse to send if the sequence would repeat.
3. Construct authenticated metadata:

```text
AAD = encode(
  protocol="abracadabra/1",
  app_id,
  object_id,
  object_epoch,
  session_id,
  direction,
  sequence,
  expiry,
  chunk_index,
  chunk_count
)
```

4. Set `nonce = sender_id_32 || sequence_64`. `sender_id_32` is allocated per key epoch and stored with the send state. The pair must never repeat for the same traffic key.
5. Encrypt `plaintext` with `ChaCha20-Poly1305(K_tx, nonce, AAD, plaintext)`.
6. Define `signed_bytes = "abracadabra/v1/record" || SHA-384(AAD) || SHA-384(ciphertext)` using length-delimited fields, then sign `signed_bytes` with both Ed25519 and ML-DSA-65.
7. Persist the next sequence before releasing the packet to an unreliable transport.

### Receiver

1. Parse canonical encoding and reject oversized or duplicate fields.
2. Check protocol version, app audience, expiry, grant scope, key epoch, sender identity, and sequence window.
3. Verify both signatures over the exact encoded header and ciphertext digest.
4. Reject a sequence already accepted or outside the allowed replay window.
5. Verify the AEAD tag before exposing plaintext to the application.
6. Commit the sequence receipt only after successful authentication and decryption.

## Spoof and replay protection

A valid-looking ciphertext is not enough. The receiver binds acceptance to:

- authenticated app and device identity;
- both signature types;
- handshake transcript hash;
- protocol version and algorithm profile;
- direction and session ID;
- object/key epoch;
- expiry and monotonic sequence.

A challenge-response enrollment must sign a fresh server challenge and device nonce. A copied packet, copied token, or copied ciphertext therefore fails because its challenge, audience, epoch, or sequence is stale or bound to a different transcript.

## Bit-flip meaning

For ordinary storage or network bit flips, ChaCha20-Poly1305 authentication should reject the modified packet; it is detection, not correction.[3] If a transport needs correction of accidental errors, add a separate forward-error-correction layer outside the cryptographic verification boundary:

```text
plaintext → optional compression → AEAD → optional FEC/parity → transport
transport → FEC recovery → AEAD verification → decryption
```

Recovered data is never accepted until the AEAD tag and signatures verify. This does not make a quantum computer's qubits error-correcting; “quantum bitflip” is interpreted here as post-quantum cryptography plus classical bit-flip detection.

## Wire record

```text
magic             4 bytes   "ABR1"
profile           varint    1
algorithm_id      varint    hybrid-x25519-mlkem768 + chacha20poly1305 + hybrid-signature
app_id            bytes
object_id         bytes
object_epoch      uint64
session_id        bytes
sender_id         uint32
sequence          uint64
expiry            uint64
chunk_index       uint32
chunk_count       uint32
ciphertext        bytes
classic_key_id    bytes
pq_key_id         bytes
signature_classic bytes      Ed25519
signature_pq      bytes      ML-DSA-65
```

All integer encodings, maximum lengths, and signature input boundaries must be fixed in the protocol specification. The record is rejected if any field is ambiguous or exceeds the profile limit.

## Required tests before an agent may integrate it

- Known-answer vectors for KEM combination, HKDF labels, AEAD, and both signatures.
- One-bit flips in every header, ciphertext, tag, and signature field.
- Replay, reordering, duplicate, expired, wrong-audience, wrong-direction, and wrong-epoch packets.
- Classical-only and PQ-only signature failures.
- KEM decapsulation failure and downgrade-attempt tests.
- Sequence persistence across crash/restart; prove no nonce reuse.
- Fuzzing for canonical decoding and record length handling.
- Timing and error-message review so failures do not disclose key state.

## Explicit non-goals

- Inventing a “quantum-safe cipher.”
- Claiming protection against quantum bit errors on a physical quantum channel.
- Silent recovery from corrupted ciphertext.
- A server master key that bypasses grants without an explicit recovery policy.
- Production deployment before cryptographic review and dependency validation.

## Sources

[1] [NIST FIPS 203 — ML-KEM](https://csrc.nist.gov/pubs/fips/203/final)

[2] [NIST FIPS 204 — ML-DSA](https://csrc.nist.gov/pubs/fips/204/final)

[3] [RFC 8439 — ChaCha20 and Poly1305 for IETF Protocols](https://www.rfc-editor.org/rfc/rfc8439)
