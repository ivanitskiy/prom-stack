# Prom-Stack

This repo contains a quick start for running a development instance of Prometheus.

## What's Included?

* Prometheus Server
* Push Gateway
* Alertmanager
* Grafana
* Node Exporter
* Process Exporter
* NGINX exporter


```
               +--------------+
               |              |
               |   Grafana    |
               |              |
               +--------------+
                      |
                      | datasource
                      |
               +------v-------+           +--------------+
         +-----+              |           |              |
 +scrape |     |  Prometheus  +-----------> AlertManager |
|        +----->    Server    |   push    |              |
|              |              |   alerts  |              |
|              +--------------+           +--------------+
|                     |
|                     | scrape
|                     |
|              +------v-------+
|              |              |
|              | Pushgateway  |
|              |              |
|              +--------------+
|================= Target Host ===========================
|  +------v-------++
|  |              ||
|-->    Node      ||
|  |   Exporter   ||
|  |              ||
|  +--------------++
|  +------v-------++
|  |              ||
|-->    nginx     ||
|  |   Exporter   ||
|  |              ||
|  +--------------++
|  +------v-------++
|  |              ||
|-->   Process    ||
   |   Exporter   ||
   |              ||
   +--------------++

```

## How do I use it?

1. Clone the repo.
1. Navigate to the directory and run `make up`
1. Go to [http://localhost:9090](http://localhost:9090) for Prometheus.
1. Go to [http://localhost:9091](http://localhost:9091) for the Push Gateway.
1. Go to [http://localhost:9093](http://localhost:9093) for Alertmanager.
1. Go to [http://localhost:3000](http://localhost:3000) for Grafana.

# Guides
## Add a Scrape Target

To add a new scrape target, edit the `scrape_configs` section of `/prometheus/prometheus.yml` and run `make reload-prom`

See [here](https://prometheus.io/docs/operating/configuration/#%3Cscrape_config%3E) for more details on scrape configs.

## Add an alert

To add a new alert, create or edit a `*.rules` file in the `prometheus/alerts` directory and run `make reload-prom`

## Use the Push Gateway

The Pushgateway can be used as an intermediary to push metrics, where the Prometheus pull model doesn't fit.  Examples
of this include short lived processes or batch jobs.

To push a metric in Prom-Stack, you can do something like this:

`echo "mymetric 99" | curl --data-binary @- http://localhost:9091/metrics/job/my-push-job`

You can confirm this has worked by navigating to the [Push Gateway](http://localhost:9091) UI or the [Prometheus](http://localhost:9090) expression browser.

## External host monitoring

### Add node_exporter metrics

Install node-exporter on ubuntu: `sudo apt update && sudo apt -y install prometheus-node-exporter`. Config file on Ubuntu located at `cat /etc/default/prometheus-node-exporter`

then to add node-exporter running on the host system to the prometheus running in a container:
```diff
scrape_configs:
  - job_name: 'prom-stack'
    static_configs:
      - targets:
        - prometheus:9090
        - pushgateway:9091
        - alertmanager:9093
        - grafana:3000
+       - host.docker.internal:9100
```

If want to run node-exporter in a container then follow [this example](https://grafana.com/docs/grafana-cloud/quickstart/docker-compose-linux/), like:
```yaml
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    expose:
      - 9100
```
### Process monitoring

[process-exporter](https://github.com/ncabatoff/process-exporter) can be used to monitor processes with Prometheus

Createa configuration file:
```yaml
process_names:
  # comm is the second field of /proc/<pid>/stat minus parens.
  # It is the base executable name, truncated at 15 chars.
  # It cannot be modified by the program, unlike exe.
  - comm:
    - nginx
```
Docker compose example:

```yaml
version: "3"
services:
  process-exporter:
    image: ncabatoff/process-exporter:0.7.10
    container_name: process-exporter
    restart: unless-stopped
    command: --config.path=/etc/process-exporter/process-exporter.yml --procfs /host/proc -children=false
    privileged: true
    ports:
      - 9256:9256
    extra_hosts:
      - host.docker.internal:host-gateway
    volumes:
      - /proc:/host/proc:ro
      - ./process-exporter/process-exporter.yml:/etc/process-exporter/process-exporter.yml

```

### nginx exporter

[nginx-prometheus-exporter](https://github.com/nginxinc/nginx-prometheus-exporter) can be used to monitor running nginx. Please refer to the [documentation](https://github.com/nginxinc/nginx-prometheus-exporter#getting-started) on details how to set up nginx and prometheus exporter.

Here is an example, when nginx-prometheus-exporter is running in a docker container, nginx is running in a `host` environment. Please update `-nginx.scrape-uri` accordingly:

```yaml
version: "3"
services:
  nginx-prometheus-exporter:
    image: nginx/nginx-prometheus-exporter:0.11
    container_name: nginx-exporter
    restart: unless-stopped
    command: -nginx.scrape-uri=http://host.docker.internal:8080/stub_status
    ports:
      - 9113:9113
    extra_hosts:
      - host.docker.internal:host-gateway

```

## Recommended Grafana Dashboards

Recommended Grafana dashboards can be imported:

- node_exporter dashboard by ID `1860` or from [here](https://grafana.com/grafana/dashboards/1860-node-exporter-full)
- process_exporter dashboard by ID `249` or from [here](https://grafana.com/grafana/dashboards/249-named-processes/)
- nginx_exporter dashboard by ID `12708` or from [here](https://grafana.com/grafana/dashboards/12708-nginx/)
