# docket-monitoring

A monitoring stack for the Docket app, built with Prometheus, Grafana, and node-exporter and run with Docker Compose. It sits on the same EC2 host as the app and gives a live view of the server's CPU, memory, disk, and network.

Docket is a .NET task management app I run in production on AWS EC2 (live at [docket.work.gd](https://docket.work.gd)). This repo is the monitoring side of it. The app itself lives in [TaskProject](https://github.com/ShaiganH/TaskProject).

## What's in here

| File | What it does |
|------|--------------|
| `docker-compose.yml` | Defines the three services (Prometheus, Grafana, node-exporter) and the networks they run on |
| `prometheus.yml` | Prometheus scrape config: what to collect, from where, and how often |

## The stack

| Service | Port | Role |
|---------|------|------|
| Prometheus | 9090 | Scrapes metrics on an interval and stores them |
| Grafana | 3000 | Reads from Prometheus and draws the dashboards |
| node-exporter | 9100 | Exposes the host's system metrics for Prometheus to scrape |

The flow is straightforward:

```
node-exporter  ->  Prometheus  ->  Grafana
(host metrics)     (scrape +        (dashboards)
                    store)
```

## How it reaches the app

Prometheus is attached to two Docker networks: its own `monitoring` network, and the app's existing network (`ubuntu_default`), which is declared as external in the compose file. That second connection is what lets Prometheus resolve the app's containers by name, so it can scrape them once they expose metrics.

## Running it

You need Docker and Docker Compose on the host.

```bash
git clone https://github.com/ShaiganH/docket-monitoring.git
cd docket-monitoring
docker compose up -d
```

Confirm all three containers came up:

```bash
docker compose ps
```

## Accessing the services

| Service | URL |
|---------|-----|
| Prometheus | `http://<host-ip>:9090` |
| Grafana | `http://<host-ip>:3000` |
| node-exporter | `http://<host-ip>:9100/metrics` |

On a cloud host you'll need to open ports 9090, 3000, and 9100 in the security group. I scope these to my own IP rather than the public internet, since monitoring isn't something that needs to be exposed.

Grafana's first login uses the admin password set in `docker-compose.yml`. Change it before anything beyond local testing.

## Setting up Grafana

1. Log in and add Prometheus as a data source. Use `http://prometheus:9090` as the URL. The service name works because both containers share the `monitoring` network.
2. Import a dashboard: go to Dashboards, then Import, and enter ID `1860` (Node Exporter Full). Pick the Prometheus data source. You get CPU, memory, disk, and network panels populated straight away.

## Current scope and what's next

Right now the stack monitors host and system metrics through node-exporter, which covers what an ops team usually cares about first.

The Docket app does not expose a Prometheus `/metrics` endpoint yet, so the scrape job pointed at it in `prometheus.yml` stays down until the app publishes its own metrics. Wiring the `prometheus-net` library into the .NET app to surface request rate, latency, and error counts is the obvious next step, and would let the same Grafana instance show application metrics alongside the host ones.

## A note on secrets

Nothing sensitive should be committed here. The Grafana admin password lives in `docker-compose.yml` for convenience while testing. For real use, move it to an environment variable or a `.env` file (already gitignored) and set it to something that isn't the default.
