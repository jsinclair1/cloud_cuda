# Cloud CUDA — GPU Infrastructure Engineering Lab

A hands-on engineering lab for learning how GPU workloads run **from cloud infrastructure down to Linux, NVIDIA drivers, and CUDA**—then scaling those workloads into secure, observable, cost-conscious GPU clusters.

> **Status:** Early-stage learning project. The roadmap below describes planned work, not completed implementations.

## Objectives

- Understand the full hardware–software path of a GPU workload: application → CUDA libraries/runtime → NVIDIA driver → GPU.
- Provision repeatable cloud GPU environments with infrastructure as code.
- Deploy containerized GPU workloads and eventually orchestrate them on Kubernetes.
- Investigate GPU utilization, memory, data movement, networking, and distributed execution.
- Apply least-privilege security, operational reliability, and disciplined cost management.
- Record reproducible experiments and measurable engineering outcomes.

## Roadmap

| Phase | Project | Scope | Definition of done |
| --- | --- | --- | --- |
| 1 | **GPU Systems Lab** | GCP GPU VM, Linux, NVIDIA drivers, CUDA, PyTorch, profiling | Reproducible environment, working CUDA workload, baseline benchmarks and documented results |
| 2 | **Cloud GPU Platform** | Terraform, container images, GKE GPU nodes, scheduling, IAM, monitoring | Workload deploys through documented automation; security controls and operational metrics are verified |
| 3 | **Distributed GPU Cluster** | Multi-GPU/multi-node workloads, NCCL, network and storage performance | Distributed workload executes; scaling efficiency, bottlenecks, failure behavior, and cost are measured |

### Phase 1 — GPU Systems Lab

1. Research GPU architecture, memory hierarchy, and the CUDA execution model.
2. Provision a GPU-enabled Linux VM in Google Cloud.
3. Inspect the hardware and driver stack (`lspci`, `lsmod`, `nvidia-smi`).
4. Run a CUDA-enabled PyTorch workload and a simple CUDA C++ kernel.
5. Benchmark CPU vs GPU execution; profile utilization, memory, and data transfers.
6. Automate provisioning, document findings, and tear down unused resources.

### Phase 2 — Cloud GPU Platform

1. Provision infrastructure using Terraform.
2. Package a reproducible GPU workload in a container.
3. Deploy to Google Kubernetes Engine (GKE) using GPU-capable nodes.
4. Configure workload identity, least-privilege IAM, network controls, and secret handling.
5. Implement monitoring, resource limits, and repeatable deployment workflows.
6. Measure throughput, latency, utilization, and cost under different loads.

### Phase 3 — Distributed GPU Cluster

1. Explore GPU topology, PCIe/NVLink, network fabric, and collective communication.
2. Run multi-GPU and, where feasible, multi-node PyTorch workloads using NCCL.
3. Compare single-GPU and distributed performance.
4. Profile communication overhead, input pipelines, and memory pressure.
5. Document scaling efficiency, failure recovery, and cost/performance tradeoffs.

## Planned technology stack

| Layer | Technologies |
| --- | --- |
| Cloud | Google Cloud Platform (Compute Engine, GKE, IAM, VPC, Cloud Monitoring) |
| Infrastructure as code | Terraform |
| OS and automation | Linux, Bash, Python |
| GPU compute | NVIDIA GPU drivers, CUDA, CUDA Toolkit, PyTorch |
| Containers and orchestration | Docker/containerd, Kubernetes |
| Distributed workloads | PyTorch Distributed, NCCL |
| Diagnostics | `nvidia-smi`, NVIDIA profiling tools, logs and metrics |
| Project management | GitHub Issues and GitHub Projects |

Specific GPU types, machine families, CUDA versions, and tooling will be selected and recorded per experiment rather than assumed to be compatible.

## Engineering principles

**Reproducibility:** Record software versions, hardware configuration, infrastructure definitions, commands, and benchmark inputs.

**Security:** Use least-privilege identities, minimize exposed services, protect secrets, and document workload isolation assumptions.

**Performance:** Profile before optimizing. Compare against a baseline and explain what changed.

**Reliability:** Test deployment, recovery, observability, and failure cases—not just the happy path.

**Cost:** Estimate cloud costs before provisioning, set budgets and alerts, and destroy idle resources. GPU VMs and cluster nodes may incur significant charges even when no workload is running.

## Experiment documentation

For each experiment, record:

- **Question / hypothesis:** What behavior are we testing?
- **Environment:** Cloud region, VM/GPU model, driver, CUDA, OS, container and framework versions.
- **Method:** Commands, code revision, input sizes, warm-up and measurement procedure.
- **Metrics:** Throughput, latency, GPU/CPU utilization, GPU memory, network or storage I/O, and estimated cost where relevant.
- **Results:** Baseline, new measurements, and any limitations.
- **Conclusions:** What was learned, which tradeoffs matter, and what to test next.

## Tracking progress

Work is managed through [GitHub Issues](https://github.com/jsinclair1/cloud_cuda/issues) and an associated GitHub Project.

Suggested issue workflow: **Backlog → Ready → In Progress → Review / Validate → Done**.

A task is complete when its acceptance criteria are met and the evidence (configuration, code, logs, benchmarks, or documentation) is committed or linked.

Do **not** commit cloud credentials, service-account keys, tokens, or other secrets. Review quotas, billing, and IAM before running cloud examples.

## References

- [NVIDIA CUDA documentation](https://docs.nvidia.com/cuda/)
- [NVIDIA NCCL documentation](https://docs.nvidia.com/deeplearning/nccl/)
- [Google Cloud: GPUs on GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/gpus)
- [Terraform Google provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [PyTorch CUDA semantics](https://docs.pytorch.org/docs/stable/notes/cuda.html)

---

**North star:** Build a secure, reproducible GPU computing platform, understand every critical layer, and prove improvements through measurement.
