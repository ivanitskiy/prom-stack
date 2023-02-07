# Prom-Stack

This repo contains a quick start for running a development instance of Prometheus.  

## What's Included?

* Prometheus Server
* Push Gateway
* Alertmanager
* Grafana

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
  scrape |     |  Prometheus  +-----------> AlertManager |
         +----->    Server    |   push    |              |
               |              |   alerts  |              |
               +--------------+           +--------------+
                      |
                      | scrape
                      |
               +------v-------+
               |              |
               | Pushgateway  |
               |              |
               +--------------+
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

## Add node_exporter metrics

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

if want to run node-exporter in container then follow [this example](https://grafana.com/docs/grafana-cloud/quickstart/docker-compose-linux/), like:
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
