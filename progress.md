# Project Learning Guide — Secure Software Supply Chain

---

## What Is This Project?

This project builds a **secure software supply chain** — a system that guarantees every piece of code you write goes through a strict, automated security process before it ever runs in production.

Think of it like a factory assembly line, but for software. At each station on the line, a quality check happens. If anything fails, the line stops. Nothing broken reaches the end. And every product that does reach the end has a certificate proving exactly how it was made.

The "product" here is a **Docker container image** — a packaged version of your application that runs on Kubernetes.

---

## Why Does This Project Exist?

### The problem it solves

In most companies, the path from "developer writes code" to "code runs in production" looks like this:

```
Developer writes code → someone manually reviews it → someone manually builds it
→ someone manually deploys it → it runs in production
```

This is dangerous for several reasons:

1. **A developer could accidentally commit a password or API key** into the code. It goes to production. Now your database credentials are in a git repo.

2. **The Docker image could contain known security vulnerabilities** (CVEs — Common Vulnerabilities and Exposures). Nobody checked. A hacker exploits one and gets into your server.

3. **Someone could build an image on their laptop** (not in CI), push it to the registry, and deploy it. That image was never scanned, never verified. You have no idea what's in it.

4. **An attacker who gains access to your registry** could replace a legitimate image with a malicious one. Your deployment system would pull and run it without knowing.

5. **You have no record of what software is inside your running containers.** When a new CVE is announced, you can't quickly answer "are we affected?"

This project solves all five of these problems with automation and cryptographic proof.

### The real-world context

In 2020, attackers compromised SolarWinds' build pipeline and inserted malicious code into a software update. That update was then distributed to 18,000 customers — including US government agencies. The attack was undetected for months.

This is called a **supply chain attack** — instead of attacking your application directly, the attacker attacks the process that builds and delivers your application.

This project implements defenses against exactly this class of attack.

---

## What Real Problem Does It Solve?

| Problem | How This Project Solves It |
|---|---|
| Secret accidentally committed to git | Gitleaks scans every PR and blocks the merge |
| Vulnerable dependencies in the image | Trivy + Grype scan the image and fail the pipeline |
| No record of what's inside the image | Syft generates an SBOM (full ingredient list) |
| Image tampered with after build | Cosign signs the image; Kyverno verifies the signature before running |
| Image built outside CI (unverified) | Kyverno blocks any unsigned image from running |
| Attacker replaces image in registry | Signature verification catches it — the new image won't have a valid signature |
| Pods running as root, accessing everything | SecurityContext + NetworkPolicy + Kyverno policies enforce least privilege |
| No audit trail | Every image has a cryptographic signature tied to the exact git commit and workflow run |

---

## What Do You Get When This Is Implemented?

1. **You can prove, cryptographically, that any running container was built by your CI pipeline** — not by a developer's laptop, not by an attacker.

2. **You have a complete ingredient list** (SBOM) for every image. When a new CVE is announced, you can search your SBOMs and immediately know which services are affected.

3. **No secret can reach production** without being caught by automated scanning.

4. **No vulnerable image can be deployed** — the pipeline blocks it.

5. **Even if an attacker gets into your Kubernetes cluster**, they cannot run arbitrary containers. Kyverno will reject anything that isn't signed by your pipeline.

6. **Your cluster is auditable** — every deployment is traceable to a specific git commit, a specific workflow run, and a specific point in time.

---

## Technologies Used — Plain English Explanations

### Docker & Container Images
A Docker image is a packaged, self-contained version of your application. It includes your code, its dependencies, and a minimal operating system. When it runs, it becomes a **container** — an isolated process on a server. Kubernetes manages many containers across many servers.

---

### GitHub Actions (CI/CD)
CI/CD stands for Continuous Integration / Continuous Deployment. GitHub Actions is GitHub's built-in automation system. You write a YAML file describing a series of steps, and GitHub runs those steps automatically every time you push code. In this project, those steps are: build the image, scan it, sign it, and deploy it.

---

### GHCR — GitHub Container Registry
GHCR is GitHub's built-in Docker image registry. Think of it like a warehouse where your built images are stored. When Kubernetes needs to run your application, it pulls the image from GHCR. It's the same concept as Docker Hub, but integrated with GitHub so permissions and authentication are handled automatically.

---

### Trivy
A vulnerability scanner made by Aqua Security. After your image is built, Trivy inspects every package and library inside it and compares them against a database of known vulnerabilities (CVEs). If it finds a CRITICAL or HIGH severity vulnerability that has a fix available, it fails the pipeline. You must update the dependency before the image can be deployed.

---

### Grype
A second vulnerability scanner made by Anchore. It uses a different vulnerability database than Trivy. We run both because no single database is complete — a CVE might be in Grype's database before it appears in Trivy's. Grype runs in audit mode (doesn't block the pipeline) but its results appear in the GitHub Security tab for review.

---

### SBOM — Software Bill of Materials
An SBOM is a complete list of every software component inside your container image — every library, every package, every dependency, and their exact versions. It's the software equivalent of a food nutrition label.

**Why it matters:** When a new vulnerability is announced (e.g., "Log4Shell affects Log4j versions X through Y"), you can search all your SBOMs and immediately know which of your services use that library. Without an SBOM, you'd have to manually inspect every service.

In this project, the SBOM is generated by **Syft** in SPDX-JSON format (SPDX is an industry standard format for SBOMs, maintained by the Linux Foundation). The SBOM is then attached to the image in the registry as an **attestation** — a cryptographically signed document that travels with the image.

---

### Cosign
A tool from the Sigstore project for signing container images. After your image is built and scanned, Cosign attaches a cryptographic signature to it. This signature proves:
- Who built it (which GitHub Actions workflow)
- When it was built
- What exact image it applies to (by digest, not tag)

**Keyless signing** means no private key is stored anywhere. Instead, Cosign uses a temporary certificate issued by GitHub's identity system (OIDC) that expires in minutes. The signature is recorded in a public transparency log called **Rekor** — similar to how certificate authorities work for HTTPS.

---

### Kyverno
Kyverno is a **policy engine** for Kubernetes. It sits between you and the Kubernetes API server as an **admission webhook** — every time something tries to create or update a resource (like a Pod), Kyverno intercepts the request, checks it against your policies, and either allows or rejects it.

In this project, Kyverno enforces three things:
1. **Image signature verification** — if the image isn't signed by our CI workflow, the pod is rejected before it ever starts
2. **Security hardening** — no privileged containers, no root user, resource limits required
3. **Traceability** — deployments must have labels linking them to a git commit

The key insight: even if someone bypasses your CI pipeline and pushes an image directly to the registry, Kyverno will reject it when they try to run it in the cluster.

---

### SLSA — Supply-chain Levels for Software Artifacts
SLSA (pronounced "salsa") is a security framework from Google that defines levels of supply chain integrity. Level 2 (what this project implements) requires:
- The build runs on a hosted, ephemeral build platform (GitHub Actions ✓)
- The build is defined in version-controlled configuration (ci.yml ✓)
- Provenance (a document describing how the artifact was built) is generated and signed by a system separate from the build itself ✓

The provenance document answers: "What source code was used? What build steps ran? What was the output?" It's generated in a **separate job** from the build — this isolation is the key SLSA requirement.

---

### ArgoCD (GitOps)
ArgoCD is a **GitOps** tool. GitOps means your git repository is the single source of truth for what should be running in your cluster. ArgoCD watches a git repo (`gitops-config`) and continuously ensures the cluster matches what's in that repo.

When CI finishes building and signing an image, it updates a file in `gitops-config` with the new image digest. ArgoCD detects that commit and automatically syncs the cluster to match. You never manually run `kubectl apply` in production.

**Why this matters for security:** The deployment path is auditable. Every change to production is a git commit with an author, a timestamp, and a diff. You can see exactly what changed and when.

---

### Kustomize
Kustomize is a tool for managing Kubernetes YAML files across multiple environments (staging, production) without duplicating them. You write a `base` set of manifests once, then write `overlays` that describe only the differences per environment (different replica count, different image digest). Kustomize merges them at deploy time.

---

### Falco
Falco is a **runtime security** tool. While Kyverno operates at admission time (before a pod starts), Falco operates at runtime (while the pod is running). It hooks into the Linux kernel's system call interface and watches what every container is actually doing — what files it opens, what network connections it makes, what processes it spawns.

If a container does something it shouldn't (opens a shell, makes an unexpected network connection, tries to escalate privileges), Falco fires an alert. This is your last line of defense — if an attacker somehow gets code running inside a container, Falco will detect the suspicious behavior.

---

### NetworkPolicy
A Kubernetes resource that acts as a firewall for pods. By default, all pods in a Kubernetes cluster can talk to all other pods. NetworkPolicy lets you restrict this. In this project, the `my-app` pods:
- Can only receive traffic from the ingress controller (the thing that handles external HTTP requests)
- Can only make outbound connections for DNS (port 53)
- Cannot reach any other service, database, or the internet

This is **zero-trust networking** — deny everything, allow only what's explicitly needed.

---

### PodDisruptionBudget (PDB)
A Kubernetes resource that protects your application during maintenance. When a cluster node needs to be drained (for upgrades, for example), Kubernetes will evict pods from it. Without a PDB, it might evict all your pods at once, causing downtime. The PDB says "always keep at least 1 pod running" — Kubernetes will wait for a new pod to start before evicting the old one.

---

### Distroless Images
A "distroless" Docker base image contains only your application and its runtime dependencies — no shell (`bash`/`sh`), no package manager (`apt`/`apk`), no debugging tools. This matters for security because:
- An attacker who gets code execution inside your container cannot open a shell
- There's no package manager to install additional tools
- The attack surface is dramatically smaller
- Vulnerability scanners find far fewer CVEs because there are far fewer packages

The image used here is `gcr.io/distroless/static-debian12:nonroot` — it runs as a non-root user (UID 65532) by default.

---

## End-to-End Flow — From Code to Running Pod

Here is the complete journey of a single code change, step by step.

### Step 1 — Developer opens a Pull Request

A developer pushes a branch and opens a PR against `main`. GitHub Actions immediately triggers the `pr-gate` job.

**What happens:**
- **Gitleaks** scans the diff for secrets (API keys, passwords, tokens). If it finds one, the PR is blocked. The developer must remove the secret and force-push.
- **Kubeconform** validates all YAML files in `gitops-config/` against the Kubernetes schema. A typo in a manifest is caught here, not in the cluster.
- **Docker build** runs without pushing. This confirms the Dockerfile is valid and the image compiles. Nothing is written to the registry.

**Output:** A green or red check on the PR. No artifacts are produced.

---

### Step 2 — PR is approved and merged to main

The `build` job triggers.

**What happens:**
1. Gitleaks runs again on the full branch (not just the diff).
2. Docker builds the image and pushes it to GHCR. The output is an **immutable digest** — a `sha256:abc123...` fingerprint of the exact image contents. This digest is used for every subsequent step. The tag (`:abc123commit`) is written but never used again.
3. **Trivy** scans the image by digest. If it finds a CRITICAL or HIGH CVE that has a fix available, the pipeline fails here. The image exists in the registry but nothing else happens — it will never be deployed.
4. **Grype** scans the same image. Results go to the GitHub Security tab. The pipeline continues regardless.
5. **Syft** generates an SBOM — a JSON file listing every package inside the image.
6. **Cosign** signs the image. The signature is stored in GHCR alongside the image (not as a separate file — as an OCI artifact attached to the same digest).
7. **Cosign** attaches the SBOM as a signed attestation. Now the SBOM is cryptographically bound to the image — you can't swap the SBOM without invalidating the signature.

**Output:** An image in GHCR with three things attached to its digest: the image itself, a Cosign signature, and a signed SBOM attestation.

---

### Step 3 — SLSA Provenance is generated

The `provenance` job runs after `build` completes. It runs in a **completely separate job** — this isolation is required by the SLSA specification.

**What happens:**
- The `slsa-github-generator` workflow receives only the image digest as input. It cannot modify the build.
- It generates a provenance document: "This image was built from commit X, by workflow Y, at time Z, producing digest sha256:abc123."
- It signs this document with its own Cosign keyless signature and attaches it to the image in GHCR as a second attestation.

**Output:** A third artifact attached to the image digest — a signed SLSA provenance attestation.

---

### Step 4 — GitOps promotion

The `promote` job runs after both `build` and `provenance` succeed.

**What happens:**
- It checks out the `gitops-config` repository (a separate repo).
- It runs `kustomize edit set image my-app=ghcr.io/your-org/my-app@sha256:abc123...` — this writes the immutable digest into `apps/my-app/overlays/production/kustomization.yaml`.
- It commits and pushes: `"ci: promote abc123 to production"`.

**Output:** A new commit in `gitops-config` with the updated image digest.

---

### Step 5 — ArgoCD detects the change

ArgoCD continuously polls `gitops-config`. It detects the new commit.

**What happens:**
- ArgoCD compares the cluster's current state against the new desired state in git.
- It sees the image digest has changed.
- It prepares to apply the updated Deployment to the `production` namespace.
- Before applying, it sends the Pod spec to the Kubernetes API server.

---

### Step 6 — Kyverno admission gate

This is the most important step. Before any pod is scheduled, Kyverno intercepts the request.

**What happens (three checks, all must pass):**

**Check 1 — Image signature:**
Kyverno calls the Cosign verification logic. It checks: "Does this image have a valid signature from `https://github.com/your-org/app-source/.github/workflows/ci.yml@refs/heads/main`?" If the image was built outside CI, or on a different branch, or by a different workflow — no valid signature exists. Kyverno rejects the pod. It never starts.

**Check 2 — SBOM attestation:**
Kyverno checks: "Does this image have a signed SPDX SBOM attestation?" If the SBOM step was skipped or the attestation is missing — rejected.

**Check 3 — SLSA provenance:**
Kyverno checks: "Does this image have a SLSA provenance attestation signed by the `slsa-github-generator`?" If not — rejected.

**Additional checks (restrict-privileged policy):**
- Is the container trying to run as root? → Rejected
- Is `allowPrivilegeEscalation` set to true? → Rejected
- Is the filesystem writable? → Rejected
- Are CPU/memory limits missing? → Rejected
- Is the image referenced by a mutable tag instead of a digest? → Rejected

If all checks pass, Kyverno approves the pod.

---

### Step 7 — Pod runs, Falco watches

The pod is scheduled on a node and starts running.

**What happens:**
- Falco is running on every node, watching system calls at the kernel level.
- If the pod tries to open a network connection to anything other than DNS → Falco fires a WARNING alert.
- If a shell process spawns inside the container (impossible with distroless, but Falco catches it anyway) → Falco fires a CRITICAL alert.
- If the pod tries to write to the filesystem → Falco fires a WARNING alert.
- If any process tries to call `setuid` or `setgid` (privilege escalation) → Falco fires a CRITICAL alert.

Alerts go to wherever Falco is configured to send them — typically a SIEM, Slack, or PagerDuty.

---

### The complete picture

```
Developer pushes code
        │
        ▼
[PR Gate]
  Gitleaks → block if secret found
  Kubeconform → block if YAML invalid
  Docker build (no push) → block if Dockerfile broken
        │
        ▼ (PR merged to main)
[Build Job]
  Gitleaks (full branch)
  Docker build + push → image@sha256:digest
  Trivy scan → BLOCK if fixable CRITICAL/HIGH CVE
  Grype scan → audit only
  Syft → sbom.spdx.json
  Cosign sign → signature stored in GHCR
  Cosign attest → SBOM stored in GHCR
        │
        ▼
[Provenance Job] (isolated)
  SLSA provenance generated + signed → stored in GHCR
        │
        ▼
[Promote Job]
  kustomize edit set image → digest written to gitops-config
  git commit + push
        │
        ▼
[ArgoCD]
  Detects commit → prepares sync
        │
        ▼
[Kyverno Admission Gate]
  ✓ Cosign signature valid?
  ✓ SBOM attestation present?
  ✓ SLSA provenance present?
  ✓ No privileged settings?
  ✓ Image pinned by digest?
  → ALL PASS: pod scheduled
  → ANY FAIL: pod rejected, error returned to ArgoCD
        │
        ▼
[Pod Running]
  Falco watches syscalls
  → Alert on any deviation from expected behavior
```

---

## Directory Structure — Every File Explained

```
secure-supply-chain/
├── README.md                          ← Engineering documentation (ADRs)
├── SECURITY.md                        ← How to report vulnerabilities
│
├── app-source/                        ← The application + its CI pipeline
│   ├── .github/
│   │   ├── CODEOWNERS                 ← Who must review which files
│   │   └── workflows/
│   │       └── ci.yml                 ← The entire CI/CD pipeline
│   ├── src/
│   │   ├── main.go                    ← The Go application
│   │   └── go.mod                     ← Go dependency manifest
│   ├── Dockerfile                     ← How to build the container image
│   └── .trivyignore                   ← CVE exceptions (with required justification)
│
└── gitops-config/                     ← What should run in the cluster
    ├── apps/my-app/
    │   ├── base/                      ← Shared across all environments
    │   │   ├── deployment.yaml        ← How the app runs (replicas, security, probes)
    │   │   ├── service.yaml           ← How the app is reached internally
    │   │   ├── network-policy.yaml    ← What network traffic is allowed
    │   │   ├── pdb.yaml               ← Minimum pods during maintenance
    │   │   └── kustomization.yaml     ← Lists all base resources
    │   └── overlays/
    │       ├── staging/               ← Staging-specific overrides (1 replica)
    │       │   └── kustomization.yaml
    │       └── production/            ← Production-specific overrides (2 replicas, pinned digest)
    │           └── kustomization.yaml ← CI writes the image digest here
    ├── policies/
    │   ├── verify-image-signature.yaml ← Kyverno: check signature + SBOM + SLSA
    │   ├── restrict-privileged.yaml    ← Kyverno: no root, no latest tag, resource limits
    │   └── require-labels.yaml        ← Kyverno: traceability labels required
    ├── argocd/
    │   ├── application.yaml           ← Tells ArgoCD what to deploy and where
    │   └── appproject.yaml            ← Scopes ArgoCD's permissions (RBAC)
    └── falco/
        └── falco-rules.yaml           ← Runtime threat detection rules
```

### File-by-file breakdown

**`app-source/.github/CODEOWNERS`**
Defines who must approve changes to sensitive files. Changes to `ci.yml` require the security team's approval. This prevents a developer from weakening the pipeline (removing a scan step, lowering severity thresholds) without a security review. GitHub enforces this — a PR cannot be merged without the required approvers.

**`app-source/.github/workflows/ci.yml`**
The heart of the project. Four jobs:
- `pr-gate` — runs on PRs, fast feedback, no registry writes
- `build` — runs on main, full security pipeline
- `provenance` — isolated SLSA provenance generation
- `promote` — writes the new digest to gitops-config

**`app-source/Dockerfile`**
Two-stage build. Stage 1 uses `golang:alpine` to compile the binary. Stage 2 copies only the compiled binary into `distroless/static:nonroot`. The final image has no compiler, no shell, no package manager. The `LABEL` instructions bake the git commit SHA and build date into the image metadata.

**`app-source/.trivyignore`**
A list of CVEs that are accepted as known risk. Every entry requires a reason, an expiry date, and an owner. An expired entry causes the pipeline to fail — this forces regular review of accepted risks rather than letting them accumulate silently.

**`gitops-config/apps/my-app/base/deployment.yaml`**
The most detailed file. Key settings:
- `maxUnavailable: 0` — zero-downtime rolling updates
- `topologySpreadConstraints` — pods spread across zones AND nodes (not both on the same server)
- `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `capabilities: drop: [ALL]` — container cannot do anything it doesn't need to
- `seccompProfile: RuntimeDefault` — kernel-level syscall filtering
- `preStop: sleep 5` — graceful shutdown, lets in-flight requests finish

**`gitops-config/apps/my-app/base/network-policy.yaml`**
Three policies:
1. Default deny all ingress and egress for `my-app` pods
2. Allow ingress from `ingress-nginx` namespace on port 8080 only
3. Allow egress to DNS (port 53) only

The pod cannot reach your database, other services, or the internet unless you explicitly add a rule.

**`gitops-config/apps/my-app/base/pdb.yaml`**
`minAvailable: 1` — Kubernetes will never evict the last running pod during node maintenance. Prevents accidental downtime during cluster upgrades.

**`gitops-config/apps/my-app/overlays/production/kustomization.yaml`**
The CI pipeline writes to this file. It contains the image digest that ArgoCD will deploy. This is the link between CI and GitOps — CI produces an image, writes its digest here, ArgoCD reads it and deploys it.

**`gitops-config/policies/verify-image-signature.yaml`**
Two Kyverno rules:
1. Verify the Cosign signature (subject must be the exact CI workflow URL) + require SBOM attestation
2. Verify the SLSA provenance attestation (signer must be `slsa-github-generator`)

Both rules exclude system namespaces — without this, Kyverno would block ArgoCD and Kubernetes system pods from starting.

**`gitops-config/policies/restrict-privileged.yaml`**
Three Kyverno rules:
1. Block privileged containers, host namespaces, root user, writable filesystem
2. Require CPU and memory limits on every container
3. Block `:latest` tag and any image not referenced by `@sha256:` digest

**`gitops-config/argocd/appproject.yaml`**
Scopes ArgoCD's permissions. Without this, ArgoCD's `default` project allows deploying anything from any repo to any namespace. This `AppProject` restricts it to: only from `gitops-config`, only to `production` and `staging`, only specific resource types. It cannot deploy a `ClusterRole` or write to `kube-system`.

**`gitops-config/falco/falco-rules.yaml`**
Four runtime detection rules, all based on the principle "alert on deviation from known-good behavior":
1. Unexpected outbound network connection
2. Shell spawned (impossible in distroless — if this fires, something is seriously wrong)
3. Unexpected file write
4. Privilege escalation via setuid/setgid

---

## Coding Practices Used in This Project

### Principle of Least Privilege
Every component only has the permissions it absolutely needs. The CI pipeline's `pr-gate` job has `contents: read` only — it cannot write to the registry. The `build` job has `packages: write` — it can push images. The `promote` job has `contents: write` — it can push to git. Nothing has more than it needs.

This same principle applies to the Kubernetes pods: no root, no writable filesystem, no network access beyond DNS, no Linux capabilities.

### Defense in Depth
No single security control is relied upon alone. The same guarantee (only CI-built images run in production) is enforced at three independent layers:
- The registry (only CI can push signed images)
- Kyverno (verifies the signature at admission time)
- Falco (detects suspicious behavior at runtime)

An attacker must defeat all three simultaneously.

### Immutability
Tags like `:latest` or `:main` are mutable — they can be moved to point at a different image. A digest like `sha256:abc123...` is immutable — it is a cryptographic hash of the image contents. If the contents change, the digest changes. This project uses digests everywhere after the build step. The tag is written for human readability but never used for deployment.

### Separation of Concerns
The two repos (`app-source` and `gitops-config`) are intentionally separate:
- `app-source` is owned by developers. It contains code and CI.
- `gitops-config` is owned by the platform/ops team. It contains what runs in the cluster.

A developer cannot directly change what runs in production — they can only change the application code. The CI pipeline, after passing all security gates, is the only thing that updates `gitops-config`.

### Fail Fast, Fail Loud
Every security check that fails stops the pipeline immediately. Nothing proceeds past a failed scan. The error is surfaced in the GitHub UI, in the Security tab (SARIF format), and in the PR checks. There is no silent failure.

### Documented Exceptions
The `.trivyignore` file requires every CVE exception to have a reason, an expiry date, and an owner. An undocumented exception is not allowed. An expired exception breaks the pipeline. This forces accountability — you cannot silently accept risk indefinitely.

### GitOps — Git as the Source of Truth
The cluster's desired state is always a git commit. This means:
- Every change to production is auditable (who, what, when)
- Rolling back is a `git revert`
- You can see the exact state of production at any point in history
- No manual `kubectl apply` in production — humans don't touch the cluster directly

---

## Things You Should Know That You Didn't Ask

### What is a CVE?
CVE stands for Common Vulnerabilities and Exposures. It's a standardized identifier for a known security vulnerability. For example, `CVE-2021-44228` is the identifier for Log4Shell. When Trivy scans your image, it checks every package version against a database of CVEs and tells you which ones are present.

### What is OIDC?
OIDC (OpenID Connect) is an identity protocol. GitHub Actions uses it to issue short-lived identity tokens to workflow runs. When Cosign signs an image "keylessly," it's using this token to prove "I am GitHub Actions workflow X, running on repo Y, on branch Z." The token expires in minutes — there's no long-lived secret to steal.

### What is a Digest vs a Tag?
- **Tag**: A human-readable pointer like `:latest` or `:v1.2.3`. It can be moved. `docker push myimage:latest` overwrites the previous `:latest`.
- **Digest**: A SHA-256 hash of the image contents like `sha256:abc123...`. It cannot be changed. If you change one byte of the image, the digest changes. This is why production deployments always use digests.

### What is an Attestation?
An attestation is a signed statement about an artifact. In this project:
- The SBOM attestation says: "This image contains these packages" — signed by our CI workflow
- The SLSA provenance attestation says: "This image was built from this commit, by this workflow" — signed by the SLSA generator

Attestations are stored in the registry alongside the image. They travel with the image. Kyverno can verify them at admission time.

### Why Two Separate Repos?
In real companies, the application code repo and the infrastructure/deployment repo are always separate. Reasons:
1. Different access controls — not every developer should be able to change production deployment config
2. Different review processes — infrastructure changes need ops review, code changes need dev review
3. Audit trail — you want a clean history of "what changed in production" separate from "what code changed"
4. GitOps tools like ArgoCD watch a specific repo — keeping it clean makes the history meaningful

### What Does an SBOM Actually Look Like?
An SBOM in SPDX-JSON format (what Syft generates) is a JSON file listing every package inside the image with its version, license, and CPE identifier:

```json
{
  "spdxVersion": "SPDX-2.3",
  "name": "ghcr.io/your-org/my-app@sha256:abc123",
  "packages": [
    {
      "name": "stdlib",
      "version": "go1.22.2",
      "licenseConcluded": "BSD-3-Clause",
      "externalRefs": [
        {
          "referenceCategory": "SECURITY",
          "referenceType": "cpe23Type",
          "referenceLocator": "cpe:2.3:a:golang:go:1.22.2:*:*:*:*:*:*:*"
        }
      ]
    }
  ]
}
```

The CPE identifier is what vulnerability scanners use to match packages against CVE databases. For a distroless Go image, the SBOM is typically 10–20 packages. A full Ubuntu-based image would have 200+.

### What Is the SBOM Actually Used For?
Three concrete uses:

1. **CVE impact analysis** — when a new vulnerability is announced, query all SBOMs to instantly know which services are affected. Without SBOMs you'd have to manually inspect every running container.
2. **Kyverno admission enforcement** — Kyverno verifies the SBOM attestation is cryptographically signed by CI. An image built on a developer's laptop has no signed SBOM and gets rejected. The SBOM becomes proof of CI provenance, not just a document.
3. **License compliance** — every package has a `licenseConcluded` field. Legal teams run automated checks for GPL in proprietary products, required in regulated industries.

### Can an SBOM Be Generated for Any Docker Image?
Yes, including images you didn't build yourself:

```bash
syft ubuntu:22.04        # public image
syft nginx:latest        # any registry image
syft my-local-image:dev  # local untagged image
```

Syft unpacks the image layers and scans for OS package databases (`/var/lib/dpkg/status`, `/var/lib/rpm`) and language manifests (`go.sum`, `package-lock.json`, etc.). Completeness depends on how much metadata survived into the final image layers.

### Is an SBOM the Biometric of a Docker Image?
Close but not quite. The roles are distinct:

- **Digest (`sha256:abc123`)** is the biometric — a unique cryptographic fingerprint. Change one byte, the digest changes. Two different images can never share a digest.
- **SBOM** is more like a medical record — it lists what's inside. Two completely different images built from the same packages would have identical SBOMs but different digests.

They work together: the digest identifies the exact image, the SBOM describes its contents, and the Cosign attestation cryptographically binds the SBOM to that digest so neither can be swapped without detection.

### What Is a Mutable vs Immutable Image Reference?
The image bits are always immutable. What's mutable or immutable is the *reference* — how you point to it:

```
MUTABLE (tag):
  nginx:latest ──► image A  (today)
  nginx:latest ──► image B  (tomorrow, after a new push)
  same name, different image — non-deterministic

IMMUTABLE (digest):
  nginx@sha256:abc123 ──► image A  (always, forever)
  cannot be reassigned
```

Can two different images share the same digest? No. SHA-256 is a cryptographic hash — a collision has never been achieved and is computationally infeasible.

### What Do Cosign Sign and Cosign Attest Actually Do Differently?
They protect different things:

- **`cosign sign` (step 6)** — signs the image itself. Proves: *"this image was built by our CI workflow."*
- **`cosign attest` (step 7)** — bundles the SBOM document + a signature binding it to the image digest. Proves: *"this SBOM was produced by our CI workflow for this exact image."*

Without step 7, someone could attach a fake SBOM that hides a vulnerable package. Kyverno checks that the SBOM attestation is signed by `ci.yml@main` — a fake SBOM won't have that signature and the pod is rejected.

### How Does the Provenance Job Produce Provenance From Just a Digest?
The digest is only one input. The `slsa-github-generator` also reads the GitHub Actions runtime context — which is injected by GitHub's infrastructure, not your workflow code, and cannot be forged by a compromised build step:

```
digest (from build job output)  ──┐
github.sha       (commit)         │
github.repository                 ├──► provenance document ──► signed
github.workflow  (ci.yml)         │
github.ref       (refs/heads/main)│
timestamp                         │
                                ──┘
```

The digest anchors the provenance to the specific image. The GitHub context anchors it to the specific commit and workflow. A compromised build step can manipulate its own outputs but cannot change what `github.sha` or `github.repository` says.

### What Is the Promote Job Actually Doing?
It's the bridge between CI and the cluster. It doesn't build, scan, or sign — it only writes the new image digest into the gitops-config repo:

```
Before:  newTag: old-sha
After:   newDigest: sha256:abc123...
```

ArgoCD detects that git commit and syncs the cluster. The promote job never touches the cluster directly — it just writes to git. This is the core idea of GitOps: git is the source of truth, not manual `kubectl apply`.

The `[skip ci]` in the commit message prevents an infinite loop — without it, pushing to gitops-config would trigger another pipeline run.

### What Is the GHCR Box in the Flow Diagram?
It is not a step. It is a state diagram showing what GHCR contains after all jobs finish:

```
build job      ──► GHCR gets: image + signature + SBOM attestation
provenance job ──► GHCR gets: + SLSA provenance attestation
promote job    ──► gitops-config gets: digest pinned  (doesn't touch GHCR)
```

The box summarizes the final state of the registry that ArgoCD and Kyverno will read from when verifying the image. It should be read as a side note next to the build + provenance jobs, not as a downstream step.

### What Would Break Without Each Component?

| Remove this | What breaks |
|---|---|
| Gitleaks | Secrets can reach production undetected |
| Trivy | Vulnerable images deploy to production |
| Cosign signing | Anyone can build and deploy an image |
| Kyverno signature check | The signing step is theater — nothing enforces it |
| SLSA provenance | You can't prove the image came from a specific build |
| NetworkPolicy | A compromised pod can reach your database, other services, the internet |
| Falco | Post-compromise activity goes undetected |
| PDB | Cluster upgrades can cause downtime |
| Digest pinning | Deployments are non-deterministic — same manifest, different image |
| AppProject | ArgoCD can deploy anything to anywhere |

### What's the Next Level After This?

This project covers the supply chain and admission control. The natural next steps in a real company would be:

1. **Service mesh (Istio/Linkerd)** — mTLS between every pod, so even internal traffic is encrypted and authenticated
2. **Runtime image re-scanning** — re-scan running containers periodically, not just at build time. A CVE published after your image was built won't be caught until the next build.
3. **SIEM integration** — send Falco alerts to a centralized security information and event management system (Splunk, Elastic, etc.)
4. **Dependency-Track** — a platform that ingests your SBOMs and continuously monitors them against new CVEs. You get alerted when a new CVE affects a package in any of your running services.
5. **OPA/Gatekeeper** — an alternative to Kyverno with more expressive policy language (Rego) for complex policy requirements

---

## Summary

This project answers one question: **"How do you know that what's running in your cluster is exactly what your developers wrote, built securely, and hasn't been tampered with?"**

The answer is a chain of cryptographic proof:
- The code was scanned for secrets before it was merged (Gitleaks)
- The image was scanned for vulnerabilities before it was signed (Trivy + Grype)
- The image has a complete ingredient list (Syft SBOM)
- The image is cryptographically signed by the CI pipeline (Cosign)
- The build process itself is documented and signed (SLSA provenance)
- Nothing runs in the cluster without that proof (Kyverno)
- Anything that deviates from expected behavior at runtime is detected (Falco)

Every step produces evidence. Every piece of evidence is cryptographically verifiable. The chain cannot be broken without detection.

---

## Flow Diagrams, Infrastructure, and DevOps vs DevSecOps Comparison

---

### DevOps Pipeline (Without SecOps Practices)

A standard DevOps pipeline focuses on speed: code ships fast, tests run, image is built and deployed. No security gates exist.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DEVOPS PIPELINE (no security)                    │
└─────────────────────────────────────────────────────────────────────┘

Developer
    │
    │  git push
    ▼
GitHub Repository
    │
    │  triggers CI
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GitHub Actions                                                     │
│                                                                     │
│  1. Checkout code                                                   │
│  2. Run unit tests                                                  │
│  3. docker build                                                    │
│  4. docker push → registry (tagged :latest or :branch)             │
└─────────────────────────────────────────────────────────────────────┘
    │
    │  image pushed (mutable tag, no scan, no signature)
    ▼
Container Registry (Docker Hub / GHCR)
    │
    │  manual kubectl apply  OR  CD tool detects new tag
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                                 │
│                                                                     │
│  Pod starts immediately                                             │
│  No admission checks                                                │
│  No runtime monitoring                                              │
│  Any image from any source can run                                  │
└─────────────────────────────────────────────────────────────────────┘

WHAT CAN GO WRONG:
  ✗ Secret committed to git → ships to production undetected
  ✗ Vulnerable dependency in image → never scanned
  ✗ Developer builds image on laptop → pushed directly, no CI
  ✗ Attacker replaces image in registry → cluster pulls it silently
  ✗ :latest tag → different image on every pull, non-reproducible
  ✗ No audit trail → can't answer "what's running and how was it built?"
  ✗ Pod runs as root → full node compromise if container is breached
  ✗ No network isolation → compromised pod reaches database freely
```

---

### DevSecOps Pipeline (This Project)

Security is embedded at every stage. Each gate is automated and blocking. The chain is cryptographically verifiable end-to-end.

```
┌─────────────────────────────────────────────────────────────────────┐
│                  DEVSECOPS PIPELINE (this project)                  │
└─────────────────────────────────────────────────────────────────────┘

Developer
    │
    │  git push (feature branch)
    ▼
GitHub Repository
    │
    │  Pull Request opened
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PR GATE (GitHub Actions — pr-gate job)                             │
│                                                                     │
│  [1] Gitleaks ──────────────── secret in diff? → BLOCK PR          │
│  [2] Kubeconform ───────────── invalid YAML?   → BLOCK PR          │
│  [3] docker build (no push) ── Dockerfile bad? → BLOCK PR          │
│                                                                     │
│  permissions: contents:read only — cannot touch registry           │
└─────────────────────────────────────────────────────────────────────┘
    │
    │  PR approved (CODEOWNERS enforced for ci.yml, policies/)
    │  merged to main
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  BUILD JOB (GitHub Actions — build job)                             │
│                                                                     │
│  [1] Gitleaks (full branch scan)                                    │
│  [2] docker build + push → GHCR                                     │
│       output: image@sha256:<digest>  ← immutable from this point   │
│  [3] Trivy scan ────────────── CRITICAL/HIGH CVE + fix? → FAIL     │
│  [4] Grype scan ────────────── audit only → GitHub Security tab    │
│  [5] Syft ──────────────────── generate sbom.spdx.json             │
│  [6] Cosign sign ───────────── keyless OIDC signature → GHCR       │
│  [7] Cosign attest ─────────── SBOM attestation → GHCR             │
│                                                                     │
│  permissions: packages:write, id-token:write                       │
└─────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PROVENANCE JOB (isolated — slsa-github-generator)                  │
│                                                                     │
│  Receives only: image digest (cannot modify build)                  │
│  Generates SLSA L2 provenance document                              │
│  Signs + attaches provenance attestation → GHCR                    │
│                                                                     │
│  Isolation is a SLSA L2 requirement                                 │
└─────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PROMOTE JOB (GitHub Actions — promote job)                         │
│                                                                     │
│  Checks out gitops-config repo (separate repo)                      │
│  kustomize edit set image my-app=ghcr.io/org/my-app@sha256:<digest> │
│  git commit + push → "ci: promote <sha> [skip ci]"                 │
│                                                                     │
│  permissions: contents:write on gitops-config only (fine-grained)  │
└─────────────────────────────────────────────────────────────────────┘
    │
    │  new commit in gitops-config
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GHCR (GitHub Container Registry)                                   │
│                                                                     │
│  image@sha256:<digest>                                              │
│    ├── cosign signature                                             │
│    ├── SBOM attestation (SPDX-JSON)                                 │
│    └── SLSA provenance attestation                                  │
└─────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ArgoCD (GitOps controller)                                         │
│                                                                     │
│  Watches gitops-config repo continuously                            │
│  Detects new commit → computes diff vs cluster state               │
│  Prepares sync → sends Pod spec to Kubernetes API server           │
│                                                                     │
│  AppProject scopes: only gitops-config repo, only production/      │
│  staging namespaces, only allowed resource types                    │
└─────────────────────────────────────────────────────────────────────┘
    │
    │  Pod creation request → Kubernetes API server
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  KYVERNO ADMISSION GATE (webhook — runs before pod is scheduled)    │
│                                                                     │
│  Policy: verify-image-signature                                     │
│    ✓ Cosign signature valid?                                        │
│      subject = ci.yml@refs/heads/main exactly                       │
│      issuer  = token.actions.githubusercontent.com                  │
│    ✓ SBOM attestation (spdxjson predicate) present?                │
│    ✓ SLSA provenance attestation present?                           │
│      signer  = slsa-github-generator@v2.0.0                        │
│                                                                     │
│  Policy: restrict-privileged                                        │
│    ✓ No privileged / hostNetwork / hostPID / hostIPC               │
│    ✓ runAsNonRoot, readOnlyRootFilesystem, drop ALL caps            │
│    ✓ CPU + memory limits set                                        │
│    ✓ Image reference contains @sha256: (no mutable tags)           │
│                                                                     │
│  Policy: require-labels                                             │
│    ✓ app.kubernetes.io/name, version, commit labels present        │
│                                                                     │
│  ANY CHECK FAILS → Pod rejected, error returned to ArgoCD          │
│  ALL CHECKS PASS → Pod scheduled                                    │
└─────────────────────────────────────────────────────────────────────┘
    │
    │  Pod running
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  KUBERNETES CLUSTER (production namespace)                          │
│                                                                     │
│  Pod: distroless, nonroot (UID 65532), readOnlyRootFilesystem      │
│  NetworkPolicy: default-deny + allow ingress-nginx:8080 + DNS only │
│  PodDisruptionBudget: minAvailable:1                                │
│  topologySpreadConstraints: spread across zones + nodes            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  FALCO (runtime — every node, kernel syscall level)         │   │
│  │                                                             │   │
│  │  Alert: unexpected outbound connection  → WARNING           │   │
│  │  Alert: shell spawned in container      → CRITICAL          │   │
│  │  Alert: unexpected file write           → WARNING           │   │
│  │  Alert: setuid / setgid call            → CRITICAL          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
    │
    │  Falco alerts → SIEM / Slack / PagerDuty
    ▼
  Security Operations
```

---

### Side-by-Side Comparison

```
┌──────────────────────────────┬──────────────────────────────────────────────┐
│  DEVOPS (no security)        │  DEVSECOPS (this project)                    │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Code pushed                  │ Code pushed                                  │
│ Unit tests run               │ Gitleaks scans for secrets → blocks if found │
│                              │ Kubeconform validates YAML                   │
│                              │ Docker build (no push) verifies Dockerfile   │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ docker build + push          │ docker build + push → immutable digest       │
│ (mutable tag)                │ Trivy scan → blocks on fixable CVEs          │
│                              │ Grype scan → audits with second DB           │
│                              │ Syft → SBOM generated                        │
│                              │ Cosign → image signed (keyless OIDC)         │
│                              │ Cosign → SBOM attested                       │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ —                            │ SLSA provenance (isolated job)               │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ kubectl apply (manual)       │ Digest pinned in gitops-config via kustomize │
│ OR CD watches mutable tag    │ ArgoCD syncs from git (auditable history)    │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Pod starts — no checks       │ Kyverno verifies signature + SBOM + SLSA     │
│                              │ Kyverno enforces no-root, no-latest, limits  │
│                              │ Pod rejected if any check fails              │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ Pod runs — no monitoring     │ Falco monitors syscalls at runtime           │
│                              │ Alerts on any deviation from known-good      │
├──────────────────────────────┼──────────────────────────────────────────────┤
│ No audit trail               │ Every step produces cryptographic evidence   │
│ Non-reproducible             │ Fully reproducible from git commit           │
│ No SBOM                      │ SBOM attached to every image                 │
│ No provenance                │ SLSA provenance on every image               │
│ Secrets can leak             │ Secrets blocked at PR gate                   │
│ CVEs ship to prod            │ CVEs blocked before image is deployed        │
└──────────────────────────────┴──────────────────────────────────────────────┘
```

---

### Infrastructure Required to Run This Project

#### GitHub (SaaS — no self-hosting needed)

| Component | Purpose |
|---|---|
| GitHub repository: `app-source` | Application code + CI pipeline (`ci.yml`) |
| GitHub repository: `gitops-config` | Cluster desired state (Kustomize overlays) |
| GitHub Actions (hosted runners) | CI/CD execution environment |
| GitHub Container Registry (GHCR) | Stores images, signatures, SBOM attestations, SLSA provenance |
| GitHub branch protection on `main` | Enforces PR + status checks + CODEOWNERS |
| GitHub Secret: `GITOPS_PAT` | Fine-grained PAT for `promote` job to write to `gitops-config` |
| Sigstore Rekor (public SaaS) | Transparency log for keyless Cosign signatures |

#### Kubernetes Cluster

| Component | Purpose | Notes |
|---|---|---|
| Kubernetes ≥ 1.29 | Container orchestration | EKS / GKE / AKS / self-hosted |
| Namespaces: `production`, `staging` | Workload isolation | |
| Kyverno ≥ 1.12 | Admission webhook — policy enforcement | Deployed before any workloads |
| ArgoCD ≥ 2.10 | GitOps controller | Watches `gitops-config` repo |
| Falco | Runtime syscall monitoring | Deployed as DaemonSet on every node |
| CNI plugin (e.g. Calico, Cilium) | NetworkPolicy enforcement | Must support NetworkPolicy |
| Ingress controller (ingress-nginx) | External traffic routing | Required by NetworkPolicy allow rule |
| cert-manager (optional) | TLS certificate management | Listed in Kyverno namespace excludes |

#### Cluster Sizing (minimum for this stack)

| Node role | Count | Why |
|---|---|---|
| Control plane | 1–3 | Kubernetes API server, etcd |
| Worker nodes | ≥ 2 | PDB requires minAvailable:1 + topologySpread across nodes |
| Zones | ≥ 2 | topologySpreadConstraints spreads pods across zones |

Falco requires privileged access to the host kernel (eBPF or kernel module). Managed Kubernetes services (EKS, GKE) support this via Falco's eBPF driver.

#### What You Do NOT Need

- A private signing key or key management system (KMS) — keyless signing uses GitHub OIDC
- A private Rekor instance — the public Sigstore Rekor is used (acceptable for non-air-gapped)
- A separate artifact store — GHCR stores images, signatures, and attestations together
- A separate secrets manager for CI — `GITHUB_TOKEN` is auto-provisioned per workflow run; only `GITOPS_PAT` is a stored secret

#### Air-Gapped / Private Deployment Additions

If the cluster has no internet access, these SaaS dependencies must be self-hosted:

| SaaS dependency | Self-hosted alternative |
|---|---|
| Sigstore Rekor (transparency log) | Deploy private Rekor instance |
| GHCR | Private OCI registry (Harbor, ECR, Artifact Registry) |
| GitHub Actions hosted runners | Self-hosted runners |
| slsa-github-generator | Pin to a specific digest; mirror the workflow |

