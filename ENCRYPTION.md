# Rhooa Brain — Encryption Design

Deep-dive companion to [ARCHITECTURE.md](ARCHITECTURE.md) §6. Everything cryptographic in one place: algorithms, key hierarchy, lifecycle, memory handling, and the erasure story.

**Design rules:** boring, standard, misuse-resistant cryptography only — no custom constructions; every key has exactly one job; deleting a key must be equivalent to deleting the data it protects (crypto-erasure); assume the disk will be stolen and the cloud will be subpoenaed.

---

## 1. Threat model (what crypto must and must not solve)

| Threat | In scope? | Answer |
|---|---|---|
| Stolen/lost laptop, disk pulled | ✅ | Encrypted store, keys bound to the Windows user (DPAPI/TPM) — disk alone is ciphertext |
| Another *user account* on the same PC | ✅ | Per-Windows-user DPAPI scope; OS ACLs on the data directory |
| Cloud breach / subpoena of Rhooa infra | ✅ | Brain data isn't there; transit is ephemeral; only dirty-flags & billing counters exist server-side |
| Network attacker (LAN, hostile Wi-Fi, MITM) | ✅ | TLS 1.3 + certificate pinning to the gateway; loopback WSS on the bridge |
| Malicious website / tab | ✅ | Bridge auth crypto (§5) — origin-lock + signed JWTs + pairing binding |
| Same-user malware with full user privileges | ⚠️ partial | It can do whatever the agent can (incl. asking DPAPI). Crypto cannot fix this; OS hardening reduces it (§7). Stated honestly in the threat model — this is true of every local-data product |
| Forensic recovery after purge | ✅ | Crypto-erasure: destroy keys → all ciphertext is garbage (§8) |

---

## 2. Algorithm suite (fixed, versioned as `crypto_suite: 1`)

| Purpose | Algorithm | Notes |
|---|---|---|
| Symmetric encryption (DB, files, exports) | **AES-256-GCM** | Hardware-accelerated (AES-NI) on effectively all target PCs; ChaCha20-Poly1305 as the fallback suite for TLS |
| Key wrapping | **AES-256-GCM** with random 96-bit nonces + key-commitment tag check | Every wrapped key stores `{nonce, ciphertext, tag, kek_id, suite}` |
| Key derivation (from secrets) | **HKDF-SHA-256** | Context strings namespace every derived key (`"rhooa/db"`, `"rhooa/export"`, …) |
| Password-based derivation (exports only) | **Argon2id** (m=64 MiB, t=3, p=1 minimum) | Only place a human passphrase ever meets crypto |
| Signatures (JWT, licenses, policy files, updates) | **Ed25519** | Small, fast, misuse-resistant; keys distributed as JWKS with `kid` |
| Key agreement (pairing) | **X25519** | Ephemeral, inside the pairing handshake (§5.1) |
| Content hashing (dedup, audit chain) | **BLAKE3** | Fast enough to hash the whole corpus on CPU; SHA-256 accepted where interop requires |
| TLS | **TLS 1.3 only** to Rhooa endpoints | Pinned SPKI (§6.1) |

Every ciphertext record and signature carries its `suite`/`kid` version → future migration is a background re-encrypt, never a flag day.

---

## 3. Key hierarchy (at rest)

```
Windows DPAPI (user-scoped)  [+ TPM via CNG Platform Provider when available]
        │ protects
        ▼
   KEK  (Key-Encryption Key, 256-bit, random, generated at install)
        │ wraps (AES-256-GCM)
        ▼
 ┌──────┴────────────┬───────────────────┬──────────────────┬────────────────┐
 ▼                   ▼                   ▼                  ▼                ▼
DEK-db              DEK-cache           DEK-audit          DEK-adapters     DEK-export (ephemeral)
SQLCipher key for   extracted-text &    audit log          v2 LoRA          per-export, derived
vectors+graph+      thumbnail cache     encryption +       adapter files    from user passphrase
memory (one DB)     files               MAC key (§6.3)                      via Argon2id
```

Rules:

1. **One KEK, many DEKs, one job each.** A compromise or rotation of any DEK never touches the others; wrapped DEKs live in a small keyring file next to the DB.
2. **KEK protection ladder:** DPAPI (baseline, every Windows machine) → **TPM-sealed via CNG Platform Crypto Provider when hardware allows** (key never extractable, survives only on that machine) → optional Windows Hello gate for the paranoid tier (biometric prompt unlocks the Brain per session). The agent picks the strongest available at install and records which; policy files can mandate a minimum.
3. **Nothing key-like ever exists in plaintext on disk.** Not in config, not in logs, not in crash dumps (§7).
4. **SQLCipher specifics:** raw 256-bit DEK-db passed directly (no PBKDF inside SQLCipher — derivation already happened), `cipher_page_size 4096`, per-page HMAC integrity on, memory security on (`PRAGMA cipher_memory_security = ON`).
5. **Key rotation:** KEK rotates by re-wrapping DEKs (instant, no data re-encryption); DEKs rotate by background re-encryption with dual-key read window; both are audit-logged events with key generation numbers, never key material.

---

## 4. What exactly is encrypted at rest

| Artifact | Protection |
|---|---|
| Vector store + graph + memory + metadata (single SQLite DB) | SQLCipher / DEK-db |
| Extracted-text & preview cache | AES-256-GCM files / DEK-cache; filenames are BLAKE3 hashes, no plaintext leakage via names |
| Audit log | Encrypted (DEK-audit) **and** hash-chained (§6.3) |
| Ingest upload queue (chunks awaiting single-transit) | Encrypted at rest with DEK-cache; deleted after ack |
| OAuth/refresh tokens for integrations (INTEGRATIONS.md) | DPAPI directly (they're small, OS store fits) + Credential Manager entries |
| v2 LoRA adapters | AES-256-GCM / DEK-adapters — trained weights *are* company data |
| Crash dumps / temp files | Dumps scrubbed of key regions (§7); temp extraction files live under the encrypted cache dir, never `%TEMP%` |
| **Instant Web Mode (browser)** | OPFS ciphertext: AES-256-GCM via WebCrypto, key held as a **non-extractable CryptoKey** in IndexedDB — honest note: browser-grade isolation is weaker than DPAPI/TPM; this is a declared Instant-Mode limit and one more reason the full agent is the upgrade |

---

## 5. Bridge & identity cryptography

### 5.1 Pairing (the trust root between Brain and account)
The pairing code is a **low-entropy human secret**, so it must never be sent — it feeds a **PAKE (SPAKE2)**:

1. Agent generates a 6–8 char pairing code, shows it in the tray UI.
2. User types it in the web app. Browser and agent run SPAKE2 over the code → both derive the same strong session secret **without the code (or anything offline-bruteforceable) crossing the wire**; 3 failed attempts burn the code.
3. Inside that authenticated channel: the cloud presents the account (`user_id`, tenant), the agent presents a freshly generated **device keypair (Ed25519)**; each signs the binding `{user_id, device_pub, tenant, timestamp}`.
4. Result stored: agent holds the binding + device private key (DPAPI/TPM); cloud registers `device_pub` under the seat.

**B2B silent enrollment:** the MSI carries a short-lived, tenant-scoped, Ed25519-signed **enrollment token** (single-use per machine, delivered via Intune, auditable). At the user's first SSO login on that machine, the same binding ceremony runs automatically — SSO identity instead of a typed code.

### 5.2 Session auth (per ARCHITECTURE §3, crypto view)
- Access JWTs: **Ed25519 (EdDSA)**, 15 min TTL, verified offline against cached JWKS (`kid`-addressed, rotation-ready), ±60 s skew.
- The agent additionally requires `sub == paired user_id` — the signature proves Rhooa issued it; the binding proves it's the *owner*.
- MCP calls (§8a) inherit the same session; the agent identity claim (`agent_id`, scopes) is inside the signed JWT, so scope spoofing requires forging a signature.
- **Device attestation option (strict tier):** the agent signs bridge responses with its device key so the cloud can verify answers really came from the enrolled machine, not an emulator.

---

## 6. Data in transit

### 6.1 To Rhooa cloud (`/brain/ingest`, chat, refresh, metering)
- TLS 1.3 exclusively, **SPKI pinning** to Rhooa gateway keys (pin set of 2 — current + next — shipped with agent updates so rotation never bricks agents).
- Ingest batches are additionally **compressed then encrypted per-batch** with an ephemeral key HKDF-derived from the TLS-exported session secret — defense-in-depth against TLS-terminating middleboxes: corporate MITM proxies see nothing even where IT force-installs roots. (Flag: this breaks DLP-inspection expectations of some enterprise IT — policy file can disable it *explicitly*, making the trade visible instead of silent.)
- Zero-retention: the gateway processes in memory; no request-body logging at any tier — contractual + verified in SOC 2 audit scope.

### 6.2 Browser ↔ agent (loopback)
- WSS on 127.0.0.1 with a per-install self-signed cert whose SPKI hash the web app learns **during pairing** (inside the SPAKE2 channel) and pins thereafter — solves the "browsers distrust localhost certs" problem without touching the OS trust store (with the documented per-browser `ws://` loopback-exemption fallback, ARCHITECTURE §3).
- Session messages carry the JWT; no additional message-level crypto needed on loopback beyond WSS — the threat there is other *local* processes, handled by OS-level loopback semantics + auth, not ciphers.

### 6.3 Audit log integrity
- Append-only, each entry: `hash_n = BLAKE3(hash_{n-1} ‖ entry)`; daily checkpoint heads signed with the device key.
- Tampering (deletion, reordering, edit) breaks the chain verifiably; export for compliance carries the signed heads.

---

## 7. Keys in memory (the part everyone skips)

- Agent in Rust: all key material in `zeroize`-on-drop containers; keys live in **locked pages** (`VirtualLock`) so they never hit the pagefile.
- Crash handling: minidumps configured to exclude locked key regions; no key material in logs at any level, enforced by typed "secret" wrappers whose `Debug`/`Display` print `[REDACTED]`.
- DEKs unwrapped lazily, per subsystem, dropped on idle timeout; the KEK plaintext exists in memory only for microseconds during unwrap.
- Parser sandbox children (untrusted-file handlers) receive **no keys at all** — they read plaintext bytes from a pipe and return extracted text; encryption happens only in the parent.

---

## 8. Erasure & lifecycle

- **Crypto-erasure is the primary delete:** "Forget everything" destroys the keyring (DPAPI-unprotect, zeroize, overwrite, delete) → every byte of Brain ciphertext on disk is permanently meaningless, instantly — no multi-hour secure-wipe theater. File deletion follows as hygiene.
- **Per-source purge** (a folder, an integration): those chunks/nodes/vectors are deleted from the DB (SQLCipher `secure_delete` on), and provenance cascade (GRAPH-TRAINING §2) guarantees completeness. Full crypto-erasure remains the nuclear option.
- **Uninstall = purge** by default (policy can require it).
- **Backups/exports:** encrypted container (age-style: X25519 recipient or Argon2id passphrase), including keyring *contents* re-wrapped under the export key — restorable on a new machine only with the passphrase; export events are audit-logged and policy-gateable (IT may forbid exports entirely).
- **Instant→Agent migration** (ARCHITECTURE §4a): the OPFS mini-brain export crosses the bridge inside the paired WSS channel, re-encrypted under the agent's DEKs on arrival, then OPFS copy destroyed.

---

## 9. Compliance mapping (quick crosswalk)

| Requirement | Where satisfied |
|---|---|
| GDPR Art. 17 (erasure) | §8 crypto-erasure + provenance cascade |
| GDPR Art. 32 (state of the art protection) | §2 suite, §3 hierarchy, §6 transit |
| ISO 27001 A.8.24 / SOC 2 CC6 (crypto & key mgmt) | §3 lifecycle, rotation, audit-logged key events |
| Right of access / transparency | "What was sent" inspector + signed audit chain (§6.3) |
| Data residency | Trivially satisfied — the data *resides* on the client's own machine |

---

## 10. Build vs. adopt — the OSS map (don't reinvent the wheel)

No single repo does all of this, but ~90% assembles from proven, actively-maintained open source (status verified July 2026). What remains custom is thin glue: the keyring file format, the audit hash chain (~200 lines), and policy enforcement.

| Need (§) | Adopt | Notes |
|---|---|---|
| Encrypted SQLite (§3, §4) | **SQLCipher** (BSD, community edition) or **SQLite3MultipleCiphers** (CC0, v2.3.5 Jun 2026, ChaCha20-Poly1305 default, benches faster than AES on CPUs w/o AES-NI) | Both battle-tested; pick SQLite3MultipleCiphers if we want ChaCha20 + zero license friction; `rusqlite`/`sqlx` bindings exist for both |
| AEAD, HKDF, Argon2id, BLAKE3, Ed25519/X25519 (§2) | **RustCrypto** crates (`aes-gcm`, `chacha20poly1305`, `hkdf`, `argon2`), **`blake3`**, **`ed25519-dalek`/`x25519-dalek`** | The standard Rust stack; audited, ubiquitous. `ring` is the alternative single-dep option |
| Misuse-resistant "just encrypt this" layer | **libsodium** (via `dryoc` pure-Rust port) — optional | Only if we prefer NaCl-style APIs over RustCrypto composition |
| PAKE pairing (§5.1) | **`spake2`** (RustCrypto/PAKEs, updated Jan 2026) — the exact crate magic-wormhole uses; **`opaque-ke`** if we later want OPAQUE | Magic-wormhole *is* the reference UX for code-based pairing — steal its ceremony design |
| Encrypted exports/backups (§8) | **age / `rage`** (Rust) | X25519 recipients + passphrase mode out of the box; de-facto standard format |
| JWT EdDSA + JWKS (§5.2) | **`jsonwebtoken`** crate (EdDSA support) | Server side: any mainstream JOSE lib |
| TLS 1.3 + SPKI pinning (§6.1) | **`rustls`** with a custom certificate verifier | Pinning is a ~50-line verifier, not a product |
| DPAPI / Credential Manager (§3) | **`keyring`** crate + **`windows-rs`** (`CryptProtectData`) | Thin, well-trodden |
| TPM sealing (§3) | **CNG Platform Crypto Provider** via `windows-rs`; **`tss-esapi`** if we go TPM2-stack direct | CNG route is the Windows-native, least-friction one |
| Key memory hygiene (§7) | **`zeroize`** + **`secrecy`** + **`region`** (VirtualLock) | Exactly what they're for |
| Secrets vault alternative | IOTA **Stronghold** | Evaluate-only: could replace our keyring file, but adds a heavy dependency for what §3 does in little code |
| Reference implementations to study (not depend on) | **Bitwarden clients**, **Standard Notes** | Open-source KEK/DEK hierarchies + export formats in production for years — sanity-check our §3 against theirs |

**Rule for the codebase:** we write *zero* cryptographic primitives. Custom code is limited to: keyring format (wrapped-DEK records), audit chain append/verify, policy checks, and wiring. Each of those is small, deterministic, and unit-testable — the dangerous parts are all adopted.

## 11. Open items (decide before Phase 1 code)

1. TPM-sealing default-on vs opportunistic (recovery UX if the motherboard dies: without export, a TPM-sealed Brain is unrecoverable — that's a *feature* for security and a *ticket generator* for support; recommendation: opportunistic + loud "make an encrypted backup" nudge).
2. Whether Instant Web Mode warrants a passphrase-derived key (UX cost) or accepts browser-grade `CryptoKey` isolation (current design: accept + declare).
3. Pin-set update path if both pinned gateway keys must be replaced in an emergency (documented break-glass: signed agent update carries the new pin set).
