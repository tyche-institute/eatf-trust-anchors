# Key Ceremony — Demo PQC Genesis (2026-05-13)

> **DEMO CEREMONY.** Sibling of the classical RSA-4096 anchor recorded
> in [`2026-05-13-demo-genesis.md`](2026-05-13-demo-genesis.md). EATF
> signs hybrid (RSA + ML-DSA-65), so the trust-anchors mirror lists
> both halves of the keypair from manifest v2 onwards. The private key
> below is a developer demo key, **NOT** in HSM, **NOT** signing any
> production `.aep` package.

---

## Metadata

| Field | Value |
|---|---|
| Ceremony date (UTC) | 2026-05-13 |
| Ceremony purpose | Add ML-DSA-65 (Dilithium3) anchor alongside the RSA-4096 demo |
| Anchor kid generated | `kid_demo_pqc_genesis_2026_05` |
| Algorithm | ML-DSA-65 (FIPS 204 / CRYSTALS-Dilithium3, NIST level 3) |
| Fingerprint (SHA-256 of SPKI DER) | `E1:A8:88:E5:08:22:74:02:B8:72:A6:2C:B7:CE:8A:1E:65:10:97:DC:15:46:3F:AC:FF:A8:7F:7F:BE:05:5C:57` |
| Manifest version | 2 (chained from v1) |
| Previous manifest SHA-256 | `c48709b75076a80ab90db29c9620a5baa7f9af0a111458d7dc4b9cf4f3d2542c` |
| Witness role | None (single-operator demo, four-eyes waived) |
| HSM | None — keypair generated via the project's `ai.aletheia.crypto.PqcKeyGen` tool (BouncyCastle Dilithium3 implementation) on the operator dev machine |
| Backup | None — DEMO key, intentionally ephemeral |

## Why ML-DSA-65

EATF signs **hybrid**: every `.aep` carries an RSA-4096 PKCS#1 v1.5
signature (classical, for compatibility with the broad PKI ecosystem
relying parties already trust) AND an ML-DSA-65 signature (post-quantum,
for forward-secrecy against a future cryptographically-relevant quantum
computer). Both signatures cover the same canonical bytes; a verifier
that trusts either algorithm can accept the artefact.

The trust-anchors mirror lists **both** keys because both must be
verifiable. A relying party verifying an `.aep` from 2026 needs:

1. The RSA-4096 public key — for the classical signature.
2. The ML-DSA-65 public key — for the post-quantum signature.

Listing only the RSA half would silently downgrade EATF's security
posture and defeat the point of being post-quantum from day one. Hence
this manifest contains both anchors with the same `validFrom` window.

## What was done

1. **Keypair generation** — `mvn -q exec:java -Dexec.mainClass="ai.aletheia.crypto.PqcKeyGen"` from the `backend/` directory. Internally:
   ```
   DilithiumKeyPairGenerator gen = new DilithiumKeyPairGenerator();
   gen.init(new DilithiumKeyGenerationParameters(new SecureRandom(), DilithiumParameters.dilithium3));
   AsymmetricCipherKeyPair keyPair = gen.generateKeyPair();
   ```
2. **Public key extraction** — BouncyCastle's `SubjectPublicKeyInfoFactory.createSubjectPublicKeyInfo(...)` produces the standard X.509 SPKI wrapper, written as PEM to `ai_pqc_public.pem` (2730 bytes).
3. **Fingerprint** — base64-decode the PEM body to 1976 DER bytes, SHA-256 → uppercase hex with colons.
4. **Manifest update** — `trust-list.json` bumped to version 2, `previousManifestSha256` chained from v1, this anchor appended.
5. **Validator self-test** — `bash scripts/key-ceremony/validate-trust-list.sh trust-list.json` returned `OK` for both anchors. The validator is algorithm-agnostic — it base64-decodes the PEM body and recomputes SHA-256 over the DER, so the same code path validates RSA and Dilithium.
6. **Publication** — pushed `trust-list.json` (v2) + this ceremony log to `main`.

## OID note for relying parties

BouncyCastle's `SubjectPublicKeyInfo` for Dilithium3 uses the
round-3-era OID `1.3.6.1.4.1.2.267.7.6.5`, **not** the NIST FIPS 204
ML-DSA-65 OID `2.16.840.1.101.3.4.3.18`. This is a known transitional
state: BC 1.78.x keeps the round-3 OID for compatibility with the
existing EATF backend codepath (`FilePqcKeyProvider` →
`SubjectPublicKeyInfoFactory.createSubjectPublicKeyInfo`); a future
BC version may flip to the FIPS OID. Relying parties verifying `.aep`
packages must accept both OIDs as "ML-DSA-65". The fingerprint check
(SHA-256 of the full SPKI DER) is what binds the public key to the
anchor entry regardless of OID drift; verifiers should NOT pattern-match
on OID alone.

## Hash chain link

`previousManifestSha256`: `c48709b75076a80ab90db29c9620a5baa7f9af0a111458d7dc4b9cf4f3d2542c` (SHA-256 of v1 `trust-list.json`).

This locks v2 to v1 in a tamper-evident chain. A relying party can
walk the chain by retrieving every historical commit of
`trust-list.json` from the mirror's git history and verifying each
hash forward.

## How a production PQC ceremony would differ

Same matrix as the RSA-4096 ceremony, plus:

| Concern | Demo (this run) | Production |
|---|---|---|
| HSM | None — software BC | HSM with PQC algorithm support (currently rare; AWS CloudHSM does not yet support ML-DSA, so v1 production may still keep the PQC key in encrypted filesystem with Shamir-split passphrase while waiting for HSM vendor support) |
| Algorithm pin | Dilithium3 (round-3 BC) | Same Dilithium3 for now; migrate to FIPS-204 OID + parameters when BC ships it. Migration is a new ceremony with `retiredAt` on this anchor |

## Files in this ceremony

| Path | Description |
|---|---|
| `trust-list.json` (v2) | Manifest with both RSA-4096 and ML-DSA-65 anchors. |
| `ceremony-logs/2026-05-13-demo-pqc-genesis.md` | This file. |
| `ceremony-logs/2026-05-13-demo-genesis.md` | The v1 RSA ceremony log (unchanged). |

The demo PQC private key (`ai_pqc.key`) is **NOT** published; it lives
in `/tmp` on the operator machine.

---

*Real production runbook: maintained internally by the operator
(currently focused on RSA; PQC-specific HSM guidance is tied to
vendor availability and will be added once HSM support for ML-DSA
matures).*
