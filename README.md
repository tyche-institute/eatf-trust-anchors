# EATF Public Key History Mirror

**Version:** 1.0-draft, dated 2026-05-12. Comment period open through 2026-08-12.
**Stable identifier:** `urn:eatf:spec:key-mirror:1.0`
**Phase:** 1 step 1.7 of the EATF roadmap.
**Status:** **mirror repo live with hybrid (RSA + ML-DSA) demo bootstrap, production ceremony pending.** Phase 1.7 v0.4 + v0.5 (2026-05-13) created [`github.com/tyche-institute/eatf-trust-anchors`](https://github.com/tyche-institute/eatf-trust-anchors) and published manifest v2 with **both halves of the hybrid signing pair** EATF actually uses:

- `kid_demo_genesis_2026_05` — **RSA-4096** classical anchor, fingerprint `62:63:3B:14:6B:BF:2A:C1:D6:87:49:CC:F5:17:A2:CC:F9:E5:66:12:55:F4:C3:59:39:A6:5E:00:65:43:8F:37`. See [`ceremony-logs/2026-05-13-demo-genesis.md`](https://github.com/tyche-institute/eatf-trust-anchors/blob/main/ceremony-logs/2026-05-13-demo-genesis.md).
- `kid_demo_pqc_genesis_2026_05` — **ML-DSA-65** (Dilithium3) post-quantum anchor, fingerprint `E1:A8:88:E5:08:22:74:02:B8:72:A6:2C:B7:CE:8A:1E:65:10:97:DC:15:46:3F:AC:FF:A8:7F:7F:BE:05:5C:57`. See [`ceremony-logs/2026-05-13-demo-pqc-genesis.md`](https://github.com/tyche-institute/eatf-trust-anchors/blob/main/ceremony-logs/2026-05-13-demo-pqc-genesis.md).

Both private keys are **demo** keys generated on the operator workstation and **not** in HSM. The format end-to-end is now exercised: schema → seed → validator → mirror repo → backend `TrustAnchorsService` → `/api/public/keys/history` → audit-archive restore fallback — and the manifest now reflects the hybrid signature scheme EATF actually uses rather than silently downgrading to RSA-only. Manifest v2 is hash-chained from v1 via `previousManifestSha256`. Production ceremony per [`docs/internal/key-ceremony.md`](../internal/key-ceremony.md) will land as manifest v3 with `retiredAt` set on both demo anchors. Backend prod profile pulls the mirror at `https://raw.githubusercontent.com/tyche-institute/eatf-trust-anchors/main/trust-list.json` by default; override via `AI_ALETHEIA_TRUST_ANCHORS_MANIFEST_PATH` for pinned-revision deployments.

> **One-sentence guarantee.** Every public key that EATF has ever used
> to sign an `.aep` evidence package is permanently published, in a
> tamper-evident chain, at multiple locations that the maintainer team
> does not unilaterally control.

---

## 1. Why a mirror exists

Verification of an `.aep` is offline by design — the relying party
holds the public keys needed inside the package itself. But over a
10-year audit horizon a verifier may also need:

- To confirm that the key that signed a 2026 package was indeed the
  active EATF key in 2026 (defends against "EATF rotated and replays
  history" attacks).
- To rebuild the chain of trust if a future ML-DSA scheme retires
  ML-DSA-65 and we need to bridge.
- To verify that a key shown in an `.aep` was not silently revoked.

A single canonical location for "the EATF public-key history" answers
all three. We put it under a separate GitHub repository owned by the
maintainer team, plus three independent mirrors so any one going dark
does not break the chain.

## 2. Canonical location

**Primary:** `https://github.com/tyche-institute/eatf-trust-anchors`

A separate, intentionally tiny GitHub repository. Layout:

```
eatf-trust-anchors/
├── README.md                 # Plain-English explainer and chain rules
├── trust-list.json           # Machine-readable trust list (§4)
├── genesis/                  # First-ever keys, immutable
│   ├── kid_rsa_2026-genesis.pem
│   ├── kid_mldsa65_2026-genesis.pem
│   └── ceremony-log.md
├── 2026-Q1/                  # One folder per rotation
│   ├── kid_rsa_2026-Q1.pem
│   ├── kid_rsa_2026-Q1.signed-by-previous     # ECDSA sig from previous active key
│   ├── kid_mldsa65_2026-Q1.pem
│   ├── kid_mldsa65_2026-Q1.signed-by-previous
│   ├── ceremony-log.md
│   └── fingerprints.txt
├── ...
└── revocations.json          # Append-only revocations with timestamps
```

## 3. Mirrors

| Mirror | Refresh cadence | Purpose |
|---|---|---|
| `github.com/tyche-institute/eatf-trust-anchors` | On every rotation | Primary |
| `web.archive.org/web/eatf-trust-anchors` | Monthly + on rotation | Drift detection |
| IPFS (CID published in the GitHub repo) | On every rotation | Censorship resistance |
| Zenodo DOI | Annually + on rotation | Long-term EU-funded preservation |

A verifier MAY consult any one of the mirrors; if they disagree, the
verifier MUST trust the entry that is signed by the previous active
key and reject the one that is not (chain-of-trust rule, §4).

## 4. Trust list format

`trust-list.json` is the machine-readable index. v1 schema:

```json
{
  "schema": "urn:eatf:spec:key-mirror:1.0",
  "issuer": {
    "name": "EATF.eu",
    "url": "https://eatf.eu",
    "framework_ops_version": "urn:eatf:framework-ops:1.0"
  },
  "generated_at": "2026-05-12T00:00:00Z",
  "keys": [
    {
      "kid": "kid_rsa_2026-genesis",
      "algorithm": "RSA-4096",
      "valid_from": "2026-01-01T00:00:00Z",
      "valid_until": null,
      "active": true,
      "fingerprint_sha256": "f3a1...",
      "pem_path": "genesis/kid_rsa_2026-genesis.pem",
      "signed_by_previous": null,
      "ceremony_log": "genesis/ceremony-log.md",
      "revoked": null
    },
    {
      "kid": "kid_mldsa65_2026-genesis",
      "algorithm": "ML-DSA-65",
      "valid_from": "2026-01-01T00:00:00Z",
      "valid_until": null,
      "active": true,
      "fingerprint_sha256": "9c2b...",
      "pem_path": "genesis/kid_mldsa65_2026-genesis.pem",
      "signed_by_previous": null,
      "ceremony_log": "genesis/ceremony-log.md",
      "revoked": null
    }
  ],
  "revocations": "revocations.json"
}
```

Rotation rules:

- A new key entry is appended only after a successful ceremony
  (Phase 1 step 1.5).
- Each new entry MUST cite `signed_by_previous` — the path to a file
  containing a signature, computed by the previous active key for the
  same algorithm, over the SHA-256 fingerprint of the new key. Genesis
  entries set this to `null`.
- An old entry is never deleted. When a key is rotated, the old entry
  remains `active=false` with a `valid_until` timestamp.
- Revocations are written to a separate append-only `revocations.json`
  AND the key's `revoked` field is updated to a non-null object with
  `revoked_at` and `reason`. Verifiers respect a revocation only when
  validating an `.aep` whose `metadata.created_at` is AFTER the
  revocation timestamp — earlier packages remain trusted.

## 5. Publication script

`scripts/publish-key-history.sh` (Phase 1 step 1.7 deliverable) drives
the publication. Outline (real script will be committed alongside this
spec):

```bash
#!/usr/bin/env bash
# Usage: publish-key-history.sh <new-kid> <pem> <previous-kid> <previous-key>
set -euo pipefail

NEW_KID="$1"
NEW_PEM="$2"
PREV_KID="$3"
PREV_KEY="$4"

# 1. Verify fingerprint matches the public ceremony log.
# 2. Sign the new fingerprint with the previous key.
# 3. Copy into the trust-anchors repo working tree.
# 4. Regenerate trust-list.json from disk + manual valid_until updates.
# 5. Commit, push.
# 6. Re-publish to archive.org via SavePageNow API.
# 7. Pin to IPFS (use a Pinata or Web3.Storage account; the API key
#    lives in 1Password and is NOT in this repo).
# 8. Update the Zenodo deposit (DOI-anchored) annually.
# 9. Print SHA-256 of trust-list.json + the IPFS CID for the
#    ceremony log.

echo "Done. Verify at: https://github.com/tyche-institute/eatf-trust-anchors"
```

## 6. Verifier usage

Offline verifiers use the keys embedded in the `.aep`. The mirror is
consulted only when:

- A verifier wants to confirm "this kid is the EATF key from this date"
  beyond what is in the package.
- A verifier wants to know if a key has been revoked.
- A long-living verifier wants to build a complete chain over the
  EATF lifetime.

Verifiers MAY mirror the trust list locally and check for updates on
their own schedule.

## 7. Recovery scenarios

| Scenario | Behaviour |
|---|---|
| Primary mirror taken down | Verifier consults `archive.org` snapshot. |
| Primary mirror tampered (silent rewrite) | The new entry without a `signed_by_previous` value fails the verifier's chain check; archive.org diff also exposes the rewrite. |
| Both primary and archive.org compromised | Verifier consults IPFS CID published in earlier announcements and Zenodo DOI. |
| EATF disappears | See `docs/legal/termination-plan.md` § 4. The mirror remains under the maintainer team's repository for as long as GitHub honours its archive policy; archive.org + IPFS + Zenodo preserve beyond that. |

## 8. Open questions

- Should we operate the mirror as a separate organisation (e.g.
  `eatf-trust` GitHub org) rather than the maintainer's personal
  account? Yes, target Phase 2.15 when ISO 27001 work makes the
  governance separation natural.
- Should we adopt the IETF "Transparency Service" framework
  (draft-ietf-cose-merkle-tree-proofs, etc.) for the trust list?
  Tracking; not in v1 to keep the dependency surface minimal.

## 9. Related documents

- `docs/legal/framework-operations.md` — non-TSP description of how the reference implementation is operated.
- `docs/legal/project-sustainability-plan.md` (replaces the earlier Termination Plan).
- `docs/internal/key-ceremony.md` (Phase 1 step 1.5).
- `docs/specs/aep-profile-v1.md` (Phase 1 step 1.2).
- `scripts/publish-key-history.sh` — executable companion (planned).

## 10. Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0-draft | 2026-05-12 | Initial draft. Phase 1 step 1.7. |
