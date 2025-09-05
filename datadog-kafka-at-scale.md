
[Achieving relentless Kafka reliability at scale with the Streaming Platform](https://www.datadoghq.com/blog/engineering/streaming-platform-kafka-custom-abstractions/)<br>
by [Guillaume Bort](https://x.com/guillaumebort)<br>
Feb 19, 2025


### Scale
- 600 Kafka clusters
- more than 10,000 brokers
- 1000s of topics
- 100s of TB/s
- more than a million partitions
- **100s of millions of messages per second!**

### Challenges
- You cannot pause an unhealthy cluster for debugging or wait for a problematic broker to recover ...
  Need to be able to redirect traffic to a healthier cluster and re-route consumers in realtime.

### Technology Stack
- [Apache Kafka](https://kafka.apache.org/)
- Datadog Streaming Platform
- libstreaming: a Datadog custom Kafka client


🚧 Work in Progress ...<br>
🚧 Work in Progress ...<br>
🚧 Work in Progress ...<br>


### Key Takeaways
1. At extreme scale, failures (e.g. disks filling up, failing brokers) happen all the time.

