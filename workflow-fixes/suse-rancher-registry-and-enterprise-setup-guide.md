# SUSE / Rancher Private Container Registries Analysis & Enterprise Setup Guide

## 1. What Registry Does SUSE / Rancher Use? Is It Harbor?

**Yes — SUSE's enterprise private container registry product (`SUSE Private Registry`, used within the SUSE / Rancher Prime ecosystem) is powered directly by [Harbor](https://goharbor.io/)**, a CNCF-graduated open-source enterprise registry licensed under **Apache-2.0**.

### 1.1 Exact Evidence in Upstream K3s Workflows
In [`.github/workflows/release.yml`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml), upstream K3s publishes multi-architecture container images (`linux/amd64`, `linux/arm64`, `linux/arm/v7`) to four distinct registry targets:

| Target | Registry Identity / Endpoint | Secret Source & Evidence in `release.yml` | Role in Upstream K3s |
|---|---|---|---|
| **Staging SUSE Registry** | `stgregistry.suse.com` (`STAGING_REGISTRY`) | `k3s-suse-registry-stg/credentials` ([`release.yml#L141-L148`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L141-L148), [`#L184-L190`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L184-L190)) | Pre-release / release-candidate (`-rc`) image validation and staging. |
| **Prime Commercial Registry** | `registry.rancher.com` / `registry.suse.com` (`REGISTRY`) | `k3s-suse-registry/credentials` ([`release.yml#L150-L157`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L150-L157), [`#L192-L198`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L192-L198), [`#L414-L418`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L414-L418)) | Commercial subscriber registry for Rancher Prime & enterprise air-gapped mirrors (`sed -i "s|docker.io|${REGISTRY}|g" dist/artifacts/k3s-images.txt`). |
| **Docker Hub** | `docker.io/rancher/k3s` | `dockerhub/rancher/credentials` ([`release.yml#L159-L172`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L159-L172)) | Public community container image distribution. |
| **GitHub Container Registry** | `ghcr.io` | Native `GITHUB_TOKEN` ([`release.yml#L200-L205`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L200-L205)) | Open-source GitHub-native OCI registry (used as the primary target in fork native mode). |

### 1.2 Architecture Behind SUSE Private Registry (Harbor)
Under the hood, **SUSE Private Registry** deploys standard Harbor microservices natively onto Kubernetes (via Helm charts) using hardened SUSE Base Container Images (BCI):
- **`harbor-registry`**: Wraps the underlying CNCF **Distribution** (`distribution/distribution`, formerly Docker Registry v2) engine for OCI push/pull storage.
- **`harbor-core` & `harbor-portal`**: Provides Web UI, project multi-tenancy, granular RBAC, OIDC/LDAP SSO, and robot service accounts.
- **`harbor-trivy-adapter`**: Built-in Trivy vulnerability scanning on push/schedule with CVE severity blocking policies.
- **`harbor-jobservice`**: Automated **proxy caching** and **inter-registry replication** (pulling/mirroring upstream images into air-gapped networks).
- **Supply Chain Extensions**: Supports OCI artifacts, Helm charts, **Cosign** image signatures, **SBOM** attachments, and AI Model Specs.

---

## 2. Registry Options Comparison for Enterprises

| Option | Underlying Engine | License & Cost | Support & Security Features | Best Fit Use Case |
|---|---|---|---|---|
| **GitHub Container Registry (`ghcr.io`)** | GitHub Managed Cloud OCI | **$0** for public images (Free private allowance on GitHub plans, pay per GB above quota) | Native `GITHUB_TOKEN` auth, zero infra ops, linked directly to GitHub Releases & Dependabot/CodeQL. | **Public open-source distribution** of your forked Company X K3s builds. |
| **Upstream Harbor (Self-Hosted Community)** | CNCF Distribution + Harbor Microservices | **Apache-2.0 ($0 license cost)**; runs on your Proxmox/K8s servers | Full RBAC, Trivy scanning, Cosign/Notary signing, proxy cache, air-gapped replication, Web UI. Community support. | **On-premises / private enterprise builds**, air-gapped K3s test/prod clusters, and internal proxy mirroring. |
| **SUSE Private Registry (Rancher Prime)** | SUSE-hardened Harbor | **Commercial Subscription** (Rancher Prime / SUSE Private Registry add-on) | L3 24/7 enterprise SLA, SUSE-curated CVE patching, SLSA Level 3 provenance alignment, verified Helm deployment into Rancher Prime clusters. | Enterprises already standardized on **Rancher Prime** requiring 24/7 vendor SLA and audited supply chain compliance. |
| **CNCF Distribution (`distribution/distribution`)** | Bare-bones OCI registry v2 | **Apache-2.0 ($0)** | Minimal footprint; **no** Web UI, **no** RBAC, **no** vulnerability scanner, **no** replication policies. | Lightweight ephemeral CI/CD caches or K3s `privateregistry` E2E test fixtures only. |
| **Cloud Managed (AWS ECR / Azure ACR / Google GAR)** | Cloud Vendor Managed OCI | Pay-per-GB storage (~$0.10/GB/mo) + egress | Cloud IAM integration, KMS encryption, managed HA & scanning add-ons. | Cloud-only deployments running EKS/AKS/GKE or EC2 workloads in a single cloud vendor. |

---

## 3. What Should You Set Up for Your Enterprise?

For **Company X's forked K3s distribution** (combining public open-source releases with private enterprise dev/test/airgap servers such as Proxmox), the optimal **Dual-Registry Architecture** costs **$0 in software licensing**:

### Tier 1 (Public Open-Source Release Distribution): **GitHub Container Registry (`ghcr.io`)**
- **Setup Cost:** **$0.00** (Zero servers or maintenance).
- **How it works:** Your [`release.yml`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml) automatically authenticates via `${{ secrets.GITHUB_TOKEN }}` and publishes multi-arch images (`ghcr.io/<org>/k3s:<tag>`) alongside GitHub Releases and CycloneDX/OpenVEX artifacts.
- **Optional Mirror:** If customers expect `docker.io`, optionally add a Docker Hub Team/Org secret (`DOCKER_USERNAME`, `DOCKER_TOKEN`) to mirror public tags.

### Tier 2 (Internal Enterprise, Air-Gap & Proxmox Lab Registry): **Self-Hosted Harbor (Community Edition, Apache-2.0)**
- **Setup Cost:** **$0.00 software license** (Apache-2.0 open source). Deploy via the official Harbor Helm chart onto a VM/K3s cluster on your planned physical Proxmox server (allocating ~4 vCPU, 8–16 GB RAM, and mirrored NVMe/S3-compatible MinIO storage).
- **Why Harbor beats plain CNCF Distribution for your enterprise:**
  1. **Pull-Through Proxy Cache:** Mirrors Docker Hub, `ghcr.io`, and `registry.k8s.io` locally so your Proxmox dev VMs and test pipelines **never hit Docker Hub pull rate limits** (100 pulls/6hr).
  2. **Automated Trivy Security Gates:** Scans every internal Company X K3s build on push and blocks deployment of images containing unpatched Critical/High CVEs.
  3. **Air-Gap Replication:** Easily replicates signed OCI artifacts, Helm charts, and K3s airgap images between dev, staging, and isolated customer networks.
  4. **Full Compatibility with K3s `registries.yaml`:** Configures K3s agent/server nodes natively via `/etc/rancher/k3s/registries.yaml` with Robot Accounts and TLS certificates.

### When to Upgrade to Commercial **SUSE Private Registry (Rancher Prime)**
Only purchase **SUSE Private Registry** if:
- Your enterprise customers or compliance auditors contractually mandate **24/7 L3 vendor support SLAs** for the container registry itself (rather than self-supporting upstream Apache-2.0 Harbor), **or**
- You are already purchasing **Rancher Prime** multi-cluster subscriptions and want unified SUSE lifecycle updates and SLSA-3 curated registry images.

---

## 4. Related References
- All underlying web search summaries and source URLs are archived in [`workflow-fixes/web-search-results-archive.md`](file:///c:/projects/k3s_new/k3s/workflow-fixes/web-search-results-archive.md).
