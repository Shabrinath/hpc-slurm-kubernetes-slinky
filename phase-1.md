# Slurm Phase-1 Approach for On-Prem H100 GPU Cluster

## Goal

This document describes a practical Phase-1 deployment for a Slurm-based GPU cluster running on physical Linux servers or VMs. The goal is to establish a working cluster with:

- one Slurm control-plane VM
- one or more GPU compute nodes running `slurmd`
- shared cluster configuration
- basic job scheduling and validation

This is a simple starting point before integrating Kubernetes, Slinky, or hybrid platform workflows.

---

## Phase-1 architecture

### Core model

- Control-plane VM runs the Slurm management services.
- Compute nodes run `slurmd` and execute jobs.
- Login node (optional but recommended) is used by users to submit jobs.
- All compute nodes expose CPU, memory, and GPU resources to Slurm.

### Typical layout

```text
+---------------------------+
| Control-plane VM          |
|  slurmctld                |
|  slurmdbd (optional)      |
|  munged                   |
|  shared slurm.conf        |
+---------------------------+
            |
            | cluster network
            v
+---------------------------+
| Login node                |
|  sbatch / srun / sinfo    |
+---------------------------+
            |
            v
+---------------------------+
| GPU compute node 1        |
|  H100 + slurmd            |
+---------------------------+
            |
            v
+---------------------------+
| GPU compute node 2        |
|  H100 + slurmd            |
+---------------------------+
```

---

## Phase-1 assumptions

- Linux OS: Ubuntu 22.04 LTS or RHEL/Rocky 8+
- NVIDIA GPU drivers installed on compute nodes
- Shared network connectivity among all nodes
- NTP is synchronized across nodes
- SSH access from controller to compute nodes
- `munge` service is installed and configured consistently

---

## Phase-1 components

### 1) Control-plane VM
Runs:

- `slurmctld` – central scheduler/controller
- `slurmdbd` – accounting and job database (optional but commonly used)
- `munged` – authentication/daemon trust
- `slurm.conf` – cluster configuration

### 2) Compute nodes
Runs:

- `slurmd` – node execution daemon
- NVIDIA drivers and CUDA stack
- optional monitoring and health checks

### 3) Login node
Runs:

- Slurm client tools (`sacct`, `sinfo`, `squeue`, `sbatch`, `srun`)
- user access for submitting jobs

---

## Install steps

## A. Install dependencies on all nodes

### Ubuntu/Debian example

```bash
sudo apt-get update
sudo apt-get install -y munge build-essential libmunge-dev mariadb-server mariadb-client \
  libmysqlclient-dev openssl nfs-common wget curl
```

### RHEL/Rocky example

```bash
sudo dnf install -y munge munge-libs mariadb-server mariadb-devel gcc make openssl-devel
```

Create a shared user for Slurm:

```bash
sudo groupadd -r slurm
sudo useradd -r -g slurm -d /var/lib/slurm -s /usr/sbin/nologin slurm
```

---

## B. Install Slurm binaries

Download the Slurm package for your OS and install it on all nodes.

Example:

```bash
wget https://download.schedmd.com/slurm/slurm-23.02.7.tar.bz2
# or use distro packages where available
```

For Ubuntu, installation is often simpler using repo packages from the Slurm project or OS package manager.

---

## C. Configure Munge

Generate a key on the controller and distribute it to all nodes.

```bash
sudo dd if=/dev/urandom of=/etc/munge/munge.key bs=1 count=1024
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl enable --now munge
```

Copy the same key to all compute/login nodes:

```bash
scp /etc/munge/munge.key user@compute-node:/etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl enable --now munge
```

---

## D. Configure Slurm control plane

### Controller VM setup

Create `/etc/slurm/slurm.conf` with a cluster definition similar to the sample below.

Set the controller host and node definitions:

```text
ClusterName=onprem-ai
ControlMachine=slurm-controller
ControlAddr=10.10.10.10
SlurmUser=slurm
AuthType=Munge
CryptoType=openssl
StateSaveLocation=/var/spool/slurm
SlurmdSpoolDir=/var/spool/slurm/slurmd
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
ProctrackType=proctrack_linux_cgroup
ReturnToService=2
JobCheckpointInterval=300
TaskPlugin=task/cgroup
PlugStackConfig=/etc/slurm/plugstack.conf
```

Then create the database config if using `slurmdbd`.

---

## E. Sample `slurm.conf`

```text
# Slurm configuration example for a Phase-1 H100 cluster

ClusterName=onprem-ai
ControlMachine=slurm-controller
ControlAddr=10.10.10.10
SlurmUser=slurm
AuthType=Munge
CryptoType=openssl
StateSaveLocation=/var/spool/slurm
SlurmdSpoolDir=/var/spool/slurm/slurmd
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
ProctrackType=proctrack_linux_cgroup
ReturnToService=2
TaskPlugin=task/cgroup
JobCheckpointInterval=300
NodeName=compute01 NodeAddr=10.10.10.11 CPUs=64 Sockets=2 CoresPerSocket=16 ThreadsPerCore=2 RealMemory=515000 Gres=gpu:h100:8 State=UNKNOWN
NodeName=compute02 NodeAddr=10.10.10.12 CPUs=64 Sockets=2 CoresPerSocket=16 ThreadsPerCore=2 RealMemory=515000 Gres=gpu:h100:8 State=UNKNOWN
NodeName=compute03 NodeAddr=10.10.10.13 CPUs=64 Sockets=2 CoresPerSocket=16 ThreadsPerCore=2 RealMemory=515000 Gres=gpu:h100:8 State=UNKNOWN

PartitionName=gpu Nodes=compute01,compute02,compute03 Default=NO MaxTime=24:00:00 State=UP
PartitionName=debug Nodes=compute01 Default=YES MaxTime=1:00:00 State=UP

SelectType=select/cons_tres
SelectTypeParameters=CR_CPU_Memory,GRES

AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=slurm-controller
AccountingStorageUser=slurm
AccountingStoragePass=your-db-password
JobCompType=jobcomp/none
```

Notes:

- `Gres=gpu:h100:8` assumes each node has 8 H100 GPUs.
- `RealMemory` is in MB.
- `PartitionName` defines queue-like scheduling policy.
- `SelectTypeParameters` helps Slurm manage CPU, memory, and GPU resources.

---

## F. Install and start Slurm daemons on controller

```bash
sudo systemctl enable slurmctld
sudo systemctl start slurmctld
sudo systemctl status slurmctld
```

If using `slurmdbd`:

```bash
sudo systemctl enable slurmdbd
sudo systemctl start slurmdbd
sudo systemctl status slurmdbd
```

---

## G. Configure compute nodes

On each compute node:

1. Install the same Slurm binaries
2. Copy the same `munge.key`
3. Copy the same `slurm.conf`
4. Ensure `nvidia-smi` works
5. Start `slurmd`

Example:

```bash
sudo systemctl enable slurmd
sudo systemctl start slurmd
sudo systemctl status slurmd
```

---

## H. Validation commands

From the controller or login node:

```bash
sinfo
squeue
scontrol show nodes
scontrol show partition
```

Expected behavior:

- nodes show as `IDLE` or `MIXED`
- partitions are visible
- jobs can be submitted and run

Example test job:

```bash
cat > /tmp/testjob.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=testgpu
#SBATCH --partition=gpu
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:1
#SBATCH --time=00:10:00

nvidia-smi
srun hostname
EOF

sbatch /tmp/testjob.sh
```

---

## Phase-1 success criteria

A Phase-1 environment is considered successful when:

- controller can see all compute nodes
- partitions are visible
- jobs can be submitted
- GPU resources are reported correctly
- Slurm scheduling works without Kubernetes
- cluster can host basic AI/HPC training workloads

---

## What comes next after Phase-1

After the cluster is stable, the next phases may include:

- adding Kubernetes for platform orchestration
- integrating Slurm workloads with Slinky
- introducing shared storage and networking policies
- enabling more complex scheduling, QoS, and accounting
- adding autoscaling and cloud edge patterns

---

## Short interview summary

A Phase-1 Slurm cluster is typically built from a Linux control-plane VM and multiple GPU compute nodes. The controller runs `slurmctld` and policy logic, while each GPU server runs `slurmd` and exposes CPU, memory, and GPU resources. The key to the design is that Slurm decides job placement, queue priority, and resource allocation across shared compute nodes.
