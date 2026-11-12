# HashiCorp Vault Editions & Enterprise Procurement Guide for K3s Fork

## 1. Clarification: What Is "HashiCorp Vault (SUSE / Rancher Enterprise Cluster)"?

The entry **"HashiCorp Vault (SUSE / Rancher Enterprise Cluster)"** in the cost and tooling documents is **not** a SUSE or Rancher software SKU you buy from SUSE. It is a descriptive label for **SUSE/Rancher's internal deployment of HashiCorp Vault** (from HashiCorp, now owned by IBM), which upstream `k3s-io/k3s` queries during official release pipelines.

**Yes — HashiCorp Vault has an Enterprise Edition**, offered in both **Self-Managed** and **Managed Cloud (HCP)** formats, alongside a free self-hosted Community Edition.

---

## 2. HashiCorp Vault Editions & Pricing Comparison

| Edition / Offering | Hosting Model | License / Contract | Key Capabilities & Limits | Approximate Pricing (List / Market Reference) |
|---|---|---|---|---|
| **Vault Community Edition** | Self-managed (your VM / Proxmox / K8s) | **BSL 1.1** (Business Source License 1.1 — Free for internal corporate use; cannot resell as a managed Vault service) | Core KV secret store, dynamic secrets, PKI, basic Raft HA. **Lacks:** Namespaces, Performance/DR replication, HSM auto-unseal, FIPS builds, vendor SLA. | **$0** software license (you pay for VMs & engineering ops) |
| **HCP Vault Dedicated — Development** | Managed SaaS (IBM/HashiCorp on AWS or Azure) | Pay-as-you-go / Cloud Marketplace | Single-node sandbox, capped at **25 clients**, **no HA SLA**. Uses the Vault Enterprise binary. | **~$0.62 / cluster hour** (~$450/mo continuous); $0 per-client fee |
| **HCP Vault Dedicated — Essentials** | Managed SaaS (IBM/HashiCorp on AWS or Azure) | Pay-as-you-go / Annual Flex contract | **Production High Availability (HA)**, managed upgrades/backups, cloud KMS auto-unseal, Namespaces, vendor SLA. | **~$1.58+ / cluster hour** (~$1,150+/mo base) + **~$72.92 / mo per active client** |
| **HCP Vault Dedicated — Standard** | Managed SaaS (IBM/HashiCorp on AWS or Azure) | Pay-as-you-go / Annual Flex contract | Adds multi-datacenter **Performance & DR Replication**, Sentinel policy-as-code, Advanced Data Protection. | **~$1.85 – $9.40+ / cluster hour** + **~$72.92 / mo per active client** |
| **Vault Enterprise (Self-Managed)** | Self-managed (on-prem, air-gapped, private cloud) | Annual custom enterprise contract (IBM / HashiCorp Sales) | Full enterprise feature parity + **HSM Auto-Unseal**, **FIPS 140-2/140-3** builds, air-gapped networks, custom storage backends, enterprise SLA. | **Custom quote** (typically annual 5-to-6 figure contract based on client/cluster count) |

### How BSL 1.1 Affects Internal Corporate Use
- **Permitted (`$0` License Cost):** HashiCorp's BSL 1.1 FAQ confirms that **internal use is unrestricted**. You may deploy, modify, and self-host Vault Community Edition for your company's internal production infrastructure and CI/CD release workflows without buying a commercial license.
- **Restricted:** You cannot embed or host Vault as a competing **commercial, managed Vault-as-a-Service** sold to external third parties.

---

## 3. Evidence of How Upstream K3s Actually Uses Vault

Inspection of [`.github/workflows/release.yml`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml) confirms that Vault is **never compiled into K3s binaries or shipped to end users**. It is strictly invoked during GitHub Actions tag releases via `rancher-eio/read-vault-secrets@v3` to fetch static KV v2 secrets (`secret/data/github/repo/...`):

| Vault Path / Secret Scope | Injected Environment Variables | Evidence Location in Repo |
|---|---|---|
| Staging SUSE Registry (`k3s-suse-registry-stg/credentials`) | `STAGING_REGISTRY`, `STAGING_REGISTRY_USERNAME`, `STAGING_REGISTRY_PASSWORD` | [`.github/workflows/release.yml#L141-L148`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L141-L148), [`#L329-L335`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L329-L335) |
| Prime SUSE Registry (`k3s-suse-registry/credentials`) | `REGISTRY`, `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` | [`.github/workflows/release.yml#L150-L157`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L150-L157), [`#L336-L341`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L336-L341) |
| Docker Hub Rancher Account (`dockerhub/rancher/credentials`) | `DOCKER_USERNAME`, `DOCKER_TOKEN` | [`.github/workflows/release.yml#L159-L165`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L159-L165) |
| AWS S3 Prime Artifacts Uploader (`prime-artifacts-uploader/credentials`) | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `PRIME_ARTIFACTS_BUCKET_NAME` | [`.github/workflows/release.yml#L320-L327`](file:///c:/projects/k3s_new/k3s/.github/workflows/release.yml#L320-L327) |

---

## 4. Which Subscription Should You Buy for Your Enterprise?

Select the tier that matches your deployment scope and compliance obligations:

1. **If used ONLY for GitHub Actions CI/CD release workflows (`release.yml`) -> Buy NOTHING ($0.00):**
   - Vault is unnecessary for a GitHub-based fork release model.
   - `${{ secrets.GITHUB_TOKEN }}` authenticates GitHub Container Registry (`ghcr.io`) and GitHub Releases automatically for free.
   - Encrypted **GitHub Repository / Environment Secrets** store optional credentials (`DOCKER_USERNAME`, `DOCKER_TOKEN`), and **GitHub OIDC** handles keyless AWS authentication at **$0.00**.

2. **If you want a self-hosted secrets vault on your on-prem Proxmox/VM cluster without commercial support -> Vault Community Edition ($0.00) or OpenBao ($0.00):**
   - Deploy **Vault Community Edition** on your internal Proxmox/K8s server at **$0** license cost under BSL 1.1 internal-use rights.
   - Alternatively, deploy **OpenBao** (the Linux Foundation / MPL-2.0 open-source fork of Vault) if your legal department prohibits BSL 1.1 software entirely.

3. **If your enterprise mandates a vendor-supported Managed Cloud Vault with HA SLA -> Buy `HCP Vault Dedicated — Essentials Tier`:**
   - **Minimum production subscription** with High Availability, automated backups, and IBM/HashiCorp SLA (~$1,150+/mo cluster base + ~$72.92/mo per active client).
   - Step up to **Standard Tier** only if you require active multi-region Disaster Recovery / Performance Replication or Sentinel policy governance.

4. **If your enterprise mandates On-Premises / Air-Gapped deployment, Hardware Security Modules (HSM), or FIPS 140-2/140-3 compliance -> Request a quote for `Vault Enterprise (Self-Managed)`:**
   - Required when cloud-hosted HCP Vault cannot meet data sovereignty or air-gapped defense/finance mandates. Contact IBM/HashiCorp Sales for an annual enterprise license agreement.

---

## 5. Related References
- All underlying web search summaries and source URLs are archived in [`workflow-fixes/web-search-results-archive.md`](file:///c:/projects/k3s_new/k3s/workflow-fixes/web-search-results-archive.md).
