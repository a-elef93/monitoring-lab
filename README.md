# Monitoring Lab

A hands-on Linux VM monitoring lab using Prometheus, Grafana, and Node Exporter with Docker Compose.

## Architecture

```text
Ubuntu VM metrics
       |
       v
Node Exporter :9100
       |
       | scraped every 15 seconds
       v
Prometheus :9090
       |
       v
Grafana :3000
```

Node Exporter collects Linux host metrics. Prometheus stores the measurements, and Grafana displays them using the community Node Exporter Full dashboard.

## Requirements

- Linux VM with Docker Engine and Docker Compose
- Network access from your browser to the VM
- Available ports 3000 and 9090

## Repository structure

```text
monitoring-lab/
├── compose.yml
├── prometheus.yml
├── dashboards/
│   └── node-exporter-full.json
└── README.md
```

## Start the stack

```bash
git clone git@github.com:a-elef93/monitoring-lab.git
cd monitoring-lab
docker compose config -q
docker compose up -d
docker compose ps
```

Open these addresses, replacing `<VM_IP>` with your VM's IP:

- Prometheus: `http://<VM_IP>:9090`
- Target health: `http://<VM_IP>:9090/targets`
- Grafana: `http://<VM_IP>:3000`

For a fresh Grafana installation, sign in with `admin` / `admin` and set a new password.

## Connect Grafana to Prometheus

1. Open Connections → Data sources.
2. Add a Prometheus data source.
3. Set the server URL to `http://prometheus:9090`.
4. Select Save & test.

The service name `prometheus` resolves inside the Docker Compose network.

In Grafana Explore, run:

```promql
up
```

Both `node` and `prometheus` should return `1`.

## Dashboard

Import `dashboards/node-exporter-full.json` through Grafana's dashboard import screen.

The export uses Grafana's `dashboard.grafana.app/v2` format and requires a Grafana version that supports it. Select your Prometheus data source in the dashboard's Datasource selector after importing.

Alternatively, import dashboard ID **1860** from Grafana.com.

Dashboard source: [Node Exporter Full](https://grafana.com/grafana/dashboards/1860-node-exporter-full/).

The dashboard was created by its upstream authors; this repository contains an exported copy used for the lab.

## Validation performed

- All three containers started successfully.
- Prometheus readiness endpoint returned HTTP 200.
- Both scrape targets reported UP.
- Grafana successfully queried Prometheus.
- A bounded CPU load test produced a visible spike and recovery:

```bash
timeout 60s yes > /dev/null
```

This loads one CPU core for 60 seconds. On the two-core lab VM, total CPU usage rose to approximately 50%.

### Disk capacity incident

The dashboard showed approximately 90% root filesystem usage, confirmed with `df -h /`.

Inspection with `lsblk` and `sudo vgs` revealed approximately 13.47 GiB of unallocated space in the existing LVM volume group. The root logical volume and ext4 filesystem were expanded online.

After expansion:

- Root filesystem capacity: approximately 27 GB
- Available space: approximately 15 GB
- Usage reported by `df`: approximately 45%

Grafana reflected the capacity change. Its percentage can differ slightly from `df` because of filesystem reserved-space calculations.

## Persistence and operation

Named Docker volumes retain Prometheus measurements and Grafana configuration when containers are recreated.

```bash
# View logs
docker compose logs --tail=50

# Stop containers while keeping their volumes
docker compose down

# Start again
docker compose up -d
```

Avoid `docker compose down -v` unless you intend to delete the stored monitoring data and Grafana configuration.

## Limitations

- Local learning environment, not a production deployment.
- Images currently use mutable `latest` tags.
- Published ports use HTTP without TLS.
- Prometheus has no authentication configured.
- Restrict access to a trusted lab network.
- Node Exporter uses host PID access and a read-only host filesystem mount.
- Node Exporter remains on the internal Compose network; port 9100 is not published.
- Network metrics currently reflect the exporter's container network namespace.
- External alert notifications have not been configured.
- The dashboard JSON alone does not provision the data source automatically.

## Alert validation

A Grafana-managed rule, `Node Exporter Down`, evaluates
`up{job="node"} < 1` every 30 seconds with a one-minute pending period.

The failure and recovery test was completed:

1. Stop the exporter: `docker compose stop node-exporter`
2. Confirm the alert transitions to Firing.
3. Restore the exporter: `docker compose start node-exporter`
4. Confirm the alert returns to Normal.

The exported rule is stored in `alerts/node-exporter-down.yaml`.
It references the original Prometheus data source UID; adapt that UID
before provisioning it in another Grafana instance. The current Compose
configuration does not automatically load this file.

Alert state transitions were verified in Grafana. External notifications
are not configured.
