# Phase 1 — Cloud GPU Design Decision Record

> Tracks issue #2. Quota and image data verified against the lab project on 2026-11-10. Prices are approximate (Nov 2026) and Spot prices change — re-verify before provisioning. Project identifiers are intentionally omitted from this public repo; use `PROJECT_ID` placeholders.

## Decision summary

| Item | Decision |
| --- | --- |
| Machine | `g2-standard-4` (4 vCPU, 16 GB RAM, 1× NVIDIA L4, 24 GB VRAM, compute capability 8.9) |
| Provisioning model | **Spot**, terminated with `DELETE` after at most 8 hours |
| Region / zone | `us-central1` / `us-central1-a`; L4 also offered in `us-central1-b` and `-c` as Spot capacity fallbacks |
| Image | Deep Learning VM `pytorch-2-9-cu129-ubuntu-2204-nvidia-580` (project `deeplearning-platform-release`): NVIDIA driver 580, CUDA 12.9, PyTorch 2.9 preinstalled |
| Boot disk | 100 GB `pd-balanced` (image minimum), `auto_delete = true` |
| Estimated hourly cost | **~$0.44/hr** while running; ~$0 while torn down |
| Quota status | `GPUS_ALL_REGIONS` = **0** — increase to 1 required before provisioning (see [Quota](#quota)) |
| Fallback | Spot `n1-standard-4` + 1× T4 if L4 quota or Spot capacity is unavailable |
| Access | IAP TCP forwarding + OS Login; no inbound internet SSH; no service-account keys |

## Hardware options compared

| | G2 `g2-standard-4` (selected) | N1 `n1-standard-4` + T4 (fallback) |
| --- | --- | --- |
| GPU | L4 (Ada Lovelace, CC 8.9) | T4 (Turing, CC 7.5) |
| VRAM | 24 GB | 16 GB |
| Spot price (VM + GPU) | ~$0.42/hr | ~$0.15–0.20/hr |
| On-demand price | ~$0.71/hr | ~$0.54/hr |
| Notes | GPU bundled in machine type; FP8, current CUDA | GPU attached separately; older but widely available |

G2 is selected for the larger VRAM and current architecture; the price difference is small at the planned usage (see below).

## Quota

Verified 2026-11-10 with read-only `gcloud` queries against the lab project:

| Quota | Scope | Limit | Usage |
| --- | --- | --- | --- |
| `GPUS_ALL_REGIONS` | global | **0** | 0 |
| `CPUS_ALL_REGIONS` | global | 32 | 0 |
| `NVIDIA_L4_GPUS` | us-central1 | 1 | 0 |
| `PREEMPTIBLE_NVIDIA_L4_GPUS` | us-central1 | 1 | 0 |
| `NVIDIA_T4_GPUS` | us-central1 | 1 | 0 |
| `PREEMPTIBLE_NVIDIA_T4_GPUS` | us-central1 | 1 | 0 |
| `CPUS` | us-central1 | 200 | 0 |
| `DISKS_TOTAL_GB` | us-central1 | 4096 | 0 |
| `IN_USE_ADDRESSES` | us-central1 | 8 | 0 |

L4 accelerator availability: `us-central1-a`, `us-central1-b`, `us-central1-c`.

**Status:** regional limits are sufficient, but the global `GPUS_ALL_REGIONS = 0` blocks any GPU VM (including the T4 fallback). Request an increase to **1** (IAM & Admin → Quotas & System Limits → `GPUS-ALL-REGIONS-per-project`) a day or two before provisioning in #3. If denied, fallbacks are: retry after some billing history accrues, or use another region with GPU capacity.

Re-check:

```bash
gcloud compute project-info describe --flatten=quotas \
  --format="value(quotas.metric,quotas.limit,quotas.usage)" | grep GPUS_ALL
gcloud compute regions describe us-central1 --flatten=quotas \
  --format="value(quotas.metric,quotas.limit,quotas.usage)" | grep -E "L4|T4"
```

## Budget estimate

### Rates

| Item | Rate | Billed when |
| --- | --- | --- |
| `g2-standard-4` + L4, Spot | ~$0.42/hr | VM running |
| 100 GB `pd-balanced` boot disk | ~$10/month ≈ $0.014/hr | Disk exists — deleted with the VM |
| Ephemeral external IPv4 (Spot VM) | ~$0.0025/hr | VM running |
| Network ingress (pip, model/dataset downloads) | $0 | — |
| Network egress | ~$0 (results pushed to GitHub are tiny) | — |
| VPC, subnet, firewall rules, service accounts | $0 | — |
| Terraform state bucket (GCS, < 1 MB) | < $0.01/month | Always |
| **Total while running** | **~$0.44/hr** | |

### Scenarios

| Scenario | Estimate |
| --- | --- |
| One run at the 8-hour cap | ~$3.50 |
| Planned monthly usage (40 GPU-hours) | ~$18 |
| Worst case per run (cap enforced by GCP) | ~$3.50 |
| Worst case if the cap were missing and the VM ran a full month (730 hr) | ~$320 — why `max_run_duration` is mandatory |

### Limits

- **Maximum experiment duration: 8 hours per run**, enforced by `max_run_duration`.
- **Monthly allowance: 40 GPU-hours.**
- **Billing budget: $25/month**, alerts at 50% / 90% / 100%, configured by the project owner in the Cloud Console. Budget alerts only notify — they do **not** stop resources.

## Automatic hard stop

GCP deletes the Spot VM when `max_run_duration` elapses or on preemption. With `DELETE` and an auto-delete boot disk, no compute or disk charges remain. Copy results off the VM (commit them or upload to GCS) before a run ends.

```hcl
scheduling {
  provisioning_model          = "SPOT"
  preemptible                 = true
  automatic_restart           = false
  instance_termination_action = "DELETE"
  max_run_duration { seconds = 28800 } # 8h
}
```

> Verify in #3 whether `max_run_duration` requires the `google-beta` provider for the pinned provider version.

## Teardown procedure

**Stop vs delete:** *stop* ends GPU/vCPU billing but the disk keeps billing. *Delete* ends all compute and disk billing. This design prefers delete.

```bash
# 1. What's running?
gcloud compute instances list

# 2. Done — remove everything Terraform created (preferred)
terraform destroy
#    or just the VM (boot disk is auto-deleted)
gcloud compute instances delete gpu-lab-vm --zone us-central1-a

# 3. Verify nothing billable remains
gcloud compute instances list
gcloud compute disks list
gcloud compute addresses list
gcloud compute snapshots list
```

Next day: confirm in **Billing → Reports** that spend dropped to zero.

**Drift after automatic deletion:** when the 8-hour cap or a preemption deletes the VM, Terraform state still lists it. `terraform plan` will show it being recreated; that is expected. The VPC, firewall rules, and service accounts remain and cost nothing.

## Access control

### Project baseline (one-time bootstrap, as project owner)

The lab project was checked on 2026-11-10 and has two insecure defaults that must be removed before any VM is created:

1. **Delete the `default` network and its firewall rules.** It ships with `default-allow-ssh` (tcp:22) and `default-allow-rdp` (tcp:3389) open to `0.0.0.0/0`, plus `default-allow-icmp` and `default-allow-internal`.
   ```bash
   gcloud compute firewall-rules delete default-allow-ssh default-allow-rdp default-allow-icmp default-allow-internal
   gcloud compute networks delete default
   ```
2. **Remove `roles/editor` from the default Compute Engine service account** (`PROJECT_NUMBER-compute@developer.gserviceaccount.com`), or disable it. Nothing in this design uses it.
   ```bash
   gcloud projects remove-iam-policy-binding PROJECT_ID \
     --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
     --role="roles/editor"
   ```
3. **Enable OS Login project-wide**, so any VM requires it by default:
   ```bash
   gcloud compute project-info add-metadata --metadata enable-oslogin=TRUE
   ```
4. **Enable required APIs:** `compute`, `iap`, `oslogin`, `iamcredentials`.
5. **Create** the two service accounts, the state bucket, and the IAM bindings described below.

The project belongs to a personal account with **no Google Cloud organization**, so organization policies (`iam.disableServiceAccountKeyCreation`, `compute.requireOsLogin`, `compute.vmExternalIpAccess`) are unavailable. "No service-account keys" and "OS Login only" are enforced by this design and by review, not by policy.

### Principals

| Principal | Purpose | Roles |
| --- | --- | --- |
| Human (project Owner) | Bootstrap, break-glass, day-to-day SSH | `roles/owner` (existing). Owner includes the access below; it is retained because this is a single-user project. |
| `tf-gpu-lab@PROJECT_ID.iam.gserviceaccount.com` | Terraform, via impersonation | `roles/compute.instanceAdmin.v1`, `roles/compute.networkAdmin`, `roles/compute.securityAdmin`; `roles/iam.serviceAccountUser` **on the VM SA only**; `roles/storage.objectAdmin` **on the state bucket only** |
| `gpu-lab-vm@PROJECT_ID.iam.gserviceaccount.com` | Identity of the VM | `roles/logging.logWriter`, `roles/monitoring.metricWriter` |

If the project is ever shared, a non-owner human needs only: `roles/iap.tunnelResourceAccessor`, `roles/compute.osAdminLogin` (or `roles/compute.osLogin` without sudo), `roles/compute.viewer`, and `roles/iam.serviceAccountUser` on the VM SA.

### Human access: IAP TCP forwarding + OS Login

No inbound SSH from the internet. Path: Mac → IAP (Google-authenticated, IAM-authorized) → VM port 22. OS Login maps the Google identity to a Linux user; no SSH keys in metadata.

- Custom VPC `lab-vpc` with one subnet in `us-central1`, **Private Google Access** enabled.
- Exactly one ingress rule, targeted by **service account** (network tags can be added by anyone who can edit a VM):

```hcl
resource "google_compute_firewall" "allow_iap_ssh" {
  name                    = "allow-iap-ssh"
  network                 = google_compute_network.lab.id
  direction               = "INGRESS"
  source_ranges           = ["35.235.240.0/20"] # IAP TCP forwarding range
  target_service_accounts = [google_service_account.vm.email]
  allow {
    protocol = "tcp"
    ports    = ["22"]
  }
}
```

All other ingress is denied by GCP's implied deny-all rule.

Connect:

```bash
gcloud compute ssh gpu-lab-vm --zone=us-central1-a --tunnel-through-iap
# Port-forward (e.g. Jupyter) without exposing it:
gcloud compute start-iap-tunnel gpu-lab-vm 8888 --local-host-port=localhost:8888 --zone=us-central1-a
```

### Outbound access

| Option | Cost | Trade-off |
| --- | --- | --- |
| Cloud NAT, no external IP | ~$1–2/month (≈ $0.045/GiB processed) | Strongest posture; no public IP |
| **Ephemeral external IP, ingress IAP-only (selected)** | ~$0.10/month at 40 hr | Cheapest; exposure depends on firewall correctness |
| Private Google Access only | $0 | No PyPI/apt access unless staged in GCS/Artifact Registry |

**Decision:** Deep Learning VM image + ephemeral external IP. Never use a reserved static IP (billed while unused). Cloud NAT is kept as a Terraform toggle (`use_cloud_nat`) for the hardened variant and becomes the default in Phase 2.

### VM service account

- Dedicated `gpu-lab-vm` SA with log and metric write only. Add bucket-scoped `roles/storage.objectViewer` later only if needed.
- Narrow access scopes (`logging-write`, `monitoring-write`), not `cloud-platform`.
- No keys — the VM uses metadata-server tokens.

### Terraform identity (no key files)

```bash
gcloud auth login
gcloud auth application-default login
```

The human holds `roles/iam.serviceAccountTokenCreator` on `tf-gpu-lab` (implied by Owner) and Terraform impersonates it:

```hcl
provider "google" {
  project                     = var.project_id
  region                      = var.region
  impersonate_service_account = "tf-gpu-lab@${var.project_id}.iam.gserviceaccount.com"
}
```

### Terraform state

Stored in a GCS bucket, never in Git:

```bash
gcloud storage buckets create gs://PROJECT_ID-tfstate \
  --location=us-central1 \
  --uniform-bucket-level-access \
  --public-access-prevention
gcloud storage buckets update gs://PROJECT_ID-tfstate --versioning
```

```hcl
terraform {
  backend "gcs" {
    bucket                      = "PROJECT_ID-tfstate"
    prefix                      = "phase-1/gpu-vm"
    impersonate_service_account = "tf-gpu-lab@PROJECT_ID.iam.gserviceaccount.com"
  }
}
```

The gcs backend provides state locking. State files, `.terraform/`, `*.tfvars`, and credential files are excluded via `.gitignore`; `.terraform.lock.hcl` is committed for reproducible provider versions.

### Secrets

Phase 1 has no secrets. If one appears (e.g. a Hugging Face token), store it in Secret Manager and grant the VM SA `roles/secretmanager.secretAccessor` on that secret only — never in Terraform variables or state.

### Security assumptions

- Dedicated GCP project used only for this lab, verified to contain no other compute resources.
- Single human principal with 2-Step Verification enabled.
- The Mac is trusted; gcloud credentials in `~/.config/gcloud/` are protected by FileVault and screen lock.

### Residual risks

- **Public IP:** a firewall mistake would expose the VM. Mitigated by deleting the default network, Terraform-owned firewall rules, and SA-targeted rules; Cloud NAT removes the risk.
- **No organization policies:** key creation and external IPs cannot be blocked by policy.
- **Owner on the human account:** broad standing access; acceptable for a single-user lab.
- **Unrestricted egress:** a compromised package could exfiltrate data. Accepted for a lab.
- **Supply chain:** DLVM images and pip/NVIDIA packages are trusted; versions are pinned and recorded per experiment.
- **Local ADC token** can impersonate the Terraform SA; revoke with `gcloud auth application-default revoke` when idle.
- **`osAdminLogin` grants root on the VM**, which is why the VM SA is limited to log/metric writes.

### Phase 2 carry-forward

Private GKE nodes + Cloud NAT become the baseline, with a dedicated minimal node SA, Workload Identity Federation for pods, the same state bucket under prefix `phase-2/gke`, and `kubectl` (IAM + RBAC) replacing SSH.
