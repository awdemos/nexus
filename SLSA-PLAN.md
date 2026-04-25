# SLSA Level 3 Compliance Plan for Nexus

> **Premise:** Agentic threats don't just write code — they modify build pipelines, inject dependencies, and forge artifacts. Supply-chain security is not a feature; it is the foundation. SLSA Level 3 is the minimum viable posture for a sovereign IDE that integrates external AI agents and LLM routers.

## Threat Model: Why SLSA-3 Is the Floor

| Threat | Vector | SLSA-3 Countermeasure |
|--------|--------|----------------------|
| **Poisoned binary** | Attacker replaces release artifact post-build | Signed provenance + transparency log |
| **Build tampering** | Malicious CI step injects code during compilation | Hosted builder with non-exportable signing key |
| **Dependency confusion** | Agent or attacker swaps a crate for a malicious version | SBOM + dependency checksum verification |
| **Source forgery** | Force-push or rebase hides malicious commits | Immutable tag + commit signing |
| **Replay attack** | Old vulnerable release re-published as "new" | Rekor timestamp + monotonic version enforcement |
| **Insider threat** | Maintainer key stolen, artifacts signed maliciously | Ephemeral Fulcio certs (no long-lived keys) |

## Architecture: Sigstore-First

We do not manage long-lived signing keys. Ever.

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub Actions                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Build Nexus  │  │ slsa-github  │  │ Cosign (keyless) │  │
│  │ (cargo build)│  │ -generator   │  │ sign artifacts   │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │            │
│         └─────────────────┴────────────────────┘            │
│                           │                                 │
│                    Provenance Attestation                     │
│                           │                                 │
└───────────────────────────┼─────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │        Fulcio              │  ← OIDC identity → ephemeral cert
              └─────────────┬─────────────┘
                            │
              ┌─────────────┴─────────────┐
              │        Rekor               │  ← Transparency log (immutable)
              └───────────────────────────┘
```

### Components

| Component | Role | Why We Use It |
|-----------|------|---------------|
**Cosign** | Sign container images and blobs | Keyless signing via OIDC; no key management ceremony |
**Fulcio** | OIDC-based CA | Issues short-lived certificates bound to GitHub Actions identity |
**Rekor** | Transparency log | Public, append-only log of all signatures; detects replay and rollback |
**slsa-github-generator** | Provenance generation | Official Google/GitHub tool for SLSA L3 provenance attestation |
**Gitsign** | Commit signing | Sigstore-powered `git commit -S` without GPG key management |

## Implementation Phases

---

### Phase 0: Baseline Hygiene (Week 0)

Before we emit a single signature, the repository must be hardened.

- [ ] **Branch protection on `nexus`**
  - Require pull request reviews (even for admins)
  - Require status checks to pass before merge
  - Dismiss stale PR approvals when new commits are pushed
  - Restrict push access to `main` and `nexus` branches

- [ ] **Require signed commits on `nexus`**
  - Enable "Require signed commits" in branch protection
  - Document Gitsign setup for contributors

- [ ] **Pin all GitHub Actions by SHA**
  - Replace `actions/checkout@v4` with `actions/checkout@<full-sha> # v4.1.1`
  - Prevent tag-hijacking attacks on third-party actions

- [ ] **Dependabot + Cargo audit**
  - Enable Dependabot for Rust (`cargo`) dependencies
  - Add `cargo audit` to CI on every PR
  - Fail CI on critical/high CVEs

- [ ] **Remove unused secrets**
  - Audit `Settings > Secrets and variables > Actions`
  - Rotate any long-lived tokens (AWS, crates.io, etc.)

---

### Phase 1: Provenance Generation (Week 1)

Generate SLSA Level 3 provenance for every release artifact.

**Action:** Add `.github/workflows/release.yml` using the official `slsa-github-generator`.

```yaml
# SLSA Level 3 provenance for Nexus release artifacts
name: SLSA Release
on:
  release:
    types: [created]

jobs:
  build:
    outputs:
      digests: ${{ steps.hash.outputs.digests }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - name: Build release artifacts
        run: script/build-release
      - id: hash
        run: |
          sha256sum target/release/*.dmg target/release/*.exe target/release/*.AppImage > checksums.txt
          echo "digests=$(cat checksums.txt | base64 -w0)" >> "$GITHUB_OUTPUT"

  provenance:
    needs: [build]
    permissions:
      actions: read
      id-token: write
      contents: write
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@<sha>
    with:
      base64-subjects: "${{ needs.build.outputs.digests }}"
      upload-assets: true
```

**What this gives us:**
- Build runs on GitHub-hosted runner (trusted build service)
- Provenance attestation is generated by a **separate** workflow with **no access to build secrets**
- Provenance file is uploaded alongside release artifacts

---

### Phase 2: Keyless Signing with Sigstore (Week 1–2)

Sign every artifact using Cosign + Fulcio + Rekor.

**Action:** Add a signing job to the release workflow.

```yaml
  sign:
    needs: [build, provenance]
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for Fulcio
      contents: write
    steps:
      - uses: actions/checkout@<sha>
      - uses: sigstore/cosign-installer@<sha>

      - name: Sign artifacts with Cosign (keyless)
        run: |
          for file in target/release/*.{dmg,exe,AppImage}; do
            cosign sign-blob "$file" \
              --output-certificate="${file}.crt" \
              --output-signature="${file}.sig"
          done
        env:
          COSIGN_EXPERIMENTAL: 1

      - name: Upload signatures to release
        run: |
          gh release upload "${{ github.ref_name }}" target/release/*.{crt,sig}
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**What this gives us:**
- No private keys stored anywhere (not in GitHub Secrets, not on developer laptops)
- Signature is bound to the GitHub Actions OIDC identity (`https://github.com/awdemos/nexus/.github/workflows/release.yml@refs/tags/vX.Y.Z`)
- Rekor entry provides public, auditable proof of signing

---

### Phase 3: Container Signing (Week 2)

If we distribute Nexus as a container (Flatpak, Docker, or devcontainer), sign the image.

```yaml
      - name: Sign container image
        run: |
          cosign sign --yes "ghcr.io/awdemos/nexus:${{ github.ref_name }}"
```

**Verification by users:**
```bash
cosign verify --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp https://github.com/awdemos/nexus/.github/workflows/release.yml \
  ghcr.io/awdemos/nexus:v1.0.0
```

---

### Phase 4: Commit Signing with Gitsign (Week 2)

Eliminate GPG key management for contributors. Use Sigstore to sign Git commits.

**Action:**
```bash
# Developer setup (one-time)
brew install gitsign  # or cargo install gitsign
git config --global commit.gpgsign true
git config --global gpg.x509.program gitsign
git config --global gpg.format x509
```

**Verify a commit:**
```bash
git verify-commit HEAD
```

**What this gives us:**
- Every commit in `nexus` is cryptographically attributable to a developer identity
- No GPG keyservers, no key expiration headaches, no subkey management
- Rekor logs every commit signature

---

### Phase 5: SBOM + Dependency Provenance (Week 3)

You can't secure what you can't inventory.

**Action:** Generate SPDX/CycloneDX SBOMs for every release.

```yaml
      - name: Generate SBOM
        uses: anchore/sbom-action@<sha>
        with:
          format: spdx-json
          output-file: nexus-${{ github.ref_name }}-sbom.spdx.json

      - name: Sign SBOM
        run: |
          cosign sign-blob nexus-*-sbom.spdx.json \
            --output-certificate sbom.crt \
            --output-signature sbom.sig
```

**Action:** Pin Rust toolchain and verify checksums.

```yaml
      - uses: dtolnay/rust-toolchain@<sha>
        with:
          toolchain: 1.80.0
          components: clippy, rustfmt
```

**Action:** Vendor critical dependencies or verify `Cargo.lock` checksums in CI.

```bash
cargo generate-lockfile
cargo tree --duplicate  # fail on duplicate versions of critical crates
cargo audit
```

---

### Phase 6: Verification & Policy (Week 4)

Security is only real if users actually verify.

**Action:** Publish a `VERIFY.md` with copy-paste commands.

```bash
# 1. Download artifact + provenance + signature
gh release download v1.0.0

# 2. Verify SLSA provenance
slsa-verifier verify-artifact \
  --provenance-path nexus-v1.0.0.intoto.jsonl \
  --source-uri github.com/awdemos/nexus \
  nexus-v1.0.0.dmg

# 3. Verify Sigstore signature
cosign verify-blob nexus-v1.0.0.dmg \
  --signature nexus-v1.0.0.dmg.sig \
  --certificate nexus-v1.0.0.dmg.crt \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity https://github.com/awdemos/nexus/.github/workflows/release.yml@refs/tags/v1.0.0
```

**Action:** Add a verification step to the auto-updater.

Before Nexus downloads an update, it should:
1. Fetch the provenance attestation from the GitHub release
2. Verify the SLSA provenance using `slsa-verifier`
3. Verify the Cosign signature using the Rekor transparency log
4. Only then replace the running binary

**Action:** Add a CI gate that fails if provenance or signing is missing.

```yaml
      - name: Enforce signed release
        run: |
          test -f nexus-${VERSION}.dmg.sig || exit 1
          test -f nexus-${VERSION}.intoto.jsonl || exit 1
```

---

## Artifact Table: What Gets Signed

| Artifact | Signed By | Stored In | Verified By |
|----------|-----------|-----------|-------------|
| Release binaries (`.dmg`, `.exe`, `.AppImage`) | Cosign (keyless) | GitHub Release + Rekor | User `cosign verify` |
| Container image | Cosign (keyless) | GHCR + Rekor | User `cosign verify` |
| SLSA Provenance | slsa-github-generator | GitHub Release + Rekor | User `slsa-verifier` |
| SBOM | Cosign (keyless) | GitHub Release + Rekor | Auditor / Enterprise |
| Git commits | Gitsign | Git history + Rekor | `git verify-commit` |
| Build attestations | GitHub OIDC | GitHub Actions logs | SLSA verifier |

## Rollout Checklist

- [ ] Phase 0: Branch protection, signed commits, pinned actions, Dependabot
- [ ] Phase 1: `slsa-github-generator` integrated into release workflow
- [ ] Phase 2: Cosign keyless signing for all release artifacts
- [ ] Phase 3: Container image signing (if applicable)
- [ ] Phase 4: Gitsign commit signing enforced via branch protection
- [ ] Phase 5: SBOM generation, `cargo audit`, dependency pinning
- [ ] Phase 6: `VERIFY.md` published, auto-updater verifies before install
- [ ] Phase 7 (Ongoing): Monitor Rekor for unauthorized signatures, rotate nothing (keyless)

## Cost

Sigstore's public infrastructure (Fulcio, Rekor) is **free** for open-source projects. There are no certificates to buy, no HSMs to provision, and no key rotation ceremonies.

The only cost is CI minutes for the additional jobs — negligible for a project that builds releases intermittently.

## Further Reading

- [SLSA Specification](https://slsa.dev/spec/v1.0/levels)
- [Sigstore Documentation](https://docs.sigstore.dev/)
- [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator)
- [Cosign Keyless Signing](https://docs.sigstore.dev/cosign/signing/overview/)
- [Gitsign](https://github.com/sigstore/gitsign)
- [Rekor Monitor](https://docs.sigstore.dev/rekor/monitor/)
