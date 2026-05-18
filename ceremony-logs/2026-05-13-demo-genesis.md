# Key Ceremony — Demo Genesis (2026-05-13)

> **DEMO CEREMONY.** This document records the bootstrap of the
> `eatf-trust-anchors` mirror at version 1. The keypair was generated on
> a developer machine to populate the manifest format end-to-end; the
> resulting private key is **NOT** used to sign any production EATF
> attestation, ledger block, or audit-archive batch. Use it as a
> template for what a real ceremony log looks like, then run the real
> ceremony per the maintainer's internal key-ceremony runbook (not public) and append a new anchor entry that retires this one.

---

## Metadata

| Field | Value |
|---|---|
| Ceremony date (UTC) | 2026-05-13 |
| Ceremony purpose | Bootstrap — establish manifest format on the mirror |
| Anchor kid generated | `kid_demo_genesis_2026_05` |
| Algorithm | RSA-4096, PKCS#8 |
| Fingerprint (SHA-256 of SPKI DER) | `62:63:3B:14:6B:BF:2A:C1:D6:87:49:CC:F5:17:A2:CC:F9:E5:66:12:55:F4:C3:59:39:A6:5E:00:65:43:8F:37` |
| Manifest version | 1 (genesis) |
| Previous manifest SHA-256 | `""` (no predecessor) |
| Witness role | None (single-operator demo, four-eyes waived) |
| HSM | None — keypair generated via `cryptography` v45.x on the operator dev machine, private key stored on local filesystem under `/tmp` |
| Backup | None — DEMO key, intentionally ephemeral |

## What was done

1. **Keypair generation** — RSA-4096 via Python `cryptography.hazmat.primitives.asymmetric.rsa.generate_private_key(public_exponent=65537, key_size=4096)`. PKCS#8 PEM written to `demo-signing-key.pem` (`NoEncryption()` — demo).
2. **Public key extraction** — SubjectPublicKeyInfo PEM written to `demo-public-key.pem` (800 bytes).
3. **Fingerprint** — SHA-256 of the public key DER (X.509 SPKI), uppercase hex with colons. Recorded in `fingerprint.txt`.
4. **Manifest construction** — `trust-list.json` v1 built per profile `urn:eatf:spec:key-mirror:1.0` (manifest structure described in this repository's `README.md`). Includes only this single anchor.
5. **Validator self-test** — the inline validator from `README.md` (openssl + jq + xxd) returned `OK` for the anchor. The validator recomputes the fingerprint from the embedded PEM and asserts equality with `fingerprintSha256`.
6. **Mirror repo creation** — operator workstation created the public mirror repository at its initial location (subsequently transferred to the `tyche-institute/` GitHub organisation).
7. **Publication** — `trust-list.json` + this ceremony log + `README.md` + `LICENSE` pushed to `main`.

## What a REAL ceremony would do differently

A production ceremony (per the operator's internal key-ceremony runbook) replaces or adds the following:

| Concern | Demo (this run) | Production |
|---|---|---|
| Operator count | 1 | 2+ (four-eyes principle) |
| Witness | None | Independent reviewer signs a witness statement |
| Recording | None | Audio + video of the ceremony, archived |
| HSM | Software / filesystem | AWS CloudHSM or Azure Dedicated HSM via PKCS#11 |
| Private key passphrase | `NoEncryption()` | Shamir-split passphrase across N=5 K=3 keyholders |
| Private key backup | None — demo key | Encrypted offline backup, geographically distributed |
| Previous-manifest hash | `""` | SHA-256 of the prior manifest (chain link) |
| `signedByPreviousKidBase64` | Omitted | RSA-4096 signature of this anchor's `publicKeyPem` SHA-256, produced by the predecessor key |
| Validation | Single `validate-trust-list.sh` run on the operator's machine | Validator run on TWO independent workstations, fingerprints compared by hand |

## How to retire this anchor when the real ceremony happens

1. Run the real ceremony per the operator's internal key-ceremony runbook. This produces:
   - A new private key in HSM
   - A new public PEM
   - A new ceremony log file under `ceremony-logs/<date>-<purpose>.md`
2. Build the new manifest:
   - Take the existing `trust-list.json`
   - Bump `version` to `2`
   - Set `previousManifestSha256` to `sha256(<bytes of v1 trust-list.json>)` in lowercase hex
   - Append the new anchor to `anchors[]`
   - **Update this anchor's entry** by setting `retiredAt` to the new ceremony's `validFrom` timestamp
   - Optionally: produce `signedByPreviousKidBase64` for the new anchor using THIS demo key (one last operational use of the demo key before retirement)
3. Validate: `bash scripts/key-ceremony/validate-trust-list.sh trust-list.json` — must return `OK` for both anchors.
4. Commit + push to `main`.

The backend's `TrustAnchorsService` will pull the new manifest within 6 hours (or immediately via `POST /api/admin/keys/history/reload`).

## Hash chain link

`previousManifestSha256`: `""` — genesis manifest, no predecessor.

`This manifest SHA-256` (for the next ceremony to chain from): see
`trust-list.json.sha256` (computed by the publication push hook).

## Files in this ceremony

| Path | Description |
|---|---|
| `trust-list.json` | Manifest v1 — this is what `https://api.eatf.eu/api/public/keys/history` will serve. |
| `ceremony-logs/2026-05-13-demo-genesis.md` | This file. |
| `README.md` | Repository overview + retrieval instructions for relying parties. |
| `LICENSE` | MIT — the manifest is public-facing trust material; no restrictions on consumption. |

The demo private key (`demo-signing-key.pem`) is **NOT** published; it
lives in `/tmp` on the operator machine and is deleted at the end of
this session.

---

*This template is for demo / format documentation. The real ceremony
runbook is maintained internally by the operator and is not
published.*
