# Slurm and Slinky in Kubernetes

## 1. Overview

This repository provides a practical overview of Slurm and the Slinky project in the context of Kubernetes-based infrastructure.

The goal is to explain the architectural relationship between:

- Slurm as a workload scheduler for HPC and batch compute environments
- Kubernetes as a general-purpose orchestration platform for containerized workloads
- Slinky as a Kubernetes-native approach to integrating Slurm-managed compute resources into a broader Kubernetes platform

The key idea is that modern infrastructure often needs both models: Kubernetes for application and platform orchestration, and Slurm for high-performance compute scheduling and resource accounting.

At a high level: Slurm acts on job priority and queue policy, while Kubernetes does not act on job priority in the same way at the cluster scheduling layer.

This is one of the biggest conceptual differences between the two systems.

---

## 2. What is Slurm?

Slurm is an open-source cluster manager and job scheduler originally built for high-performance computing (HPC). It manages compute resources across nodes and schedules jobs efficiently based on availability, partitions, constraints, and fairness policies.

### Core idea

Slurm answers questions like:

- Which nodes are free?
- Which node is best for this job?
- What priority does this job have?
- Are there limits due to GPU, memory, CPU, or partition policies?
- How do we enforce accounting and access control for shared infrastructure?

### Typical use cases

- AI training and inference clusters
- GPU-heavy scientific workloads
- Batch processing and simulation jobs
- HPC clusters with lots of parallelism
- Multi-tenant environments with strict scheduling policies

### Why Slurm is different from Kubernetes

Kubernetes optimizes for:

- Microservices
- Long-lived services
- Container lifecycle and health
- Declarative workload management
- Auto-recovery and service discovery

Slurm optimizes for:

- HPC batch execution
- Job-level resource accounting
- Partition-based resource allocation
- Large-scale parallel workloads
- High throughput compute scheduling

In practice, both are valuable and often coexist in the same organization.

---

## 3. Slurm architecture: the operating model

A Slurm deployment typically has a small control plane and many compute daemons.

### Primary components

#### 1) slurmctld
The central controller daemon. It:

- Tracks cluster state
- Maintains node and job data
- Determines where jobs should run
- Enforces policies and priorities
- Handles scheduling decisions

This is the brain of the cluster.

#### 2) slurmd
The node daemon running on each compute node. It:

- Starts and manages job steps
- Enforces resource limits
- Maintains local execution state
- Communicates with the controller

If a node is a worker, slurmd is the local execution agent.

#### 3) slurmdbd
The accounting database daemon. It handles:

- Job accounting
- Detailed tracking of usage
- Fair-share and priority calculations
- Association-based quotas

Accounting is a strategic capability in HPC clusters, especially in university and enterprise environments.

#### 4) Munge
Munge provides authentication for Slurm daemons and jobs. It is a critical security mechanism that ensures trust between cluster components.

#### 5) Optional login nodes and clients
Login nodes are used by users to submit jobs, inspect queues, and manage jobs. They often have the Slurm client tools installed but do not run the same control functions as the controller.

---

## 4. Slurm core concepts you must know

### Nodes
A node is a server in the cluster.

You may classify nodes as:

- Controller / management nodes
- Compute nodes
- Login nodes
- GPU nodes
- Special-purpose nodes

### Partitions
Partitions are scheduling groups. Similar to queues, but more structured.

Examples:

- debug
- batch
- gpu
- preemptible
- large-memory

Partitions define:

- node membership
- scheduling policy
- priority
- time limits
- resource constraints

### Jobs
A job is a unit of work submitted to Slurm. It includes:

- requested resources
- time limit
- partition
- constraints
- output handling
- dependencies

### Job steps
A job can contain multiple steps, often executed as part of the same distributed application.

### Nodes and resources
Slurm is very resource-aware. Common resource dimensions include:

- CPU cores
- Memory
- GPUs
- Node count
- Features / topology / GPU model
- Generic resources (gres)

### Reservations
A reservation is a way to reserve resources for a set of users or job types. Useful for planned maintenance, training workshops, or dedicated access.

### Fair-share and priority
Slurm uses prioritization models to decide which jobs should run first.

This often includes:

- Job age
- Size
- Fair-share usage
- Partition priority
- Quality of service (QoS)

### Accounting and association
Users and groups are tracked with accounting data. This matters for:

- Cost allocation
- Usage reporting
- Policy enforcement
- Auditing

---

## 5. Slurm job lifecycle

A typical Slurm workflow is:

1. User submits a job using sbatch, srun, or salloc.
2. Slurm validates the request.
3. The controller evaluates cluster state and scheduling policy.
4. Resources are allocated from the appropriate partition.
5. The job is placed in the queue until resources are available.
6. slurmd on the target node begins execution.
7. Job steps run and report status.
8. Job exits and accounting data is written.

### Common commands

```bash
sinfo        # view cluster state
squeue       # view pending and running jobs
sbatch job.sh
srun --pty bash
sacct        # view accounting data
scontrol show node
scontrol show partition
```

### Practical understanding

Slurm is effectively a batch scheduler for compute-intensive workloads, with an explicit model for cluster state, accounting, and resource ownership. Unlike a general-purpose orchestrator, it is purpose-built for parallel HPC workloads and fairness-driven resource management.

### Example Slurm job script

```bash
#!/bin/bash
#SBATCH --job-name=llm-train
#SBATCH --account=ai-team
#SBATCH --partition=gpu
#SBATCH --qos=high-priority
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --cpus-per-task=32
#SBATCH --mem=256G
#SBATCH --gres=gpu:4
#SBATCH --time=02:00:00
#SBATCH --output=logs/%x-%j.out
#SBATCH --error=logs/%x-%j.err
#SBATCH --mail-type=END,FAIL
#SBATCH --mail-user=user@example.com

set -eux

echo "Starting training job on $(hostname)"
nvidia-smi

srun python train.py \
  --model my-model \
  --batch-size 64 \
  --epochs 10 \
  --data /data/train \
  --output /scratch/checkpoints
```

This example shows the core Slurm model clearly:

- a job is submitted with explicit resource requirements
- the scheduler evaluates partition, QoS, and priority policies
- GPU, CPU, memory, and wall-time requirements are enforced
- the workload runs across a defined allocation of nodes

In practice, Slurm is designed around job-level scheduling, resource fairness, and accounting. Kubernetes is designed around workload placement and lifecycle orchestration.

---

## 6. Why Kubernetes environments often need Slurm-like capabilities

Kubernetes is excellent at:

- orchestrating containerized applications
- service restart and self-healing
- rolling upgrades and scale-out patterns
- ephemeral workloads and APIs

But many AI and HPC workloads require:

- direct access to GPU topology and hardware placement
- high-throughput distributed training jobs
- preemption and reservation-based scheduling
- strict job accounting
- high-performance networking assumptions
- integration with legacy HPC environments

This is where Slurm remains highly relevant.

---

## 7. What is Slinky?

Slinky is a Kubernetes-native project that brings Slurm-based workloads into Kubernetes. The goal is to make Slurm a first-class compute capability inside a Kubernetes environment rather than a fully separate operational island.

### Design intent

Slinky is about enabling a model where:

- Kubernetes remains the control and orchestration surface
- Slurm still provides HPC job scheduling semantics
- workloads can be scheduled and managed as part of a broader platform ecosystem

### Why it matters

Many organizations want to run:

- general Kubernetes workloads for services and APIs
- HPC and AI jobs using Slurm
- shared clusters with a unified platform layer

Slinky helps bridge that gap.

---

## 8. High-level Slinky architecture in Kubernetes

Slinky typically introduces a Kubernetes CRD-driven management layer around Slurm components. It is not simply “Slurm inside a pod”; it is a way to treat Slurm control and compute resources as Kubernetes-managed objects.

### Typical architecture components

#### 1) Kubernetes API as management plane
You define a Slurm cluster or similar resource in Kubernetes via a custom resource or operator-managed configuration.

#### 2) Slinky controller / operator
The operator watches the desired state and reconciles it against the cluster environment.

It may manage:

- Slurm controller deployment
- database state
- compute node pools
- login nodes
- config maps and metadata
- secret-based authentication and trust material

#### 3) Stateful services for Slurm control plane
The Slurm control plane may be represented as Kubernetes workloads such as:

- StatefulSets for controllers and databases
- DaemonSets or node pools for worker nodes
- jobs or init containers for provisioning
- config maps for slurm.conf and related files

#### 4) Compute nodes running slurmd
The actual compute nodes are still Linux servers or VM-based workers that participate in Slurm scheduling, even when they are managed through Kubernetes.

#### 5) Integration with storage and networking
Slurm workloads depend on:

- shared storage
- GPU scheduling awareness
- network policy boundaries
- service discovery and cluster-level connectivity

---

## 9. Slurm vs Kubernetes

### Slurm is optimized for HPC and batch compute

Slurm is designed around:

- node and partition scheduling
- job queues and priorities
- fair-share and resource accounting
- large-scale parallel workloads
- GPU-aware placement and topology-sensitive scheduling

### Kubernetes is optimized for application orchestration

Kubernetes is designed around:

- containers and microservices
- service health and resiliency
- deployment models and rolling updates
- declarative configuration and API-driven control
- pod-level placement and workload orchestration

### The difference in practice

Slurm asks questions like:

- Which nodes are free?
- Which partition should this job use?
- What priority does this job get?
- How do we enforce fairness and accounting across users?

Kubernetes asks questions like:

- Which pod should run where?
- Is the service healthy?
- Is there enough capacity for the workload?
- Can this deployment recover automatically?

Both are schedulers, but they optimize for different kinds of workloads.

---

## 10. What challenges does Slurm solve?

Slurm is primarily built to solve the operational problems of large-scale compute environments.

### 1) Fair sharing across many users
A cluster often has many teams, projects, and jobs competing for the same resources. Slurm helps allocate capacity according to policies and fairness rather than simply letting jobs start first-come-first-served.

### 2) Partition-aware scheduling
Jobs may need access to different classes of hardware, such as:

- debug nodes
- GPU nodes
- large-memory nodes
- general batch nodes

Partitions help isolate workloads by policy, hardware type, and access model.

### 3) Accounting and accountability
Slurm provides explicit accounting for jobs, users, and groups. This is useful for:

- usage reporting
- fairness enforcement
- chargeback or utilization analysis
- auditability in shared systems

### 4) Priority and policy enforcement
Cluster administrators can enforce job prioritization, reservations, and quotas to ensure high-priority research or production work can proceed without unfairly starving other jobs.

### 5) Large distributed jobs
HPC jobs often need to run across many nodes with tight coordination, bulk data movement, and specialized networking. Slurm is designed around that model.

---

## 11. Why Kubernetes is not a replacement for HPC scheduling

Kubernetes is not bad at HPC work; it is just optimized for a different problem.

### Kubernetes is excellent for:

- application services
- stateless and stateful microservices
- short-lived containerized workloads
- service discovery and health checks
- continuous deployment patterns

### Kubernetes is weaker for:

- multi-node batch scheduling with cluster-wide fairness
- partition-based resource isolation
- user/project-level accounting
- strict scheduling policies and reservations
- HPC networking and topology-aware placement
- multi-tenant compute resource enforcement

A Kubernetes cluster can run distributed training or simulation jobs, but it does not naturally model the same cluster scheduling semantics that Slurm provides for HPC environments.

In other words, Kubernetes is a workload platform; Slurm is a compute scheduler.

---

## 12. Idle compute and autoscaling

### Does Slurm turn compute off when it is idle?
Not as a built-in scheduler concept. Slurm focuses on resource assignment and cluster state. Whether machines are powered down or kept online is often handled by infrastructure automation, cloud tooling, or external orchestration.

Typical patterns include:

- keeping nodes online for low scheduling latency
- draining nodes for maintenance
- powering off idle nodes through cloud automation
- scaling node pools based on queue demand or reserved capacity

### Does Kubernetes autoscale?
Yes, but differently.

Kubernetes autoscaling is usually about:

- pod demand
- service demand
- CPU and memory pressure
- node pool growth and shrinkage by cluster autoscaler

So the model is:

- Kubernetes scales compute based on workloads that need to run
- Slurm scales compute based on queued jobs and HPC scheduling policy

The two systems do not solve the same problem in the same way.

### Important distinction

Slurm is not primarily an energy-saving scheduler. It is primarily a resource allocation scheduler for compute jobs. Powering down idle nodes is often a separate operational layer.

---

## 13. Security and trust model

### Slurm security foundations

- Munge for daemon authentication
- TLS for secure communication in modern deployments
- node trust boundaries
- user isolation via Linux permissions and access controls
- restrictive partitions and QoS policies

### Kubernetes security considerations

- Kubernetes RBAC for operators and controllers
- secret management for Slurm config and encryption keys
- network policies between control plane and worker pools
- pod security and admission policies
- workload isolation and access boundaries

The real challenge in hybrid environments is ensuring that both systems maintain a consistent trust model without introducing drift or privilege escalation.

---

## 14. Storage and networking: the hidden complexity

A Slurm cluster looks simple until real AI or HPC workloads are executed.

### Storage challenges

- shared filesystem for job data
- parallel filesystem performance
- scratch space and ephemeral local storage
- multi-tenant data isolation
- Lustre, NFS, Ceph, or distributed storage backends

### Networking challenges

- GPU interconnect latency
- InfiniBand / RoCE networking
- node-to-node communication for distributed jobs
- Kubernetes service networking versus HPC fabric assumptions
- network policy and security restrictions

This is often where infrastructure design matters more than the scheduler itself.

---

## 15. Observability and troubleshooting

### Core observability signals

- sinfo: cluster and partition state
- squeue: pending and running jobs
- sacct: accounting and job history
- slurm logs: controller and daemon logs
- Prometheus metrics if enabled
- node health and kernel metrics
- GPU telemetry

### Typical problems

- nodes appear DOWN or DRAINED
- jobs remain pending due to partition limits or GPU unavailability
- jobs fail due to memory or GPU constraints
- controller cannot communicate with slurmd daemons
- accounting database is unavailable or lagging
- Kubernetes workloads are healthy while Slurm scheduling is blocked

### Troubleshooting workflow

1. Check node state with sinfo and scontrol.
2. Check the job queue with squeue.
3. Validate partition configuration and limits.
4. Inspect Slurm logs for controller and daemon errors.
5. Confirm network connectivity and authentication trust.
6. Validate storage and node-level health.
7. Check Kubernetes-side controller, pod, secret, and CRD state.

---

---

## 17. GitOps and platform operations

### GitOps-friendly design patterns

- store Slurm configuration and cluster definitions in version control
- use custom resources or Helm charts to manage lifecycle
- gate configuration changes through review and policy enforcement
- validate configuration changes before rollout
- track drift and reconcile state

---

## 18. Summary

Slurm and Kubernetes solve different layers of the same infrastructure problem. Slurm focuses on fair, policy-driven scheduling for HPC and batch compute workloads, while Kubernetes focuses on container lifecycle, automation, and platform orchestration. Slinky provides a bridge so Slurm-managed compute resources can participate in a broader Kubernetes-based platform without abandoning the HPC scheduling model.

The real architectural question is not whether Kubernetes or Slurm is better, but how to combine them appropriately for mixed workloads such as service applications, AI pipelines, and large-scale compute jobs.

---

### Quick self-check questions

- What is the role of slurmctld?
- What is the role of slurmd?
- Why is Munge important?
- What is a partition in Slurm?
- Why is accounting important?
- What is the purpose of Slinky in Kubernetes?
- How would you diagnose a job stuck in pending state?
- How do security and networking requirements differ between Kubernetes and HPC clusters?
- Does Slurm turn idle compute off by itself, or is that usually handled externally?
- Why is Kubernetes not a direct replacement for HPC scheduling?
