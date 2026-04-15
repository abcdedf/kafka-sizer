# kafka-sizer

An interactive Kafka cluster capacity calculator. No sign-up. No install.

**[Open Calculator](https://abcdedf.github.io/kafka-sizer/kafka-cluster-sizer.html)** · **[Companion PDF](https://abcdedf.github.io/kafka-sizer/kafka-sizing-guide.pdf)** · **[Companion Markdown](https://github.com/abcdedf/kafka-sizer/blob/main/kafka-sizing-guide.md)**

## What it does

Give it your workload and hardware specs. It computes how many brokers you need and tells you which constraint is driving the number — network ingress, disk retention, or total egress. The answer changes what you do next.

**Inputs:**
- Message size (IoT ~40B · typical ~510B · log lines ~4KB)
- Messages per day (millions, billions, or trillions)
- Peak multiplier, consumer fan-out
- Retention window, replication factor
- Broker NIC bandwidth, usable disk, target utilization
- Partitions per broker

**Outputs:**
- Broker count (rounded to nearest multiple of 3)
- Binding constraint (network / disk / egress)
- Total partitions, controller count
- Avg and peak ingest rates, retention storage, replication traffic per broker
- Operational complexity tier with guidance

## Files

| File | Description |
|---|---|
| `kafka-cluster-sizer.html` | Interactive calculator (v1.3 · April 2026) |
| `kafka-sizing-guide.pdf` | Companion guide — walks through every variable and the math, written as a story not a spec sheet |

## Usage in production

This tool is well-suited as a learning resource and starting point for capacity planning. Use outputs as a directional estimate and validate against your specific hardware, cloud pricing, and observed traffic patterns before making production decisions. Real-world factors like compaction, uneven partition load, and rebalancing overhead are not yet modeled.

## Roadmap

- MSK instance picker
- GCP Managed Kafka support
- Azure Event Hubs / HDInsight support
- Real-world trade-off modeling per cloud provider
- Tiered Storage impact estimator

## Feedback & Discussions

Found something wrong? Have ideas? Open a [GitHub Discussion](https://github.com/abcdedf/kafka-sizer/discussions).

## License

MIT
