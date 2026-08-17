# ADR-0005: Field-Level Encryption via Adapter Pattern (AES-256-GCM)

Sensitive financial fields are encrypted at the application layer with AES-256-GCM behind an adapter interface; unimplemented adapters fail boot, never fail silently.

## Status

Accepted — amendable only via the ADR process (`docs/adr/README.md`).

## Date

2026-08-07

## Context

The PRD requires application-level encryption (AES-256) of sensitive financial data (transaction amounts, account names) with a master key managed by an external KMS or a local secret in development. v1's design was sound — an `EncryptionService` adapter with a local implementation — but its execution failed: `VaultEncryptionService` was a throwing stub while documentation claimed it worked. A security adapter that silently does nothing is worse than no adapter at all.

## Decision

- **Algorithm:** AES-256-GCM (authenticated encryption).
- **Storage layout:** `base64(IV || ciphertext || authTag)` with a fresh random 12-byte IV per encryption operation. Encrypted values are stored as plain strings in the Drizzle schema (e.g., `encryptedAmount`).
- **Key:** a 32-byte key, base64-encoded, loaded from an environment variable validated at boot by the zod env schema (ADR-0008). Missing or malformed key fails boot.
- **Adapter pattern:** an `EncryptionService` interface with swappable providers:
  - `LocalEncryptionService` — implemented now, key from validated env.
  - Vault/KMS provider — interface reserved for later. **An adapter that is selected but not implemented must fail boot.** No stub may ever masquerade as a working provider (the v1 failure mode this closes).
- **Catalog:** the authoritative list of encrypted fields lives in `docs/DATABASE.md`.
- **Operations:** the key-rotation runbook lives in `docs/OPERATIONS.md`.

## Alternatives Considered

| Option | Reason rejected |
|--------|-----------------|
| Database-level encryption (pgcrypto / TDE) | Keys live next to the data; no application-layer control; does not satisfy the PRD's KMS adapter requirement |
| KMS/Vault from day one | Operational cost before the deployment target justifies it; the adapter interface keeps the door open without paying now |
| Deterministic encryption (for queryability) | Leaks equality of plaintexts; financial fields are decrypted in the application layer for computation, so deterministic encryption buys nothing |

## Consequences

### Positive

- Encrypted fields are unreadable in database dumps and backups without the key.
- GCM authentication detects tampering; the IV||ciphertext||tag layout is self-describing.
- The provider swap (local → vault) is a configuration change plus a real implementation, with a hard boot-time guarantee that nothing unimplemented is in the path.

### Negative

- Encrypted fields cannot be filtered, sorted, or aggregated in SQL; queries must decrypt in the application layer.
- Key management is an operational responsibility: rotation runbook, backup of the key, and boot-time validation are mandatory (ADR-0008).
- Every new sensitive field requires a deliberate entry in the `docs/DATABASE.md` catalog.

## Related Documents

- ADR-0006 (tenant isolation), ADR-0008 (env validation at boot)
- `docs/DATABASE.md` §5 (encrypted-field catalog)
- `docs/OPERATIONS.md` (key rotation runbook)
- `docs/PRD.md` Module 11, `docs/SECURITY.md` §5 (cryptography)
