# Intel Full System Monitoring

Real-time GPU, CPU and vLLM inference metrics for Intel XPU hosts — four Docker
containers, one `docker compose up -d`, four pre-built Grafana dashboards.

Pure Docker Compose. No CLI to install, no Python, no agent required.

```bash
git clone https://github.com/intel/full-system-monitoring.git
cd full-system-monitoring
docker compose up -d
```

Then open **http://localhost:3000** — no login required.
Running it on a remote GPU host? See [Remote access](#remote-access-ssh-tunnel).

---

## What gets deployed

| Container | Port | What it collects |
|-----------|------|------------------|
| `xpumd` (Intel XPU Manager v2.0) | 9471 | Per-card GPU: utilization by engine, VRAM, power, energy, clocks, throttle status |
| `node-exporter` | 9100 | Host CPU, RAM, disk, network — and hwmon temperatures |
| `intel-xpu-prometheus` | 9090 | Scrapes everything above + vLLM; 24 h / 512 MB retention |
| `intel-xpu-grafana` | 3000 | Four dashboards, auto-provisioned at startup |

Overhead: **~400 MB RAM, under 3% of one core** — measured idle and under 8-card load.

### Dashboards

| Dashboard | Shows |
|-----------|-------|
| **Quickview** | GPU utilization + VRAM, vLLM TTFT/ITL/KV-cache/tokens-per-sec, CPU + RAM — one screen |
| **Intel XPU Manager v2** | Per-card gauges and time series: utilization, VRAM, power, frequency, temperature; plus engine-group utilization (compute / copy / media / render). Default home dashboard |
| **vLLM Inference** | Requests running/waiting, KV cache usage, tokens/sec, tokens per request, TTFT / end-to-end latency / inter-token latency at p50, p95, p99, and a live Models→Port table |
| **System Resources** | CPU cores, RAM, disk I/O, network |

All refresh every 5 s. Dashboards filter `hw_gpu_type="discrete"`, so a CPU
integrated GPU never appears alongside the discrete cards.

---

## Prerequisites

- Intel GPU with the kernel driver loaded — `/dev/dri/renderD*` must exist
- Docker with the Compose plugin (`docker compose version` works)
- Ports **9471, 9100, 9090, 3000** free (the stack uses host networking, so they
  cannot be remapped)
- ~1.5 GB disk for images, plus up to 512 MB for metrics

Quick check:

```bash
ls /dev/dri/renderD*        # expect at least one render node
docker compose version      # expect v2.x
```

> **Full GPU telemetry needs the `xe` kernel driver.** On hosts using the older
> `i915` driver, xpumd starts but reports fewer fields. Utilization, VRAM and
> power work on both.

---

## Repository layout

Every file here is required. There is nothing else to configure.

```
.
├── docker-compose.yml                     # the 4 containers, volumes, /dev/dri passthrough
├── AGENT.md                               # same deployment, written for an AI coding agent
├── README.md                              # this file
├── xpumd/
│   └── config.yaml                        # xpumd receiver → exports on :9471
├── prometheus/
│   ├── prometheus.yml                     # scrape jobs: xpumd, node, vLLM 8000-8008
│   └── gpu_labels.yml                     # recording rules → gpu:* series, gpu-0…gpu-7 labels
└── grafana/
    ├── provisioning/
    │   ├── datasources/prometheus.yml     # auto-wires the Prometheus datasource
    │   └── dashboards/dashboards.yml      # auto-loads the dashboards below
    └── dashboards/
        ├── quick-view.json
        ├── xpumd-single-node.json
        ├── vllm-inference.json
        └── system-resources.json
```

Volume paths in `docker-compose.yml` are **relative to the compose file**, so run
`docker compose` from the repository root.

---

## vLLM monitoring

### Ports enabled by default

Prometheus scrapes these **nine ports** for vLLM metrics, every 5 seconds:

```
localhost:8000   localhost:8001   localhost:8002
localhost:8003   localhost:8004   localhost:8005
localhost:8006   localhost:8007   localhost:8008
```

Defined in [`prometheus/prometheus.yml`](prometheus/prometheus.yml) under the
`vllm` job. **Start vLLM on any port in 8000–8008 and its metrics appear within
about 5 seconds — no configuration change, no restart.**

Ports with nothing listening are simply reported as down. Each costs well under a
millisecond per scrape, which is why the range is pre-enabled rather than
discovered.

Example — serve a model on port 8001 and it is monitored immediately:

```bash
docker run -d --net=host --device /dev/dri \
  -e ZE_AFFINITY_MASK=0 -e VLLM_TARGET_DEVICE=xpu \
  intel/vllm:0.19.0-xpu \
  vllm serve Qwen/Qwen3-1.7B --port 8001 --dtype bfloat16 --enforce-eager
```

### Adding a vLLM port outside 8000–8008

Say your server runs on **8101**. Three steps:

**1. Edit `prometheus/prometheus.yml`** — add the target to the `vllm` job:

```yaml
  - job_name: "vllm"
    scrape_interval: 5s
    scrape_timeout: 4s
    metrics_path: /metrics
    static_configs:
      - targets:
          - "localhost:8000"
          # ... 8001 through 8008 ...
          - "localhost:8008"
          - "localhost:8101"       # ← added
```

**2. Restart Prometheus so it picks up the new file:**

```bash
docker compose restart prometheus
```

Only **Prometheus** reads `prometheus.yml`. You do **not** need to restart
xpumd, node-exporter or Grafana, and you do not need `docker compose down`.
Metrics history is kept — it lives in a volume, not in the container.

**3. Confirm the target is up:**

```bash
curl -s 'http://localhost:9090/api/v1/targets?state=active' | grep 8101
```

Or in Grafana: the **vLLM Inference** dashboard's *Models → Port* table lists
every port currently reporting, with its model name.

### Which service to restart after which change

| File you changed | Apply with |
|------------------|-----------|
| `prometheus/prometheus.yml` (vLLM ports, scrape config) | `docker compose restart prometheus` |
| `prometheus/gpu_labels.yml` (recording rules) | `docker compose restart prometheus` — it owns both files |
| `grafana/dashboards/*.json` | nothing — Grafana re-reads from disk every 10 s |
| `grafana/provisioning/*` | `docker compose restart grafana` |
| `xpumd/config.yaml` | `docker compose restart xpumd` |
| `docker-compose.yml` | `docker compose up -d` (recreates only what changed) |

### Removing a port

Delete its line from the `vllm` job and run `docker compose restart prometheus`.
A removed port left in the config shows as a permanently down target — harmless,
but noisy.

---

## Remote access (SSH tunnel)

The stack binds to localhost on the GPU host, so nothing is exposed to the
network. To view the dashboards from your laptop, forward the two ports over SSH.

**No IP addresses needed** — add a host entry to `~/.ssh/config` on your laptop
once, then refer to it by name:

```
Host gpu-host
    HostName <hostname-or-address-of-the-gpu-machine>
    User <your-username>

# If the GPU host sits behind a bastion / jump host:
Host gpu-host
    HostName <gpu-host-name>
    User <your-username>
    ProxyJump <your-username>@<jump-host-name>
```

Then, from your laptop:

```bash
ssh -N -L 3000:localhost:3000 -L 9090:localhost:9090 gpu-host
```

Leave it running and open **http://localhost:3000** in your browser.

- `-N` means "no remote command" — it only forwards ports.
- `3000` is Grafana, `9090` is Prometheus (useful for querying metrics directly).
- `localhost` in the forward spec is resolved **on the GPU host**, so the tunnel
  reaches the containers without them ever listening on a public interface.

To keep the tunnel alive through idle periods, add
`-o ServerAliveInterval=60`. To run it in the background, add `-f`.

---

## Verify the deployment

```bash
# All four containers up?
docker compose ps

# Each exporter answering?
curl -s http://localhost:9471/metrics | grep -c hw_gpu      # GPU metrics present
curl -s http://localhost:9100/metrics | grep -c node_cpu    # host metrics present
curl -s http://localhost:9090/-/healthy                     # Prometheus
curl -s http://localhost:3000/api/health                     # Grafana

# How many GPUs did xpumd discover?
curl -s http://localhost:9471/metrics | grep -c 'hw_gpu_info.*discrete'

# Every scrape target and its health
curl -s 'http://localhost:9090/api/v1/targets?state=active'
```

Expect the `xpumd` and `node` targets `up`. The `vllm` targets stay `down` until a
model server is running — that is normal.

---

## Teardown

```bash
docker compose down        # stop and remove the containers, keep metrics history
docker compose down -v     # also delete the Prometheus + Grafana volumes
```

`-v` **permanently deletes stored metrics history.** Neither command touches vLLM
or any other workload on the host.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `no such file or directory` on a volume mount | Run `docker compose` from the repository root — volume paths are relative to `docker-compose.yml` |
| `xpumd` exits immediately | Check `/dev/dri` exists, then `docker logs xpumd` for Level Zero or capability errors |
| GPU panels empty while xpumd runs | `curl localhost:9471/metrics \| grep hw_gpu` — if empty, the host driver isn't exposing telemetry (see the `xe` driver note) |
| Temperature panels empty | Temperature comes from xpumd where the firmware exposes it, otherwise from node-exporter's hwmon collector. Verify either: `curl -s localhost:9471/metrics \| grep -c hw_temperature` and `curl -s localhost:9100/metrics \| grep -c node_hwmon_temp_celsius` |
| Memory-bandwidth or temperature panel empty | These depend on GPU firmware and kernel driver support. Some hosts export them, some do not. Check the source: `curl -s localhost:9471/metrics \| grep -cE "bandwidth\|temperature"` |
| "No data" right after startup | Wait 10–15 s for the first scrapes |
| vLLM panels empty | vLLM must be serving on a scraped port. Confirm with `curl localhost:<port>/metrics \| grep vllm:` and check the port is in the `vllm` job |
| A vLLM panel is empty but others have data | Rate and percentile panels need traffic within the last minute; cumulative counters persist |
| Added a vLLM port but it never appears | Run `docker compose restart prometheus`. Check the container has your edit: `docker exec intel-xpu-prometheus grep <port> /etc/prometheus/prometheus.yml` |
| Port already in use | The stack uses host networking and cannot remap. Free the port or stop the conflicting service |
| Can't reach Grafana from your laptop | Use the SSH tunnel above; confirm with `curl localhost:3000/api/health` on the GPU host first |

---

## Validated on

8× and 2× Intel Arc Pro B70 (32 GB VRAM, `xe` driver) · Ubuntu 24.04 / 25.10 ·
Docker 28.x / 29.x · XPU Manager v2.0.0 · vLLM 0.17.0-xpu / 0.19.0-xpu.

For AI coding agents deploying this stack, see [AGENT.md](AGENT.md).
