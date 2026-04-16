# How Big Should Your Kafka Cluster Be?

*A practical guide to understanding the math, the tradeoffs, and the variables behind every sizing decision — told as a story.*

> This guide has a companion interactive calculator — [kafka-cluster-sizer.html](./kafka-cluster-sizer.html) — that lets you dial in all nine variables and see broker count, partition recommendations, and complexity ratings update in real time. The defaults are set to the 2B messages/day starting point. Use the guide to understand the reasoning; use the calculator to explore your specific numbers.

---

## The Story

Picture a mid-size SaaS company — maybe 3 million active users, a couple of dozen microservices, and an engineering team of 80. They're running Kafka in production. Order events, user activity, CDC from Postgres, application metrics. About **2 billion messages a day**. The cluster is humming along on a handful of brokers. Life is good.

Then the product takes off. A new enterprise tier launches. A partnership doubles their user base. Twelve months later, traffic is 5x what it was. Now they're staring down **10 billion messages a day** — and the engineering manager is asking: do we need more brokers? A bigger instance type? Are we going to fall over at peak?

This is the story most engineering teams actually live. Not tens of trillions — that's Uber, Meta, or a hyperscale network monitoring platform. The real question for the vast majority of companies is: what does a well-sized Kafka cluster look like as we go from 2 billion to 10 billion messages a day, and what variables should we be watching along the way?

The answer depends on at least **nine variables** — and getting even one of them wrong means either a cluster that falls over at peak, or one that wastes budget running at 20% utilization. This guide walks through all of them.

---

## The Cast of Variables

Before we follow that 2B → 10B journey, meet the nine variables the calculator uses. Each one is a dial on your cluster's behavior:

| Variable | What it is | Group |
|---|---|---|
| Messages / day | Total volume — in Millions, Billions, or Trillions | Workload |
| Message size (bytes) | Payload size per message — IoT ~40 B, events ~510 B, logs ~4 KB | Workload |
| Peak multiplier | How much traffic spikes vs the daily average | Workload |
| Consumer fan-out | How many consumer groups independently read the same stream | Workload |
| Retention (hours) | How long messages are kept on broker disk before expiry | Durability |
| Replication factor | How many copies of each partition exist across brokers | Durability |
| Broker NIC (Gbps) | Network interface bandwidth available on each broker host | Hardware |
| Broker disk (TB) | Usable storage per broker after OS and overhead | Hardware |
| Target utilization | What fraction of broker capacity you're willing to use at peak | Hardware |
| Partitions / broker | How many partitions each broker hosts — drives consumer parallelism | Hardware |

---

## Chapter 1 — It All Starts With Bytes Per Second

Here's the thing about Kafka: it doesn't care about messages. It cares about **bytes**. A broker's job is to receive bytes from producers, replicate them to other brokers, and serve them to consumers. Every hardware limit — network bandwidth, disk throughput, disk capacity — is expressed in bytes.

So the first translation you need to make is from messages per day — the number your product manager gives you — into bytes per second — the number your infrastructure needs to handle.

**The core equation:**

```
ingress (bps) = (msgs/day / 86,400) × msg_size × 8
```

Divide by 86,400 to convert days to seconds. Multiply by 8 to convert bytes to bits. This gives you the average ingress rate in bits per second — what brokers receive from producers.

### Our company at 2B messages/day

Back to our mid-size SaaS company. 2 billion messages a day, each around 510 bytes — a typical JSON event with a user ID, event type, timestamp, and a few attributes. Here's what that looks like:

| Step | Calculation | Result |
|---|---|---|
| Msgs per second (avg) | 2,000,000,000 / 86,400 | 23,148 msg/s |
| Bytes per second (avg) | 23,148 × 510 bytes | ~11.8 MB/s |
| Bits per second (avg) | 11.8 MB/s × 8 | ~94 Mbps |
| With RF=3 at peak (3x) | 94 Mbps × 3 × 3 | ~848 Mbps |

848 Mbps of peak traffic across the cluster. A single `kafka.m5.2xlarge` broker has a 25 Gbps NIC — at 60% utilization that's 15 Gbps of headroom. **3 brokers (the RF=3 minimum) is capacity-sufficient for this workload.**

> Note: 3 brokers is the bare minimum topology — every replica is spread across all three, so a single broker failure during a rolling upgrade puts the cluster under real pressure. AWS recommends keeping CPU under 60% specifically to preserve headroom for these events. **Capacity-sufficient and operationally comfortable are not the same thing.**

### Now 5x it

| Metric | 2B / day (today) | 10B / day (5x growth) | Change |
|---|---|---|---|
| Avg msg/s | 23,148 | 115,741 | 5x |
| Avg ingest | 94 Mbps | 472 Mbps | 5x |
| Peak + RF=3 | 848 Mbps | 4.2 Gbps | 5x |
| Retention data (24h, RF=3) | ~3.1 TB | ~15.3 TB | 5x |
| **Brokers needed** | **3** | **3** | **Same!** |

At 10B/day, this still fits on 3 `kafka.m5.2xlarge` brokers. **The key insight: most mid-size companies are not hardware-constrained on Kafka. They are architecture-constrained.** The real question isn't how many brokers — it's whether their topics, partitions, and consumer groups are set up to handle 5x growth gracefully.

### When does it actually get hard?

The inflection point for most companies is around **100B–500B messages/day** — when disk retention starts requiring more brokers than network throughput, and when consumer fan-out begins to multiply egress bandwidth meaningfully. At that scale, instance type selection, tiered storage, and partition strategy all start mattering in ways they simply don't at 2B–10B/day. The chapters that follow explain exactly how each variable behaves as you cross those thresholds.

### Why message size matters more than you think

Doubling your message count and doubling your message size produce the same throughput increase — but they feel very different. Most engineers intuitively track message volume. Fewer track payload size. A team that switches from compact binary events (64 B) to verbose JSON (2 KB) has just multiplied their Kafka infrastructure cost by **30x**, with no change to message rate.

---

## Chapter 2 — The Peak Multiplier: Sizing for Reality

Traffic is never flat. E-commerce spikes at checkout time. Social platforms spike when a trending event breaks. IoT devices all phone home after a firmware push. The peak multiplier captures this — our 10B/day company averages 472 Mbps of ingress, but at 3x peak they need to size for 1.4 Gbps. At hyperscale (say 10T/day), that same 3x multiplier turns 474 Gbps average into 1.4 Tbps peak — a fundamentally different infrastructure problem.

```
peak_ingress = avg_ingress × peak_multiplier
```

This is where a lot of teams get burned. They provision for average load and discover their cluster saturates during the Black Friday or World Cup event they didn't plan for. **Always size for your realistic peak, not your average.**

- **3x** — reasonable default for most event-driven workloads
- **4–5x** — IoT fleets during connectivity storms
- **1.5–2x** — financial systems with more predictable load patterns

---

## Chapter 3 — Three Constraints, One Bottleneck

Once you have your peak ingress number, you need to figure out what limits your brokers. There are exactly three constraints that drive broker count. Your cluster is always limited by whichever one requires the most brokers.

**Network ingress** — Each broker has a finite NIC. At peak, the cluster must absorb `peak_ingress × RF` bits per second across all broker NICs.
```
brokers = ceil( cluster_write / (NIC_bps × utilization) )
```

**Disk retention** — Messages live on disk for your retention window. Total data = ingress rate × retention hours × RF.
```
brokers = ceil( total_bytes / disk_per_broker )
```

**Egress (fan-out)** — Each consumer group reads the full stream. Total egress = cluster write + consumer reads.
```
brokers = ceil( (cluster_write + consumer_egress) / (NIC_bps × utilization) )
```

The binding constraint shifts with your workload shape:
- Small messages at high volume → **network-bound**
- Large messages with long retention → **disk-bound**
- Many downstream consumers → **egress-bound**

The calculator shows you which constraint is driving your broker count so you know where to optimize.

---

## Chapter 4 — Replication: The Hidden Multiplier

Replication factor (RF) is one of the most impactful variables in the calculator, yet it's easy to overlook. When RF=3, every message written to your cluster is actually written three times — once by the producer, twice more as the leader replicates to two follower replicas. RF multiplies both your network load and your disk footprint. A cluster ingesting 474 Gbps with RF=3 actually moves **1.4 Tbps** internally. That's the price of fault tolerance.

| RF | Write availability (minISR=RF-1, acks=all) | Network multiplier | Recommended for |
|---|---|---|---|
| 2 | Tolerates 0 broker losses for writes; 1 for reads | 2x | Dev/test, cost-sensitive |
| 3 | Tolerates 1 broker loss for writes (minISR=2); standard production baseline | 3x | **Production standard** |
| 4–5 | Tolerates 2+ broker losses for writes (with minISR=RF-1) | 4–5x | Regulated / mission critical |

If a broker dies, Kafka can immediately elect a follower as the new leader with no data loss.

---

## Chapter 5 — Retention: How Long Is Too Long?

Kafka's killer feature compared to traditional message queues is that messages aren't deleted after consumption — they're retained on disk for a configurable window. This enables replay, late-joining consumers, and auditing. But it comes at a cost: everything you retain lives on broker disks, multiplied by RF.

```
total_data = (msgs/sec) × msg_size × retention_secs × RF
```

This is the total disk footprint across your entire cluster including replication. **24h retention on a heavy log pipeline can require more brokers than the network constraint alone.**

### Tiered Storage: the escape hatch

MSK with S3 tiered storage, and Confluent's tiered storage offering, decouple hot broker disk from cold retention. Recent data stays on broker NVMe for fast consumer reads; older data is offloaded to S3 automatically. This can reduce broker count driven by disk by **3–10x** — and should be the first option you reach for when retention is your binding constraint.

---

## Chapter 6 — Hardware Variables: What You're Actually Buying

The three hardware variables — NIC bandwidth, disk size, and target utilization — translate the abstract throughput numbers into concrete broker counts.

### Broker NIC (Gbps)
Modern broker hosts typically have 10, 25, or 100 Gbps NICs. NIC is a first-pass planning tool, but on MSK it is not the only bottleneck — EBS storage throughput and EC2 egress throughput are separate ceilings that can bind before raw NIC speed does, especially on `m5.large/xlarge` instances. Always validate against AWS's published per-instance throughput limits. On AWS, kafka.m5.2xlarge gives you up to 25 Gbps.

### Broker disk (TB usable)
NVMe SSDs on broker hosts typically range from 2 TB to 24 TB usable. Higher disk per broker reduces the number of brokers needed when retention is the binding constraint. Spinning disks are viable for retention-heavy workloads but hurt latency. On MSK, provisioned EBS throughput is only available on m5.4xlarge+ and m7g.2xlarge+ — smaller instances cap out at baseline EBS throughput.

### Target utilization (%)
Running brokers at 80% sounds efficient — but leaves almost no headroom for rebalancing, catch-up reads, or traffic spikes. **AWS explicitly recommends keeping CPU User + CPU System under 60%** to preserve headroom for broker replacement, patching, and rolling upgrades. During a rolling update, one broker goes offline and the remaining brokers absorb its partition leadership — that headroom is not optional.

---

## Chapter 7 — Three Workloads, Three Different Problems

The best way to internalize these variables is to see them in action across three very different workloads — each one revealing a different binding constraint.

### IoT Sensor Fleet
*30B msgs/day · 40 B/msg · peak 4x · fan-out 4 · 6h retention · RF=3 · 25 Gbps NIC · 8 TB disk*

| Step | Calculation | Result |
|---|---|---|
| Msgs per second | 30B / 86,400 | ~347K msg/s |
| Avg ingress | 347K × 40 B × 8 | ~111 Mbps |
| Peak ingress | 111 Mbps × 4x | ~444 Mbps |
| Peak with RF=3 | 444 Mbps × 3 | ~1.3 Gbps |
| Retention (RF=3) | 347K × 40B × 21,600s × 3 | ~898 GB |
| Brokers (network) | ceil(1.3 Gbps / (25 × 0.6)) | 1 |
| Brokers (disk) | ceil(898 GB / 8,000 GB) | 1 |

**Result: 3 brokers (minimum for RF=3)**
Tiny messages at moderate volume: network load is almost negligible. The binding factor is the minimum cluster size required by RF=3. With 4 consumer groups, egress is actually the more meaningful design consideration than raw ingest throughput.

---

### Standard Event Streaming
*10T msgs/day · 510 B/msg · peak 3x · 24h retention · RF=3 · 25 Gbps NIC · 12 TB disk*

| Step | Calculation | Result |
|---|---|---|
| Avg ingress | 10T / 86,400 × 510 × 8 | ~474 Gbps |
| Peak ingress | 474 Gbps × 3x | ~1.42 Tbps |
| Peak with RF=3 | 1.42 Tbps × 3 | ~4.25 Tbps |
| Retention (RF=3) | 115.7M/s × 510B × 86,400s × 3 | ~15.3 PB |
| Brokers (network) | ceil(4,250 Gbps / (25 × 0.6)) | 284 |
| Brokers (disk) | ceil(15,300 TB / 12 TB) | 1,275 |

**Result: 1,275 brokers — disk-bound**
Disk is the binding constraint by 4.5x. The network-bound figure of ~284 brokers is dwarfed by the disk requirement. 24h retention at this scale requires 1,275. Tiered Storage is non-negotiable here — offloading cold data to S3 brings broker count back to ~284.

---

### Log Aggregation Pipeline
*5B msgs/day · 4,090 B/msg · peak 2x · 72h retention · RF=2 · 25 Gbps NIC · 24 TB disk*

| Step | Calculation | Result |
|---|---|---|
| Avg ingress | 5B / 86,400 × 4090 × 8 | ~1.9 Gbps |
| Peak ingress | 1.9 Gbps × 2x | ~3.8 Gbps |
| Peak with RF | 3.8 Gbps × 2 | ~7.6 Gbps |
| Retention data | 57.9K/s × 4090B × 259,200s × 2 | ~122 TB |
| Brokers (network) | ceil(7.6 Gbps / (25 × 0.6)) | 1 |
| Brokers (disk) | ceil(122 TB / 24 TB) | 6 |

**Result: 6 brokers — disk-bound**
Far fewer messages than the event streaming case, but much larger payloads and 3x longer retention. RF=2 is acceptable here since logs can typically be re-ingested from source if needed.

---

## Chapter 8 — The Four Big Tradeoffs

**Message size vs. message count**
Both drive throughput equally. But message size also drives disk footprint per message, which compounds with retention. Smaller, more frequent messages are often better for Kafka — batch at the application layer if you need to reduce broker load.

**RF=2 vs RF=3**
RF=2 saves ~33% on both network and disk. It's acceptable for non-critical workloads where data can be re-derived or re-ingested. For anything where message loss is unacceptable — transactions, audit logs, user events — RF=3 is the floor.

**Long retention vs. Tiered Storage**
Every extra hour of retention multiplies your disk footprint. Beyond 24–48 hours, seriously evaluate tiered storage (S3-backed). The economics almost always favor offloading cold data to object storage over buying more broker hosts.

**Many consumer groups vs. dedicated clusters**
Each additional consumer group multiplies your egress bandwidth. At high fan-out (5+), consider whether some consumers should read from a dedicated downstream store (ClickHouse, S3, Elasticsearch) rather than directly from Kafka.

---

## Reference

### All Formulas

| Name | Formula |
|---|---|
| Msgs per second | `msgs_per_sec = msgs_per_day / 86,400` |
| Avg ingress (bps) | `avg_ingress = msgs_per_sec × msg_size × 8` |
| Peak ingress (bps) | `peak_ingress = avg_ingress × peak_multiplier` |
| Cluster write load | `cluster_write = peak_ingress × RF` |
| Consumer egress | `consumer_egress = peak_ingress × fan_out` |
| Total network | `total_network = cluster_write + consumer_egress` |
| Retention (bytes) | `retention = msgs_per_sec × msg_size × retention_secs × RF` |
| Brokers (network) | `ceil( cluster_write / (nic_bps × utilization) )` |
| Brokers (disk) | `ceil( retention_bytes / disk_per_broker )` |
| Brokers (egress) | `ceil( total_network / (nic_bps × utilization) )` |
| Final broker count | `max(net, disk, egress)` — rounded up to nearest multiple of 3 |
| Partitions (heuristic) | `brokers × partitions_per_broker` *(see caveat below)* |
| KRaft controllers | 3 by default; 5 only to tolerate 2 simultaneous controller failures |

### Two honest caveats

1. **Partitions:** the heuristic above is a capacity floor, not a derived formula. Partition count is usually driven by consumer parallelism first. True partition count should come from consumer parallelism requirements (Little's Law: `partitions = throughput × consumer_latency_secs`) and checked against AWS's per-broker partition limits. A future version will add consumer processing time as a first-class input.

2. **Network model:** this guide uses NIC bandwidth as the primary network constraint. On MSK, broker throughput is also bounded by EC2 egress throughput and EBS storage throughput — which can bind before raw NIC speed does, especially on smaller instances. Always validate against AWS's published per-instance throughput ceilings before finalising broker count.

---

### Variable Defaults & Ranges

| Variable | Default | Practical range | Notes |
|---|---|---|---|
| Msgs / day | 10T | 1M – 100T | Use M/B/T toggle in calculator |
| Message size | 510 B | 10 B – 4 KB | IoT=40B, Events=510B, Logs=4KB |
| Peak multiplier | 3x | 1.5x – 8x | 3x general; IoT 4–5x; finance 1.5x |
| Fan-out | 2 | 1 – 10 | # of independent consumer groups |
| Retention | 24 h | 1h – 7d | Use tiered storage beyond 48h |
| Replication factor | 3 | 2 – 5 | RF=3 is production standard |
| Broker NIC | 25 Gbps | 10 – 100 Gbps | kafka.m5.2xlarge default |
| Broker disk | 12 TB | 2 – 96 TB | NVMe preferred for latency |
| Target utilization | 60% | 30 – 80% | AWS recommends ≤ 60% CPU headroom |
| Partitions / broker | 2,000 | 300 – 4,000 | Per AWS MSK best practices |

### Default Instance Type: kafka.m5.2xlarge

| Spec | Value | How it maps to the calculator |
|---|---|---|
| vCPU | 8 cores | Supports 60% utilization target — leaves 3+ cores for replication & rebalancing |
| Memory | 32 GB | ~1 MB heap per partition; 2,000 partitions fits comfortably in broker JVM heap |
| Network | Up to 25 Gbps | Default NIC slider = 25 Gbps; drives the network-bound broker count formula |
| Partitions (AWS rec.) | 2,000 | AWS MSK official recommendation for m5.2xlarge; hard max is 3,000 |
| Throughput / partition | ~10 MB/s | Standard broker baseline on NVMe-backed EBS (gp3) |
| MSK storage max | 16 TB EBS | Default disk slider = 12 TB (leaving headroom below the 16 TB MSK ceiling) |

The partitions/broker slider lets you adjust this for your actual instance. Smaller instances (m5.large, m7g.large) should use 1,000. Larger instances (m5.4xlarge and above) can go up to 4,000 per AWS's published limits.

---

## What's Coming in Future Versions

**Phase 2 — MSK Instance Picker**
Select your MSK broker type — Standard (M5, M7g) or Express (M7g) — and the calculator will automatically populate NIC bandwidth, recommended partitions per broker, throughput per partition, and storage limits. Covering all sizes from `kafka.t3.small` through `kafka.m5.24xlarge` and `express.m7g.16xlarge`. No more manual lookup, with sustained vs. burst throughput clearly distinguished for Express brokers.

**Phase 3 — GCP Managed Kafka & Confluent Cloud**
Switch between platforms and see how broker counts and costs compare side by side. Each platform has different throughput limits, partition caps, and pricing models — useful for multi-cloud architecture decisions or vendor evaluations.

**Phase 4 — Cost Estimation Layer**
Once instance types are first-class inputs, the calculator can estimate monthly infrastructure cost per platform — on-demand vs. reserved pricing, MSK vs. self-managed EC2, EBS gp3 vs. Tiered Storage to S3. All will be surfaced directly in the tool.

**Phase 5 — Consumer Parallelism & Partition Validation**
Add consumer processing time as an input and wire up Little's Law to validate whether your partition count is sufficient for consumer parallelism, not just broker capacity. This will integrate topicpartitions.com's approach into the same tool, giving a complete picture from both the infrastructure and application sides.

---

Did the calculator match your real cluster? Were any variables missing? Was the math off for your workload? Your feedback makes this better for everyone. Open an issue or start a discussion at [github.com/abcdedf/kafka-sizer](https://github.com/abcdedf/kafka-sizer/discussions), or reach out directly. Every real-world data point improves the model.

*kafka-cluster-sizer · companion interactive calculator · v1.3 · April 2026*
