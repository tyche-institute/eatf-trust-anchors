# `eatf-trust-anchors`

**Public mirror of EATF historical signing-key trust anchors.**

[![EATF](https://img.shields.io/badge/EATF-Agent%20Trust%20Framework-orange)](https://eatf.eu)
[![Spec URN](https://img.shields.io/badge/spec-urn%3Aeatf%3Aspec%3Akey--mirror%3A1.0-blue)](#anchor-lifecycle)

This repository is the canonical out-of-band publication channel for
EATF's signing-key public material. Every key that has ever been used
to sign an EATF AEP evidence package, ledger block, or audit-log
archive lands here as an anchor entry — with the corresponding
ceremony log, validity window, and chain-of-custody signature from
the previous active key.

It exists so a relying party verifying an EATF artefact does NOT have
to trust the EATF backend at `api.eatf.eu` alone — the public keys live
in a separate, externally-mirrored, audit-trail-equipped GitHub repo.
Trust is anchored in this repository's commit history.

## What's here

- **`trust-list.json`** — the manifest itself. Profile URN
  `urn:eatf:spec:key-mirror:1.0`; the manifest structure is described
  inline in [Anchor lifecycle](#anchor-lifecycle) and
  [Verifying the manifest yourself](#verifying-the-manifest-yourself)
  below.
- **`ceremony-logs/`** — markdown logs of every key ceremony that
  produced an anchor in the manifest. One file per ceremony, named
  `YYYY-MM-DD-<purpose>.md`.
- **`LICENSE`** — MIT. The manifest is public trust material; nothing
  restricts consumption.

## How relying parties use this

### From the EATF backend (recommended)

The production backend at `api.eatf.eu` exposes the same manifest:

```bash
curl https://api.eatf.eu/api/public/keys/history
```

Look up a specific kid:

```bash
curl https://api.eatf.eu/api/public/keys/history/kid_demo_genesis_2026_05
```

The backend pulls this repository's `trust-list.json` every 6 hours and
caches it. If the EATF backend is down, fall through to the second
option.

### Direct from this repository

```bash
curl https://raw.githubusercontent.com/tyche-institute/eatf-trust-anchors/main/trust-list.json
```

The raw GitHub URL is the bypass path that lets you verify an AEP
package even if `api.eatf.eu` is unreachable. **This is the whole
point** — trust does not flow through a single operator.

### Verifying the manifest yourself

The manifest is a flat JSON file. For each anchor:

1. Take `publicKeyPem`, strip the PEM headers + whitespace, base64-decode → DER bytes.
2. Compute SHA-256 over those DER bytes.
3. Format as uppercase hex with colons every 2 characters.
4. Compare to `fingerprintSha256`. **MUST match.**

A minimal self-contained, algorithm-agnostic validator using `jq` +
`base64` + `sha256sum` + standard POSIX text tools (no `openssl`, so
it works equally for RSA, ECDSA, and ML-DSA anchors):

```bash
jq -c '.anchors[]' trust-list.json | while read -r anchor; do
  kid=$(echo "$anchor" | jq -r '.kid')
  expected=$(echo "$anchor" | jq -r '.fingerprintSha256')
  computed=$(echo "$anchor" | jq -r '.publicKeyPem' \
    | sed -n '/-----BEGIN/,/-----END/{/-----/!p}' \
    | tr -d ' \n' \
    | base64 -d \
    | sha256sum \
    | awk '{print toupper($1)}' \
    | sed 's/\(..\)/\1:/g; s/:$//')
  [ "$expected" = "$computed" ] && echo "$kid  OK" || echo "$kid  MISMATCH"
done
```

## Anchor lifecycle

```
genesis  ──active──────────►  ┐
                              │ rotation ceremony — new manifest
                              ▼
         (retiredAt set)      next-active ──────────►
                                                     ┐
                                                     │ etc.
                                                     ▼
```

When a rotation happens, the **next** ceremony's manifest:

- Increments `version` (e.g. 1 → 2)
- Sets `previousManifestSha256` to the SHA-256 of the manifest it
  replaces (lowercase hex)
- Sets `retiredAt` on the OLD anchor entry
- Appends the NEW anchor with its own `validFrom`
- Optionally: sets `signedByPreviousKidBase64` on the new anchor — an
  RSA signature produced by the OLD private key, over the SHA-256 of
  the new anchor's `publicKeyPem`. This chains custody cryptographically.

**Anchors are append-only.** They are never deleted from the manifest,
only `retiredAt`-marked.

## Current status

| Version | Anchors | Status |
|---|---|---|
| 1 | 1 (`kid_demo_genesis_2026_05`) | **Demo / bootstrap** — replaces the demo seed manifest used during initial bring-up. The corresponding private key is a developer demo key, NOT in HSM, NOT signing prod artefacts. First real production ceremony will land as version 2. |

## Reporting an anchor mismatch

If you find an EATF artefact (`.aep`, ledger block, archive signature)
that references a kid NOT in this manifest, or where the embedded public
key does not match the manifest's `publicKeyPem` for that kid, **this is
a security incident**.

Report via the EATF coordinated disclosure channel:

- Email: `security@eatf.eu` (PGP key: `https://eatf.eu/.well-known/pgp.asc`)
- See [`SECURITY.md`](https://github.com/tyche-institute/eatf/blob/main/SECURITY.md)
  in the main `eatf` repository for the coordinated-disclosure policy
  and SLAs.

## Why a separate repository

A deployment that publishes its key material only on its own
infrastructure forces relying parties to trust that single operator.
A relying party should be able to verify that the manifest served by
the deployment's backend matches the manifest published at an
externally-mirrored location — that's the whole point of out-of-band
publication. The reference deployment at `api.eatf.eu` exposes the
same manifest at `/api/public/keys/history`; this repository is the
externally-mirrored copy against which that endpoint can be checked.

This repository is maintained by **Tyche Institute** under the
[`tyche-institute`](https://github.com/tyche-institute) GitHub
organisation. It is public, MIT-licensed, and historically verifiable
via GitHub's commit hashes. Future mirrors at archive.org, Zenodo,
and IPFS are planned.

EATF is **not** an eIDAS trust service under Regulation (EU)
910/2014 Article 3(16); see the
[`NOTICE`](https://github.com/tyche-institute/eatf/blob/main/NOTICE)
in the main `eatf` repository.

---

For the EATF framework itself — Agent Evidence Package (AEP) wire
format, reference implementations in TypeScript and Python, test
vectors, and threat model — see
[`tyche-institute/eatf`](https://github.com/tyche-institute/eatf) and
[eatf.eu](https://eatf.eu).
