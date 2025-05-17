---
layout: post
title:  "Telemetry pipeline (Local)"
---

## Use case

We have a telemetry pipeline that collects data from different sources and sends it to a central location for analysis (hopefully 😅). The pipeline consists of several components, including data collectors, load balancer, data aggregators, storage systems and UI for analysis. We want to set up a local version of the pipeline for testing and development purposes.

## One possible solution

See the companion repository: [telemetry-pipeline](https://github.com/DavidHernandez21/telemetry-pipeline)

### Components

- **Data collector**: This component collects and preprocesses data from different sources and sends it to the load balancer. In this case, we will use [Vector](https://vector.dev/)
- **Load balancer**: This component receives data from the data collector and distributes it to the data aggregators. In this case, we will use [Nginx](https://www.nginx.com/)
- **Data aggregator**: This component receives data from the load balancer and sends it to the storage system. In this case, we will use [Vector](https://vector.dev/) following the [Aggregattor](https://vector.dev/docs/setup/going-to-prod/arch/aggregator/) architecture which provides higher availability and scalability
- **Storage system**: This component receives data from the data aggregator and stores it for later analysis. In this case, we will use [O2](https://openobserve.org/) and [VictoriaMetrics](https://victoriametrics.com/)
- **UI for analysis**: This component provides a user interface for analyzing the data stored in the storage system. In this case, we will use [Grafana](https://grafana.com/) and [OpenObserve](https://openobserve.org/)

### Architecture

![Telemetry pipeline architecture](/assets/diagrams/pipeline_flow.png)

The data flow is as follows:
1. Data collector collects logs from different sources, which are docker containers in this case. It also scrapes metrics exposed by the containers, mainly in the [Prometheus](https://prometheus.io/) format. Before sending the data to the load balancer, it preprocesses it and adds some metadata to it.
2. The load balancer receives the data from the data collector and distributes it to the data aggregator. In this specific scenario, we are using the [vector sink](https://vector.dev/docs/reference/configuration/sinks/vector/#buffers-and-batches) to send the data to the aggregator so the protocol used is gRPC.
3. Data aggregator receives the data from the load balancer and sends it to the storage system. We want to keep this component as simple as possible, data preprocessing is done in the data collector. The only responsibility of the data aggregator is to receive the data and send it to the storage system (It might need to have credentials to send the data to the storage system).
4. Storage system receives the data from the data aggregator and stores it for later analysis. Here we configure data retention `-retentionPeriod=14d` for VictoriaMetrics and environment variables for O2 to enable local storage e.g. `ZO_LOCAL_MODE_STORAGE=disk`, `ZO_LOCAL_MODE=true` and `ZO_NODE_ROLE=all`.
5. UI for analysis provides a user interface for analyzing the data stored in the storage system. In this case, we are using Grafana and OpenObserve. Grafana is used to visualize the data stored in VictoriaMetrics and OpenObserve is used to visualize the data stored in O2. Both tools provide a powerful user interface for analyzing the data and creating dashboards. The [dasboards](https://github.com/DavidHernandez21/telemetry-pipeline/blob/main/grafana/dashboards) are slightly modified version of the ones published in [grafana](https://grafana.com/grafana/dashboards/), the modifications are mainly to adapt the metric names to the ones used by Vector.

### Running the pipeline

#### Prerequisites
- [Docker](https://www.docker.com/) installed
- [Docker Compose](https://docs.docker.com/compose/) installed
- [cfssl](https://github.com/cloudflare/cfssl) *optional*: used to generate self-signed certificates

#### Steps

1. Clone the repository
2. Generate self-signed certificates (optional)
```bash
cd /certs
cfssl genkey -initca csr.json | cfssljson -bare ca
cfssl gencert -ca ca.pem -ca-key ca-key.pem csr.json | cfssljson -bare cert
```
3. Run the pipeline
```bash
docker compose up -d
```
4. Access the UI for analysis

The UI(s) are served via [nginx](https://www.nginx.com/), this streamlines certificate management and allows us to use a single port for all the services. The services are served on port `8082`, consider though the potential resource requirements for nginx, adding `keepalive` option for each `upstream` block in the nginx configuration can help reduce the number of connections to the backend services. This is particularly useful when using self-signed certificates, as it reduces the overhead of establishing new connections for each request (according to copilot XD)

To access the different UI(s) use the following URLs:
- Grafana: [https://localhost:8082/grafana](https://localhost:8082/grafana) and login with the credentials, i.e. the values of the environment variables `GF_SECURITY_ADMIN_USER` and `GF_SECURITY_ADMIN_PASSWORD`
- OpenObserve: [https://localhost:8082/openobserve](https://localhost:8082/openobserve) and login with the credentials, i.e. the values of the environment variables `ZO_ROOT_USER_EMAIL` and `ZO_ROOT_USER_PASSWORD`
- VictoriaMetrics: [https://localhost:8082/victoriametrics](https://localhost:8082/victoriametrics) to check [VMUI](https://github.com/VictoriaMetrics/VictoriaMetrics/tree/master/app/vmui), I recommend to play with the [cardinality explorer](https://victoriametrics.com/blog/cardinality-explorer/) to check the number of unique values for each metric. This is a good way to check if the data is being sent correctly and if the metrics are being scraped correctly.
- RabbitMQ: [https://localhost:8082/rabbitmq](https://localhost:8082/rabbitmq) and login with the credentials, i.e. the values of the environment variables `RABBITMQ_DEFAULT_USER` and `RABBITMQ_DEFAULT_PASS`

## Bonus

### Collect and send telemetry using Opentelemetry collector

Although not mentioned in the architecture diagram, the pipeline also uses [Opentelemetry collector](https://opentelemetry.io/docs/collector/) to collect telemetry data from the different services and send it directly to the sinks, in this case only Openobserve but in can be easily configured to push data to other sinks. **Caveats** Some receivers do not work properly on WSL2, also metrics names are not *compatible* with many of the published Grafana dashboards.

### Using docker to test before deploying on Kubernetes

Troubleshooting and testing the pipeline using docker compose is a good way to test the pipeline before deploying it on Kubernetes (even [kind](https://kind.sigs.k8s.io/)). It really helps to understand how the different components interact with each other and, **specially**, how to configure them. 
