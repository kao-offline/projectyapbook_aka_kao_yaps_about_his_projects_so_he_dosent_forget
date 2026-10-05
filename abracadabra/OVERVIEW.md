# Abracadabra — agent-ready overview

**Status:** Concept only. No repository or implementation exists yet.

## One-sentence purpose

Abracadabra is the shared authentication and application-encryption layer for Kao's apps: it establishes who or what is allowed to act, which keys may be used, and how encrypted data is granted, rotated, revoked, and audited.

## Boundary

Abracadabra owns:

- user, service, device, and application identity;
- login/session and token semantics;
- device enrollment and key lifecycle;
- authorization and encrypted-key grants;
- envelope encryption orchestration;
- key rotation, revocation, recovery, and audit events;
- SDKs and protocol messages used by apps such as stamp.dev.

Abracadabra does **not** invent a new cipher, replace TLS, define storage chunking, or decide how files are packed for transport. Those concerns belong to established libraries, transport layers, and **Who remains**.

## Relationship with Who remains

```text
Application (stamp.dev, Alplight, etc.)
        │ identity, permissions, key access
        ▼
   Abracadabra
        │ object key / grant / policy
        ▼
   Who remains
        │ encrypted object format, versions, chunks, bins, transfer
        ▼
     Storage and transport
```

Abracadabra should be able to authorize access without knowing the full plaintext payload. Who remains should be able to move and version encrypted objects without implementing user identity.

## Candidate security baseline

This is a starting point for review, not a cryptographic specification:

- Use a mature, audited library such as libsodium or an equivalent platform-reviewed implementation.
- Use authenticated encryption only; candidate payload primitive: XChaCha20-Poly1305 or AES-256-GCM, selected once per supported platform and protocol version.
- Use separate signing and key-agreement keys; candidate algorithms are Ed25519 and X25519.
- Derive subkeys with HKDF or the KDF supplied by the selected audited library.
- Hash passwords with Argon2id through a maintained implementation; never encrypt or hash passwords with the payload scheme.
- Bind ciphertext to context through associated data: tenant, app, object, version, algorithm, and protocol version.
- Never create custom cryptographic primitives, nonce formats, password handling, or key-derivation shortcuts.
- Treat key recovery, device loss, and administrator access as explicit threat-model decisions.

A cryptography review is required before production claims or interoperability guarantees.

## Core entities

- **Principal:** user, service, device, or app instance.
- **Application:** a registered consumer such as stamp.dev.
- **Session:** short-lived authenticated access context with audience and expiry.
- **Device keyset:** device-bound signing and key-agreement material, with rotation state.
- **Data-encryption key:** object or collection key used by Who remains; normally wrapped, not stored plaintext by the service.
- **Grant:** a scoped, expiring permission to unwrap or use a key for a specific object/action.
- **Key epoch:** a rotation boundary that makes old and new grants distinguishable.
- **Audit event:** append-only record of authentication, grant, rotation, revocation, and recovery actions without plaintext secrets.

## Primary flows

1. **Register application** — create an app identity, allowed audiences, redirect origins, and key policy.
2. **Authenticate principal** — verify credentials or an external identity, then issue a short-lived audience-bound session.
3. **Enroll device** — bind a device keyset using a one-time challenge; show recovery consequences clearly.
4. **Request access** — app asks for a narrowly scoped grant for an object, action, and time window.
5. **Encrypt or unwrap** — the client obtains only the key material needed for the operation; plaintext stays client-side where possible.
6. **Rotate** — create a new key epoch, rewrap or lazily migrate affected objects, and invalidate selected grants.
7. **Revoke** — disable a principal/device/grant and make future access fail without silently deleting data.
8. **Recover** — use an explicit recovery policy; never hide a server backdoor behind a vague admin role.

## First vertical slice

Build the smallest local-only reference flow before distributed deployment:

1. create one app and one test user;
2. enroll one device keyset;
3. issue and validate a short-lived session with audience and expiry;
4. generate an object key and pass it to Who remains as a wrapped key;
5. encrypt/decrypt one small payload with associated data;
6. revoke the device and prove new access fails;
7. rotate the key epoch and prove old grants cannot use the new epoch;
8. emit redacted audit events.

## Agent workstreams

- **Protocol/spec agent:** versioned message schemas, error codes, canonical encoding, compatibility rules.
- **Crypto-wrapper agent:** audited-library adapter, key representations, zeroization boundaries, known-answer vectors.
- **Auth-service agent:** registration, sessions, audiences, device enrollment, revocation, rate limits.
- **SDK agent:** safe high-level API for apps; prevent callers from selecting unsafe algorithms or nonce values.
- **Integration agent:** stamp.dev adapter and Who remains key/grant handoff.
- **Security-review agent:** threat model, misuse cases, attack tests, dependency review, release gates.
- **Documentation agent:** recovery policy, operator model, examples, and migration notes.

## Verification gates

- Known-answer tests for every protocol version and supported primitive.
- Tamper, truncation, wrong-key, wrong-audience, expired-session, replay, and rollback tests.
- Key-rotation and device-revocation tests across restarts.
- Property/fuzz tests for parsers and canonical encodings.
- Secret redaction checks for logs, errors, traces, and test artifacts.
- Dependency and reproducible-build checks before any production use.
- Independent cryptographic review before calling the system secure.

## Open decisions

- Is the first deployment server-assisted, client-held-key, or a hybrid?
- What recovery model is acceptable when every device is lost?
- Can an operator recover data, or is the system intentionally unable to do so?
- Which single payload primitive and wire encoding are supported in v1?
- Are grants object-scoped, collection-scoped, or both?
- How are multi-device conflicts and clock skew handled?
- What metadata may remain visible to storage and transport layers?

## Non-goals for v1

- New cryptographic primitives.
- Global plaintext-based deduplication.
- Anonymous permanent sessions.
- A universal identity provider for unrelated third parties.
- Production security claims before review and attack testing.
