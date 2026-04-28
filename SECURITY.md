# Security Policy

## Supply Chain Integrity

Every image built from this repository is:
- Signed with Cosign (keyless, GitHub OIDC) — verifiable without trusting us
- Accompanied by an SPDX SBOM attestation stored in GHCR
- Accompanied by a SLSA Level 2 provenance attestation

To verify an image independently:

```bash
cosign verify \
  --certificate-identity "https://github.com/your-org/app-source/.github/workflows/ci.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/your-org/my-app@sha256:<digest>
```

A verification failure means the image was not built by our CI pipeline.
