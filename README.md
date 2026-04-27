# secure-supply-chain

End-to-end secure software supply chain for containerized workloads. Every artifact produced by this system is cryptographically traceable from source commit to running pod, with no trust placed in mutable references, long-lived credentials, or implicit network access.

---

## Trust Model

The system enforces a single invariant: **a pod cannot run in production or staging unless its image was built by our CI pipeline, on the main branch, and passed every security gate.**

This is enforced at three independent layers:

| Layer | Mechanism | What it prevents |
|---|---|---|
| Registry | Cosign keyless signature | Running images built outside CI |
| Admission | Kyverno `verifyImages` | Bypassing the registry check via direct kubectl |
| Runtime | Falco syscall rules | Post-compromise lateral movement |

Defeating the invariant requires compromising all three layers simultaneously.

---

## Architecture

```
┌─ app-source repo ──────────────────────────────────────────────────────┐
│                                                                        │
│  PR push → [secret scan] → [IaC lint] → [build, no push]              │
│                                                                        │
│  main push → [secret scan]                                             │
│               → [build + push by digest]                               │
│               → [Trivy scan]  [Grype scan]  (two DBs, independent)    │
│               → [Syft SBOM → cosign attest]                            │
│               → [cosign sign (keyless OIDC)]                           │
│               → [SLSA L2 provenance] (separate job, isolated signer)  │
│               → [kustomize edit set image → gitops-config commit]      │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─ GHCR ─────────────────────────────────────────────────────────────────┐
│  image@sha256:...  +  cosign signature  +  SBOM attestation            │
│                   +  SLSA provenance attestation                       │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─ gitops-config repo ───────────────────────────────────────────────────┐
│  apps/my-app/overlays/production/kustomization.yaml                    │
│    images: digest: sha256:<immutable>   ← CI writes this              │
└────────────────────────────────────────────────────────────────────────┘
                                    │
                              ArgoCD detects diff
                                    │
                                    ▼
┌─ Kubernetes cluster ───────────────────────────────────────────────────┐
│                                                                        │
│  Kyverno admission webhook:                                            │
│    ✓ cosign signature valid (subject = ci.yml@main)                    │
│    ✓ SBOM attestation present                                          │
│    ✓ SLSA provenance present (signer = slsa-github-generator)          │
│    ✓ image reference contains @sha256: (no mutable tags)               │
│    ✓ no privileged, no hostNetwork, resource limits set                │
│                                                                        │
│  Pod scheduled → Falco monitors syscalls at runtime                    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Repository Layout

```
secure-supply-chain/
├── SECURITY.md
├── app-source/                              # Application + CI
│   ├── .github/
│   │   ├── CODEOWNERS                       # Security-gated review paths
│   │   └── workflows/
│   │       └── ci.yml                       # 4-job pipeline
│   ├── src/
│   │   ├── main.go
│   │   └── go.mod
│   ├── Dockerfile                           # Multi-stage, distroless, OCI annotations
│   └── .trivyignore                         # Accepted exceptions (reason + expiry required)
│
└── gitops-config/                           # Cluster desired state
    ├── apps/my-app/
    │   ├── base/
    │   │   ├── deployment.yaml              # topologySpread, graceful shutdown, seccomp
    │   │   ├── service.yaml
    │   │   ├── network-policy.yaml          # default-deny + explicit allow
    │   │   └── pdb.yaml                     # minAvailable: 1
    │   └── overlays/
    │       ├── staging/
    │       └── production/                  # digest pinned here by CI
    ├── policies/
    │   ├── verify-image-signature.yaml      # signature + SBOM + SLSA provenance
    │   ├── restrict-privileged.yaml         # PSS restricted + block :latest
    │   └── require-labels.yaml             # traceability enforcement
    ├── argocd/
    │   ├── application.yaml
    │   └── appproject.yaml                  # scoped RBAC, source/dest allowlist
    └── falco/
        └── falco-rules.yaml                 # deviation-from-known-good rules
```

---

## Design Decisions

### ADR-001: Keyless signing over key-based signing

**Decision:** Use Cosign keyless (GitHub OIDC) instead of a stored private key.

**Rationale:** A stored signing key is a secret that must be rotated, protected, and audited. If it leaks, every image ever signed with it is suspect. Keyless signing derives trust from the GitHub OIDC token, which is scoped to a specific workflow run and expires in minutes. The trust anchor is the workflow URL, not a key — changing the workflow path or branch invalidates the trust chain without any key rotation.

**Tradeoff:** Requires Sigstore's public Rekor transparency log. Acceptable for open-source workloads; for air-gapped environments, deploy a private Rekor instance.

---

### ADR-002: Two vulnerability scanners (Trivy + Grype)

**Decision:** Run both Trivy and Grype; fail on Trivy, audit on Grype.

**Rationale:** Trivy and Grype use different vulnerability databases (NVD + GitHub Advisory vs. Grype's own DB). A CVE present in one DB but not the other is a real-world occurrence. Running both reduces false-negative risk. Trivy is the gate because it has broader OS package coverage. Grype results go to the Security tab for review without blocking the pipeline — avoiding alert fatigue from DB lag.

---

### ADR-003: SLSA provenance as a separate job

**Decision:** Generate SLSA provenance in a dedicated job that runs after the build job, not within it.

**Rationale:** SLSA Level 2 requires that the provenance generator be isolated from the build environment. If provenance were generated in the same job, a compromised build step could forge the provenance. The separate job receives only the image digest as input — it cannot modify the build.

---

### ADR-004: Digest pinning in GitOps overlay

**Decision:** CI writes `image@sha256:...` to the production kustomization. Tags are never used in production.

**Rationale:** An image tag is a pointer that can be moved. `sha256:abc123` is a content address — it cannot be changed without changing the digest. Pinning by digest means the manifest in git is a complete, reproducible description of what runs in the cluster. A tag-based deployment is not reproducible.

---

### ADR-005: Kyverno system namespace excludes

**Decision:** All Kyverno policies explicitly exclude `kube-system`, `argocd`, `kyverno`, and other infrastructure namespaces.

**Rationale:** Without excludes, Kyverno's `Enforce` mode will reject pods from upstream components (CNI, CoreDNS, ArgoCD itself) that are not signed by our workflow. This breaks cluster bootstrap and upgrades. The excludes are explicit and reviewed via CODEOWNERS — they are not a security gap, they are a scoping decision.

---

## Operations

### Verify an image independently

```bash
# Verify signature — no trust in our infrastructure required
cosign verify \
  --certificate-identity "https://github.com/your-org/app-source/.github/workflows/ci.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/your-org/my-app@sha256:<digest>

# Inspect SBOM
cosign download attestation ghcr.io/your-org/my-app@sha256:<digest> \
  | jq -r 'select(.payload) | .payload' | base64 -d | jq .predicate.packages[].name

# Verify SLSA provenance
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-identity-regexp "https://github.com/slsa-framework/slsa-github-generator" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/your-org/my-app@sha256:<digest> | jq .
```

### Bootstrap a new cluster

```bash
# 1. Kyverno
kubectl apply -f https://github.com/kyverno/kyverno/releases/download/v1.12.0/install.yaml

# 2. ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.0/manifests/install.yaml

# 3. Falco (Helm)
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --set-file falco.rules_file=gitops-config/falco/falco-rules.yaml

# 4. Apply policies and ArgoCD resources
kubectl apply -f gitops-config/policies/
kubectl apply -f gitops-config/argocd/
```

### Required GitHub configuration

| Item | Value |
|---|---|
| Secret: `GITOPS_PAT` | Fine-grained PAT, `contents:write` on `gitops-config` only |
| Branch protection on `main` | Require PR, require status checks: `pr-gate` |
| CODEOWNERS enforcement | Require review from code owners |

---

## What this is not

This project does not include a service mesh (mTLS between pods), runtime image scanning (re-scanning running containers), or a SIEM integration. Those are the natural next layers and are documented as follow-on work in the issues.
