# DGX Spark Prometheus Exporter

A [Prometheus](https://prometheus.io) metrics exporter for [NVIDIA DGX Spark](https://www.nvidia.com/en-us/products/workstations/dgx-spark/) systems.
It exposes hardware and system metrics via an HTTP endpoint that Prometheus can scrape.

![screenshot](https://github.com/user-attachments/assets/75137a87-da9d-4482-b908-b96e4d8d4489)


## Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `cpu_usage_percent` | Gauge | CPU usage percentage (0-100) |
| `cpu_temperature_celsius` | Gauge | CPU temperature in °C |
| `cpu_frequency_mhz` | Gauge | Average CPU core frequency in MHz |
| `gpu_utilization_percent` | Gauge | GPU (GB10) utilization percentage |
| `gpu_temperature_celsius` | Gauge | GPU temperature in °C |
| `gpu_frequency_mhz` | Gauge | GPU graphics clock in MHz |
| `gpu_power_watts` | Gauge | GPU power consumption in Watts |
| `memory_total_bytes` | Gauge | Total RAM in bytes |
| `memory_used_bytes` | Gauge | Used RAM in bytes |
| `diskio_reads_completed_total` | Counter | Disk read operations (label: `device`) |
| `diskio_writes_completed_total` | Counter | Disk write operations (label: `device`) |
| `storage_used_percent` | Gauge | Used capacity of `/` in percent |
| `network_receive_bytes_total` | Counter | Bytes received (label: `interface`) |
| `network_transmit_bytes_total` | Counter | Bytes transmitted (label: `interface`) |
| `network_receive_packets_total` | Counter | Packets received (label: `interface`) |
| `network_transmit_packets_total` | Counter | Packets transmitted (label: `interface`) |


### Monitored Network Interfaces

Only the following interfaces are monitored (when they are up):

- `enP7s7`
- `enp1s0f1np1`
- `enP2p1s0f1np1`
- `enp1s0f0np0`
- `enP2p1s0f0np0`
- `wlP9s9`


## Building and installing

Run on the DGX Spark, with the Go language installed, in the root directory of this repository:

```bash
go build .
```

```bash
sudo cp ./dgx-spark-prometheus /usr/local/bin/
sudo cp ./dgx-spark-prometheus.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now dgx-spark-prometheus
sudo systemctl start dgx-spark-prometheus
```

### How to transfer build to another DGX Spark

On the originating DGX Spark `spark1`:

```
tladmin@spark1:~/dgx-spark-prometheus$ scp dgx-spark-prometheus dgx-spark-prometheus.service spark2:~
```

On the receiveing DGX Spark `spark2`:

```
tladmin@spark2:~ sudo mv ./dgx-spark-prometheus /usr/local/bin/
tladmin@spark2:~ sudo mv ./dgx-spark-prometheus.service /etc/systemd/system/
tladmin@spark2:~ sudo systemctl daemon-reload
tladmin@spark2:~ sudo systemctl enable --now dgx-spark-prometheus
tladmin@spark2:~ sudo systemctl start dgx-spark-prometheus
```

## Listen address

The binary and bundled systemd service default to `127.0.0.1:9835`.
Metrics are available at `http://127.0.0.1:9835/metrics` from the same host.
An nginx instance on that host can proxy this endpoint.

To allow direct access from other machines, listen on all IPv4 interfaces:

```bash
dgx-spark-prometheus -listen 0.0.0.0:9835
```

For the systemd service, run `sudo systemctl edit dgx-spark-prometheus`
and add:

```ini
[Service]
ExecStart=
ExecStart=/usr/local/bin/dgx-spark-prometheus -listen 0.0.0.0:9835
```

Then apply the change:

```bash
sudo systemctl daemon-reload
sudo systemctl restart dgx-spark-prometheus
```

The exporter does not provide authentication or TLS. When enabling remote
access, restrict port 9835 to trusted scrapers using your firewall.

## Service permissions and GPU timeout

The bundled service currently runs as root for compatibility. The collectors
do not perform privileged configuration changes: they read system statistics
and run a read-only `nvidia-smi` query. Root is not inherently required, but
access to GPU devices and sysfs metrics depends on the host's permissions.
A dedicated service account can be used after verifying those reads and the
GPU query work under that account; grant only the device access it needs.

Each `nvidia-smi` query has a five-second timeout. On timeout, the exporter
kills the command, logs the failure, and omits GPU metrics for that scrape.
Waiting for inherited output pipes is bounded by an additional one second.
Other collectors continue to report their metrics.

## Prometheus configuration

For Prometheus running on the same host (and in the same network namespace):

```
scrape_configs:
  - job_name: 'dgx_spark'
    scrape_interval: 15s
    scrape_timeout: 10s
    static_configs:
      - targets: ['127.0.0.1:9835']
    metrics_path: /metrics
    scheme: http
```

Use `scrape_interval: 30s` for less frequent collection. For remote scraping,
enable a reachable listen address as described above and replace the target
with the Spark's hostname or IP, for example `spark1:9835`.

## Data Sources

| Metric | Source |
|--------|--------|
| CPU usage | `/proc/stat` (delta between scrapes) |
| CPU temperature | `/sys/class/thermal/thermal_zone*/` |
| CPU frequency | `/sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq` |
| GPU metrics | `nvidia-smi --query-gpu=...` |
| Memory | `/proc/meminfo` |
| Disk I/O | `/proc/diskstats` |
| Disk capacity | `statfs("/")` |
| Network I/O | `/sys/class/net/<iface>/statistics/` |
