# SUSE Longhorn Storage Sizing Calculator

A browser-based sizing tool for planning Kubernetes persistent storage deployments on [SUSE Longhorn](https://longhorn.io). Given your hardware specs and workload profile, it estimates the IOPS, throughput, latency, and capacity you can expect — or works in reverse to find the minimum hardware required to meet performance targets.

No installation, no build step, no external dependencies. Open `index.html` in any modern browser.

---

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Operating Modes](#operating-modes)
- [Inputs Reference](#inputs-reference)
  - [Forward Mode (Size from Hardware)](#forward-mode-size-from-hardware)
  - [Reverse Mode (Find Hardware for Target)](#reverse-mode-find-hardware-for-target)
- [Calculation Methodology](#calculation-methodology)
  - [Benchmark Baseline](#benchmark-baseline)
  - [Step 1 — IOPS Ceiling](#step-1--iops-ceiling)
  - [Step 2 — Bandwidth Ceiling](#step-2--bandwidth-ceiling)
  - [Step 3 — Effective Ceiling](#step-3--effective-ceiling)
  - [Step 4 — Per-Volume Performance Under Contention](#step-4--per-volume-performance-under-contention)
  - [Step 5 — Throughput](#step-5--throughput)
  - [Step 6 — Latency Model](#step-6--latency-model)
  - [Step 7 — Capacity Model](#step-7--capacity-model)
  - [Step 8 — Network Utilisation](#step-8--network-utilisation)
  - [Step 9 — Blended Metrics](#step-9--blended-metrics)
  - [Reverse Solver](#reverse-solver)
- [Warnings & Validation Logic](#warnings--validation-logic)
- [Known Limitations](#known-limitations)
- [Disclaimers](#disclaimers)
- [Technology](#technology)

---

## Overview

Longhorn is a distributed block storage system for Kubernetes. Each Persistent Volume (PVC) is backed by a Longhorn volume that replicates its data across multiple nodes for resilience. That replication is the primary driver of performance complexity: writes must be acknowledged by every replica before the I/O is confirmed, and reads can be served locally (with the right locality setting) or must traverse the network.

This calculator models those dynamics so you can answer questions like:

- "If I deploy 50 volumes on a 6-node cluster with NVMe disks, what IOPS will each volume see?"
- "I need 5,000 write IOPS per volume at under 2 ms latency — what hardware do I need?"
- "Will my 25 Gbps network become the bottleneck before my disks do?"
- "How much raw storage do I need to provision 30 TiB of usable capacity with 3 replicas?"

---

## Quick Start

1. Clone or download this repository.
2. Open `index.html` directly in Chrome, Firefox, or any modern browser — no server required.
3. Select **Forward** or **Reverse** mode using the toggle at the top.
4. Adjust the sliders and inputs; results update in real time.
5. Use **Export** to download a JSON or CSV snapshot, or copy the shareable URL (cluster config is encoded in the URL hash).

---

## Operating Modes

### Forward Mode — "Size from Hardware"

You describe your cluster and workload; the calculator tells you what performance to expect. Use this when you have a fixed hardware budget and want to understand what it will deliver.

**Outputs:**
- Read and write IOPS per volume
- Read and write throughput (MB/s) per volume
- Read and write latency (µs) per volume
- Cluster-wide aggregate IOPS and throughput
- Raw, effective, and provisionable capacity (GiB)
- Network utilisation percentage
- Whether any ceiling (disk, network, volume density) is binding

### Reverse Mode — "Find Hardware for Target"

You specify the performance you need per volume; the solver finds the minimum hardware configuration to meet it. Use this when you have SLA requirements and want to know what to buy.

**Outputs:**
- Recommended node count and disk type (SSD or NVMe)
- Disk count per node and disk specifications needed
- Whether the target is achievable at all (some latency targets cannot be met regardless of hardware)

---

## Inputs Reference

### Forward Mode (Size from Hardware)

#### Cluster Configuration

| Parameter | Range | Default | Notes |
|-----------|-------|---------|-------|
| Worker nodes | 3 – 50 | 3 | Minimum 3 for high availability. IOPS ceiling scales linearly with node count per benchmark data. |
| Data engine | V1 / V2 | V1 | V1 uses iSCSI (kernel path). V2 uses SPDK/NVMe-oF (userspace), delivering ~1.8× higher IOPS and ~0.22× lower latency. V2 is most beneficial on NVMe disks. |
| Network bandwidth | 1 / 10 / 25 / 100 Gbps | 10 Gbps | Drives the network bandwidth ceiling. Writes replicate synchronously across the network to all replicas. |

#### Storage per Node

| Parameter | Range | Default | Notes |
|-----------|-------|---------|-------|
| Disk type | HDD / SSD / NVMe | SSD | Selecting a type auto-fills the IOPS and MB/s fields with typical values. You can override these manually. |
| — HDD preset | — | 500 IOPS, 200 MB/s | Spinning disk. High seek latency; not recommended for concurrent workloads. |
| — SSD preset | — | 10,000 IOPS, 360 MB/s | EBS gp3 equivalent; the benchmark baseline. |
| — NVMe preset | — | 100,000 IOPS, 3,000 MB/s | Local NVMe; pairs well with V2 engine. |
| Custom IOPS / disk | 100 – 2,000,000 | (from type) | Per-disk IOPS. The calculator scales proportionally from the 10,000 IOPS SSD baseline. |
| Custom MB/s / disk | 10 – 30,000 | (from type) | Per-disk bandwidth. |
| Disk size (GiB) | 1 – 65,536 | 1,024 | Used for capacity calculations only. Does not affect performance. |
| Disks per node | 1 – 24 | 1 | Multiple disks aggregate IOPS and bandwidth linearly (Longhorn distributes replicas across disks). |

#### Volume Configuration

| Parameter | Range | Default | Notes |
|-----------|-------|---------|-------|
| Replica count | 1 / 2 / 3 | 3 | 1 = no redundancy, highest write performance. 2 = balanced. 3 = default Longhorn HA. Higher replicas increase write network traffic and write latency. |
| Data locality | Disabled / Best-effort / Strict-local | Best-effort | Controls whether the primary replica is kept on the same node as the pod. Strict-local eliminates read network traffic but constrains scheduling. |
| Volume count | 1 – 300 | 10 | Total number of Longhorn volumes across the cluster. Higher volume counts increase contention for the shared IOPS/BW ceiling. |
| Volume size (GiB) | 1 – 65,536 | 100 | Per-volume size. Used for capacity calculations only. |
| Overprovisioning % | 100 – 1,000% | 200% | Longhorn's thin-provisioning multiplier. 200% means you can provision 2× the effective capacity in PVCs before Longhorn refuses new volumes. |

#### Workload Profile

| Parameter | Range | Default | Notes |
|-----------|-------|---------|-------|
| Read % | 0 – 100% | 70% | Percentage of I/O operations that are reads. The remainder are writes. Used for blended metric calculation. |
| I/O pattern | Random / Sequential / Mixed | Random | Random I/O is the hardest case for IOPS; sequential is better for throughput. Each pattern has different benchmark-derived ceiling curves. |
| Block size | 4 KB / 16 KB / 64 KB / 256 KB / 1 MB | 4 KB | Smaller blocks favour IOPS; larger blocks favour throughput. The calculator applies scaling factors derived from benchmark measurements. |

---

### Reverse Mode (Find Hardware for Target)

All performance targets are optional — specify only the constraints you care about.

#### Performance Targets (per volume)

| Parameter | Notes |
|-----------|-------|
| Read IOPS / volume | Minimum sustained read IOPS each volume must achieve. |
| Write IOPS / volume | Minimum sustained write IOPS each volume must achieve. |
| Read MB/s / volume | Minimum sustained read throughput per volume. |
| Write MB/s / volume | Minimum sustained write throughput per volume. |
| Max latency (µs) | Feasibility check only — the solver flags if latency cannot be met regardless of hardware. Does not drive hardware selection. |

#### Workload Profile

Same parameters as Forward mode, plus:

| Parameter | Notes |
|-----------|-------|
| Volume count | Total volumes to serve across the cluster. |
| Replica count | 1 / 2 / 3 — drives write amplification and hardware requirements. |

#### Constraints

| Parameter | Options | Notes |
|-----------|---------|-------|
| Fixed node count | 0 (auto) or 3–50 | 0 lets the solver pick the minimum node count. Lock to a specific count to find disk requirements for an existing cluster. |
| Disk type | Any / SSD only / NVMe only / HDD only | Restricts the solver's search space. "Any" lets it pick the most cost-effective option. |
| Network bandwidth | 1 / 10 / 25 / 100 Gbps | Applied as a hard constraint on write throughput. |

---

## Calculation Methodology

The calculator is not a generic model — it is calibrated to the specific benchmark data published with Longhorn v1.12. All scaling relationships are anchored to measured results and then extrapolated using the documented formulas.

### Benchmark Baseline

Every formula in the calculator is anchored to a reference measurement environment:

| Property | Value |
|----------|-------|
| Longhorn version | v1.12 |
| Instance type | AWS EC2 m5zn.2xlarge |
| Node count | 3 |
| Storage | AWS EBS gp3 (10,000 IOPS, 360 MB/s per disk) |
| CNI | Calico |
| Network | 10 Gbps |
| Replica count | 3 |
| Data locality | Best-effort |
| Block size | 4 KB |
| Tool | kbench (Longhorn's official benchmarking tool) |

All scaling multipliers represent ratios relative to this baseline. When you change a parameter (e.g., increase disk IOPS from 10,000 to 50,000), the calculator scales the ceiling by the same ratio (5×) observed against the baseline.

---

### Step 1 — IOPS Ceiling

The cluster-wide IOPS ceiling is calculated separately for reads and writes.

**Pattern ceilings** are the maximum observed IOPS at the 3-node, 10,000 IOPS SSD baseline:

| I/O Pattern | Read ceiling (IOPS) | Write ceiling (IOPS) |
|-------------|--------------------|--------------------|
| Random | 31,500 | 5,250 |
| Sequential | 61,500 | 10,400 |
| Mixed | 46,500 | 7,825 |

These ceilings represent the aggregate cluster capacity — they are shared across all volumes.

**Scaling formula:**

```
readIopsCeiling  = patternReadCeiling
                 × (diskIops / 10,000)      -- disk quality scaling
                 × (nodeCount / 3)           -- linear node scaling
                 × replicaReadMultiplier     -- per-replica performance factor
                 × localityMultiplier        -- data locality benefit
                 × blockSizeIopsFactor       -- block size impact on IOPS

writeIopsCeiling = patternWriteCeiling
                 × (diskIops / 10,000)
                 × (nodeCount / 3)
                 × replicaWriteMultiplier
                 × localityMultiplier
                 × blockSizeIopsFactor
```

**Per-replica multipliers** capture the complex effect of replication on achievable IOPS. Reads benefit from replicas (more disks serving reads); writes are hurt by replicas (synchronous acknowledgement from all replicas required):

| Replicas | Pattern | Read multiplier | Write multiplier |
|----------|---------|----------------|----------------|
| 3 | Random | 2.20× | 0.115× |
| 2 | Random | 1.65× | 0.115× |
| 1 | Random | 0.50× | 0.29× |
| 3 | Sequential | 1.60× | 0.11× |
| 2 | Sequential | 1.20× | 0.11× |
| 1 | Sequential | 0.40× | 0.22× |
| 3 | Mixed | 1.90× | 0.113× |
| 2 | Mixed | 1.43× | 0.113× |
| 1 | Mixed | 0.45× | 0.255× |

The write multipliers being much less than 1 reflects the reality that write IOPS in Longhorn are severely constrained by synchronous replication overhead — the primary must wait for acknowledgement from all replicas before confirming the write to the application.

**Data locality multipliers** capture how much of the read traffic can be served from local disk vs travelling over the network:

| Locality setting | Ceiling multiplier |
|-----------------|------------------|
| Disabled | 1.00× |
| Best-effort | 1.10× |
| Strict-local | 1.30× |

With strict-local, the scheduler always places the primary replica on the pod's node, so reads are always served locally. This is more beneficial for reads and has no effect on writes.

**Block size IOPS factors** (relative to 4 KB baseline):

| Block size | IOPS factor | Bandwidth factor |
|-----------|------------|-----------------|
| 4 KB | 1.000× | 1.00× |
| 16 KB | 0.400× | 1.60× |
| 64 KB | 0.110× | 1.76× |
| 256 KB | 0.030× | 1.92× |
| 1 MB | 0.009× | 2.30× |

Larger block sizes transfer more data per I/O, so IOPS naturally falls while throughput rises. The bandwidth factors are applied in Step 2.

**V2 engine multiplier:**

If the V2 (SPDK) data engine is selected, an additional 1.8× IOPS multiplier is applied to all ceilings. This is derived from SPDK architecture improvements over the iSCSI kernel path (lower CPU overhead, bypass of kernel network stack). The 1.8× figure is an estimate — see [Known Limitations](#known-limitations).

---

### Step 2 — Bandwidth Ceiling

The bandwidth ceiling is determined by the minimum of disk throughput and network throughput.

**Disk bandwidth aggregate:**
```
diskBwCeiling = diskMbps × disksPerNode × nodeCount × blockSizeBwFactor
```

The block size bandwidth factor (from the table in Step 1) captures that large-block I/O extracts more of the disk's rated MB/s.

**Network bandwidth ceiling:**

The network imposes different constraints on reads and writes.

For **writes**, every write must be synchronously replicated to all `replicaCount` replicas. This means a single write generates `replicaCount` network writes:
```
networkWriteCeiling = (networkGbps × 125) / replicaCount
```

For **reads**, the network constraint depends on how much of the read traffic traverses the network:
```
readNetFactor = {
  strict-local:  0.05   -- almost all reads served locally, minimal network
  best-effort:   0.25   -- some reads still cross the network
  disabled:      1.00   -- all reads must be served over the network
}

networkReadCeiling = (networkGbps × 125) / readNetFactor
```

The conversion `Gbps × 125` converts Gigabits per second to MegaBytes per second (÷ 8 × 1000).

---

### Step 3 — Effective Ceiling

The disk and network ceilings are independent constraints. The effective ceiling is the minimum of both, converted to a common unit (IOPS):

```
-- Convert bandwidth ceilings to IOPS-equivalent
bwToIops(mbps) = (mbps × 1024) / blockSizeKb

-- Effective read ceiling (min of disk IOPS and network BW expressed as IOPS)
effectiveReadCeiling  = min(readIopsCeiling,  bwToIops(min(diskBwCeiling, networkReadCeiling)))

-- Effective write ceiling (min of disk IOPS and network BW expressed as IOPS)
effectiveWriteCeiling = min(writeIopsCeiling, bwToIops(min(diskBwCeiling, networkWriteCeiling)))
```

This produces a single ceiling number (in IOPS) for the entire cluster for reads and writes separately. Whichever resource is binding — disk IOPS, disk bandwidth, or network — determines this number.

---

### Step 4 — Per-Volume Performance Under Contention

All volumes compete for the shared cluster ceiling. The calculator models this as a proportional division:

```
-- What all volumes are demanding in aggregate
totalDemandRead  = volumeCount × rawReadIopsPerVolume
totalDemandWrite = volumeCount × rawWriteIopsPerVolume

-- Cluster can only deliver up to its ceiling; excess demand is throttled
clusterReadIops  = min(totalDemandRead,  effectiveReadCeiling)
clusterWriteIops = min(totalDemandWrite, effectiveWriteCeiling)

-- Each volume receives a fair share
volReadIops  = clusterReadIops  / volumeCount
volWriteIops = clusterWriteIops / volumeCount
```

`rawReadIopsPerVolume` and `rawWriteIopsPerVolume` represent the unconstrained demand of a single volume — what it would consume if there were no ceiling. These are set to typical Longhorn per-volume demands derived from the benchmark data.

When `totalDemand < ceiling`, the cluster is not ceiling-bound and each volume gets its full unconstrained performance. When `totalDemand ≥ ceiling`, the ceiling is fully shared across all volumes — adding more volumes proportionally reduces each volume's share.

---

### Step 5 — Throughput

Throughput is derived directly from IOPS and block size:

```
volReadMbps  = (volReadIops  × blockSizeKb) / 1024
volWriteMbps = (volWriteIops × blockSizeKb) / 1024
```

---

### Step 6 — Latency Model

Latency is modelled in two parts: a base latency determined by hardware and configuration, and a contention multiplier that grows with volume density.

**Base latency:**
```
latencyBase = 750 µs × diskLatencyFactor × engineLatencyFactor × replicaLatencyFactor
```

The 750 µs constant is the measured baseline read latency at 3 replicas, SSD, V1 engine, best-effort locality in the benchmark environment.

**Disk latency factors:**

| Disk type | Factor |
|-----------|--------|
| HDD | 8.0× |
| SSD | 1.0× (baseline) |
| NVMe | 0.15× |

HDD's rotational seek time dominates; NVMe's low-latency queue depth eliminates most of it.

**Engine latency factors:**

| Engine | Factor |
|--------|--------|
| V1 (iSCSI) | 1.0× (baseline) |
| V2 (SPDK) | 0.22× |

V2 bypasses the kernel network stack entirely, dramatically reducing latency, particularly for NVMe.

**Replica latency factors:**

| Replicas | Factor |
|----------|--------|
| 1 | 0.65× |
| 2 | 0.85× |
| 3 | 1.00× (baseline) |

Fewer replicas means fewer synchronous acknowledgements required, so latency is lower.

**Contention multiplier:**

When many volumes are packed onto few nodes, they compete for disk and CPU resources, driving up latency:

```
volumeDensity   = (volumeCount / nodeCount) × 3   -- normalised to 3-node equivalent

contentionMult  = 1 + max(0, (volumeDensity - 1) / 50) × 1.107
```

The contention effect is zero up to approximately 1 volume per node (normalised), then grows linearly. The constant 1.107 was empirically fitted to the benchmark contention curve. The model's accuracy degrades significantly above a density of 51 volumes per 3-node equivalent — see [Known Limitations](#known-limitations).

**Final latency:**
```
readLatency  = latencyBase × contentionMult
writeLatency = readLatency × 1.85
```

Writes are consistently ~1.85× higher latency than reads because they require synchronous acknowledgement from all replicas before completing.

---

### Step 7 — Capacity Model

```
rawCapacityGiB         = nodeCount × disksPerNode × diskSizeGiB

effectiveCapacityGiB   = rawCapacityGiB / replicaCount

provisionableCapacityGiB = effectiveCapacityGiB × (overprovisioningPct / 100)

totalVolumeSizeGiB     = volumeCount × volumeSizeGiB

storageEfficiency      = (1 / replicaCount) × 100%
```

**Definitions:**

- **Raw capacity** — total disk space across all nodes.
- **Effective capacity** — usable space after accounting for replicas. With 3 replicas, only one-third of raw capacity is available for user data.
- **Provisionable capacity** — how much Longhorn will allow you to allocate across all PVCs. Longhorn's overprovisioning ratio allows thin-provisioned volumes to sum to more than the effective capacity, on the assumption that volumes are not all 100% full simultaneously. The default 200% means you can create PVCs totalling 2× your effective capacity.
- **Total volume size** — the sum of all requested volume sizes, used to check whether you have overcommitted the provisionable capacity.

A warning is raised if `totalVolumeSizeGiB > provisionableCapacityGiB`.

---

### Step 8 — Network Utilisation

```
-- Writes are replicated to all replicas; reads cross the network based on locality
netWriteLoadMbps = clusterWriteMbps × replicaCount
netReadLoadMbps  = clusterReadMbps  × readNetFactor

networkUtilisation = (netWriteLoadMbps + netReadLoadMbps) / (networkGbps × 125)
```

A warning is raised when utilisation exceeds 80%, indicating that the network is approaching saturation and latency will increase significantly.

---

### Step 9 — Blended Metrics

The blended (combined) metrics weight read and write performance by the configured read/write ratio:

```
blendedIops  = volReadIops  × (readPct / 100)  +  volWriteIops  × (writePct / 100)
blendedMbps  = volReadMbps  × (readPct / 100)  +  volWriteMbps  × (writePct / 100)
blendedLatUs = readLatency  × (readPct / 100)  +  writeLatency  × (writePct / 100)
```

These are the headline summary figures shown on the results card.

---

### Reverse Solver

The reverse solver works by iterating over candidate hardware configurations and applying the forward model to each one, keeping the first configuration that meets all targets.

**Search space:**

The solver evaluates combinations of:
- Node counts: [3, 4, 5, 6, 8, 10, 12, 16, 20, 24, 32, 40, 50] (if not locked)
- Disk types: SSD (10,000 IOPS, 360 MB/s) and NVMe (100,000 IOPS, 3,000 MB/s) (filtered by constraint)
- Disk counts per node: 1 through 12

**Selection criteria:**

For each candidate, the forward model is run and the result is checked against all specified targets. The solver prefers:
1. Meeting all targets
2. Minimum node count (fewest nodes = lowest cost)
3. SSD over NVMe if both achieve the target (lower cost)
4. Fewest disks per node

If a fixed node count is set, only that count is evaluated. If no configuration meets the target, the solver reports infeasibility with the closest candidate it found.

**Latency targets:**

The solver does not use latency to drive hardware selection — it only checks whether the best hardware configuration's modelled latency is below the target. If not, it reports an infeasibility warning, since latency floors are set by physics (NVMe + V2 engine + 1 replica) and no amount of extra hardware can reduce them further.

---

## Warnings & Validation Logic

The calculator raises the following warnings when conditions are detected:

| Warning | Condition | Recommendation |
|---------|-----------|----------------|
| HDD instability risk | Disk type = HDD | Use SSD or NVMe, especially for concurrent workloads (rebuilds + application I/O) |
| No redundancy | Replica count = 1 | Any disk failure results in data loss |
| Read IOPS ceiling reached | Cluster read demand ≥ effective read ceiling | Add nodes, upgrade disks, reduce volume count, or increase data locality |
| Write IOPS ceiling reached | Cluster write demand ≥ effective write ceiling | Add nodes, upgrade disks, reduce volume count, or increase replica count |
| Network saturation | Network utilisation ≥ 80% | Increase network bandwidth, reduce replica count, or enable strict-local locality |
| High volume density | (volumes/nodes)×3 > 51 | Latency model accuracy degrades; consider adding nodes to spread load |
| V2 without NVMe | Data engine = V2 and disk type ≠ NVMe | V2 (SPDK) benefits are minimal without NVMe disks; consider V1 to simplify operations |
| Capacity overcommit | Total volume size > provisionable capacity | Reduce volume sizes, reduce overprovisioning ratio, add disks, or reduce replica count |
| Strict-local node mismatch | Locality = strict-local and node count < replica count | Longhorn cannot schedule strict-local replicas; pods may not start |

---

## Known Limitations

### 1. Single Reference Hardware Profile

The entire model is calibrated to one specific environment: EC2 m5zn.2xlarge instances with EBS gp3 storage and Calico CNI. Your results will differ if you use:

- Bare-metal servers (typically better IOPS, lower latency, higher variance)
- Local NVMe vs EBS (very different latency characteristics)
- A different CNI (Flannel, Cilium, etc. have different network path efficiencies)
- ARM instances or different CPU architectures
- High-memory vs compute-optimised instance types

Use the results as a planning order-of-magnitude estimate, then validate with `kbench` in your actual environment.

### 2. V2 Engine Estimates Are Approximate

The V2 (SPDK) data engine multipliers (1.8× IOPS, 0.22× latency) are estimates based on SPDK architecture documentation and limited published benchmark comparisons. Longhorn v1.12's V2 engine is relatively new and public benchmark data is sparse. Actual V2 performance is highly sensitive to CPU pinning configuration, NVMe queue depth settings, and NUMA topology.

### 3. Linear Node Scaling Is Assumed

The model assumes IOPS scales linearly with node count: a 6-node cluster delivers 2× the ceiling of a 3-node cluster. This relationship was validated only at the 3-node baseline. In practice, cross-node coordination overhead, network congestion, and Longhorn scheduling behaviour may cause sublinear scaling at higher node counts. This effect is not modelled.

### 4. Latency Model Accuracy Degrades at High Density

The contention latency multiplier is an empirical fit valid up to approximately 51 volumes per 3-node equivalent (~17 volumes/node). Above this density, Longhorn's internal I/O manager, CPU contention, and kernel scheduler interactions create complex non-linear behaviour that the simple linear fit cannot capture. The calculator will still display numbers, but they should be treated as unreliable above this threshold.

### 5. No Snapshot, Clone, or Background Operation Modelling

Longhorn runs background operations that consume disk and network I/O:

- Replica rebuilds (after node restarts or disk failures)
- Snapshot creation and deletion
- Volume backups to S3 or NFS
- Recurring volume trim operations

These can consume 20–80% of available I/O during execution. The calculator models only steady-state application I/O and ignores these background operations.

### 6. No Caching or Buffer Effects

The model assumes all I/O reaches the storage medium. In practice, Linux page cache, Longhorn's read cache, and filesystem journal buffers can significantly improve read performance and reduce write amplification for certain workloads. These effects are workload-specific and cannot be generalised.

### 7. Synchronous Replication Only

The network model assumes all writes are synchronous: the primary waits for all replica acknowledgements before confirming to the application. Longhorn does support this model, but future versions may introduce async replica options. This calculator does not model any async or quorum-based replication scheme.

### 8. Thin-Provisioning Growth Not Modelled

The overprovisioning ratio is modelled as a static multiplier. In reality, volumes fill up over time. The calculator cannot model volume fill rates, predict when you will run out of real capacity, or suggest tiering strategies. Monitor actual disk utilisation separately.

### 9. Single Disk Size per Node

The calculator assumes all disks on all nodes are identical in size, IOPS, and throughput. Mixed-disk configurations (e.g., NVMe OS disk + large SSD data disks) are not modelled.

---

## Disclaimers

**This tool provides estimates, not guarantees.** Storage performance is highly sensitive to workload characteristics, hardware quality, kernel version, network topology, and Longhorn configuration parameters not captured in this calculator. Results may vary significantly from what you observe in production.

**Always validate with kbench.** Longhorn ships `kbench` specifically to measure actual cluster performance. Run `kbench` in your target environment before committing to a cluster design based on these estimates.

**Not affiliated with SUSE or Longhorn.** This is an independent community tool. It is not maintained or endorsed by SUSE, Rancher Labs, or the Longhorn project.

**Benchmark data may become stale.** The calibration data is pinned to Longhorn v1.12 benchmark results. Future Longhorn releases may change performance characteristics — especially for the V2 engine which is under active development.

**Capacity estimates assume no data corruption or disk degradation.** Real storage systems experience disk wear, filesystem fragmentation, and RAID degradation that reduce effective capacity and performance over time.

**Financial decisions should not be based solely on this tool.** Use these estimates as one input among many when sizing storage for production systems. Consult your cloud provider's documentation, Longhorn's official documentation, and your storage vendor for authoritative guidance.

---

## Technology

This project is intentionally minimal. It is a single HTML file with no build toolchain, no package manager, and no external runtime dependencies.

| Component | Choice |
|-----------|--------|
| Language | Vanilla JavaScript (ES6+) |
| Styling | CSS3 with custom properties |
| HTML | HTML5, no template engine |
| Frameworks | None |
| Build system | None |
| Dependencies | None (Google Fonts CDN for the Inter typeface only) |
| State persistence | URL hash (base64-encoded JSON) — shareable links work out of the box |
| Export formats | JSON, CSV, PDF (via browser print dialog) |

To deploy, copy `index.html` to any static file host (GitHub Pages, S3, Nginx, or just open it locally). No server-side processing is required.
