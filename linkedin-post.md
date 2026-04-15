# LinkedIn Post — kafka-sizer v1.3

---

"Can you resize the Kafka cluster? Our throughput is going up from 2B to 10B messages a day."

Every SDE has heard some version of this. Your product gets popular — great news — and suddenly infra sizing becomes urgent. The manager asks, the team looks at each other, and someone ends up Googling.

I've been that someone asking for it.

I Googled and the resources on the first page looked incomplete or had a sign-up/paywall. No single resource put it all together.

So I spent a week on it and built the resource I wished existed.

It's published as a single `.html` file. No sign-up. No install. Open it and use it.

You tell it your workload — message size, daily volume, peak multiplier, fan-out, retention, replication factor, broker specs — and it tells you exactly how many brokers you need and *why*. Is your cluster constrained by network ingress? Disk retention? Total egress? The answer changes what you do next.

It comes with a companion PDF that walks through every variable and the math behind it — written as a story, not a spec sheet.

This is v1. My aim is to extend it into a full product — with real-world trade-offs and cloud-specific guidance for AWS, GCP, Azure, and others.

If you've wrestled with Kafka sizing, drop a comment below — even a "seen this problem" helps me understand how common it is.

For deeper discussions on the math or ideas to extend it, GitHub Discussions is where I'd love to hear from you — link in the first comment.

#Kafka #SystemDesign #DataEngineering #OpenSource #SoftwareEngineering

---

*First comment:*
GitHub: https://github.com/abcdedf/kafka-sizer

---

*v1.3 · April 2026*
