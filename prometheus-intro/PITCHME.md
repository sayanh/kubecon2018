### Prometheus
- Prometheus is an open-source toolkit for monitoring and alerting
- It is a pull based metrics gathering system
- Simple text for metrics representation
- PromQL: powerful query language
+++
#### Metrics Format
![Image](./assets/md/assets/Prom_svc_port_fwd.png)
![Image](./assets/md/assets/Prom_metrics_text_eg.png)

---
### Prometheus Exporter
- System Metrics : Node exporter - exposes OS and hardware metrics (linux)
- Application metrics - a spring boot app must expose its endpoint
-

---

### Prometheus2   Features

- Optimized Scraping
- New TSDB (Can persist 1,000,000+ samples/core/sec to disk)
- Rule format in standard yaml instead of proprietary DSL
+++
![Image](./assets/md/assets/Prom_rule_format.png)
+++
![Image](./assets/md/assets/Prom_Bench_1.png)
+++
![Image](./assets/md/assets/Prom_Bench_2.png)
+++
![Image](./assets/md/assets/Prom_Bench_3.png)

---
Example spring-boot app
How to start it on a minikube show





---
Draft
alert manager
metrics collection system
keynote prom 2.0 to get long term storage and other slides
custom metrics and auto scaling
TSDB DB
