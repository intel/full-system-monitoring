# AGENT.md — Deploy Intel XPU Full-Stack Monitoring

> **For AI coding agents** (Claude Code, Cursor, Copilot, Gemini, …).
> This file is the deployment playbook. Follow it top to bottom; every step is a
> real command to run on the GPU host.
>
> This stack is **pure Docker Compose**. There is no CLI to install, no Python
> dependency, no tarball to fetch. If you find yourself installing a tool, stop —
> you are off the path.

---

## What you are deploying

Four containers on one Intel XPU host:

| Container | Port | Role |
|-----------|------|------|
| `xpumd` (Intel XPU Manager v2.0) | 9471 | Per-card GPU metrics via Level Zero |
| `node-exporter` | 9100 | Host CPU, RAM, disk, network, hwmon temperatures |
| `intel-xpu-prometheus` | 9090 | Scrapes all sources; 24 h / 512 MB retention |
| `intel-xpu-grafana` | 3000 | Four auto-provisioned dashboards |

Overhead: ~400 MB RAM, under 3% of one core.

---

## Step 1 — Preflight

Verify, do not assume. Every check must pass before deploying.

```bash
ls /dev/dri/renderD*        # Intel GPU render nodes must exist
docker info                 # daemon reachable
docker compose version      # Compose plugin present
```

Confirm the four host ports are free — the stack uses host networking and cannot
remap them:

```bash
for p in 9471 9100 9090 3000; do
  (ss -ltn 2>/dev/null || netstat -ltn) | grep -q ":$p " \
    && echo "port $p IN USE" || echo "port $p free"
done
```

If a port is held by a previous run of this same stack (`xpumd`,
`node-exporter`, `intel-xpu-prometheus`, `intel-xpu-grafana`), Compose will
recreate that container — fine. If held by anything else, resolve it first.

Optional context, useful to report back:

```bash
df -h /var/lib/docker | tail -1                       # need ~1.5 GB for images
ls /sys/module/xe /sys/module/i915 -d 2>/dev/null     # which GPU driver is loaded
```

---

## Step 2 — Get the repository onto the host

```bash
# Option A — clone on the host (preferred)
git clone https://github.com/intel/xe-full-stack-monitoring.git
cd xe-full-stack-monitoring

# Option B — no git/network on the host: copy from your laptop
#   scp -r ./xe-full-stack-monitoring <user>@<gpu-host>:~/
#   add  -J <user>@<jump-host>  if the GPU box sits behind a bastion
#   then on the host:  cd ~/xe-full-stack-monitoring
```

The repository is self-contained: compose file, Prometheus config and recording
rules, xpumd config, and all four dashboards.

**Volume paths in `docker-compose.yml` are relative to that file**, so all
`docker compose` commands must run from the repository root.

---

## Step 3 — Deploy

This step **mutates host state**: it pulls roughly 1.5 GB of images and creates
four containers. **Tell the user what will happen and get confirmation before
running it.**

```bash
docker compose up -d
```

Give it 10–15 seconds for the first scrapes to land, then verify.

---

## Step 4 — Verify

```bash
docker compose ps                                            # four containers Up

curl -s http://localhost:9471/metrics | grep -c hw_gpu       # >0 : GPU metrics flowing
curl -s http://localhost:9100/metrics | grep -c node_cpu     # >0 : host metrics flowing
curl -s http://localhost:9090/-/healthy                       # Prometheus healthy
curl -s http://localhost:3000/api/health                      # Grafana healthy

# GPUs discovered
curl -s http://localhost:9471/metrics | grep -c 'hw_gpu_info.*discrete'

# All scrape targets with health
curl -s 'http://localhost:9090/api/v1/targets?state=active'
```

Expect `xpumd` and `node` targets **up**. The `vllm` targets are **down** until a
model server runs — that is correct, not a failure. Do not report it as one.

> When probing a large metrics endpoint, avoid `curl … | grep -q` under
> `set -o pipefail`: `grep -q` exits at the first match, which SIGPIPEs `curl` and
> yields exit 141. Use `grep -c`, or capture the body first and match in-shell.

---

## Step 5 — Report back to the user

Give them:

- **Grafana** — http://localhost:3000 (no login required; the form is disabled and anonymous access is on)
- **Number of GPUs discovered** by xpumd
- **Which targets are up**, and that vLLM targets stay down until a server starts
- **The SSH tunnel command**, if they are not sitting on the GPU host:

```bash
# From the user's laptop. Prefer a ~/.ssh/config Host entry over a raw address.
ssh -N -L 3000:localhost:3000 -L 9090:localhost:9090 <ssh-host-alias>
```

`localhost` in the forward spec resolves **on the GPU host**, so the tunnel
reaches the containers without exposing them to the network. See README.md →
*Remote access* for the `~/.ssh/config` pattern, including `ProxyJump`.

---

## Step 6 — vLLM ports

Prometheus already scrapes **8000–8008**. Start vLLM on any of those and metrics
appear within ~5 s — no config change, no restart:

```bash
docker run -d --net=host --device /dev/dri \
  -e ZE_AFFINITY_MASK=0 -e VLLM_TARGET_DEVICE=xpu \
  intel/vllm:0.19.0-xpu \
  vllm serve Qwen/Qwen3-1.7B --port 8001 --dtype bfloat16 --enforce-eager
```

### To monitor a port outside 8000–8008

Edit the `vllm` job in `prometheus/prometheus.yml`, adding one target line:

```yaml
      - targets:
          - "localhost:8000"
          # ... through 8008 ...
          - "localhost:8101"       # ← added
```

Apply it by **restarting Prometheus**:

```bash
docker compose restart prometheus
```

**Only Prometheus reads this file.** Do not restart xpumd, node-exporter or
Grafana, and never `docker compose down` for a scrape-config change. Stored
history survives a restart — it is in a volume, not the container.

> **Do not rely on `curl -X POST localhost:9090/-/reload`.** It returns HTTP 200
> and appears to succeed, but after an edit it typically reloads the *old*
> config. `prometheus.yml` is a **single-file bind mount**, tracked by inode;
> `sed -i`, `vim` and most editors replace the file rather than writing in place,
> giving it a new inode while the container still holds the old one. The edit is
> then invisible inside the container. Always restart, and verify the container
> actually sees the change:
>
> ```bash
> docker exec intel-xpu-prometheus grep 8101 /etc/prometheus/prometheus.yml
> ```

Confirm:

```bash
curl -s 'http://localhost:9090/api/v1/targets?state=active' | grep 8101
```

To stop monitoring a port, delete its line and restart Prometheus again. A port
left in the config with nothing behind it shows as a permanently down target.

---

## Which file drives which service

Change the file, apply with the command — nothing more.

| File | Owned by | Apply with |
|------|----------|-----------|
| `prometheus/prometheus.yml` | Prometheus | `docker compose restart prometheus` |
| `prometheus/gpu_labels.yml` | Prometheus | `docker compose restart prometheus` |
| `grafana/dashboards/*.json` | Grafana | nothing — re-read from disk every 10 s |
| `grafana/provisioning/*` | Grafana | `docker compose restart grafana` |
| `xpumd/config.yaml` | xpumd | `docker compose restart xpumd` |
| `docker-compose.yml` | Compose | `docker compose up -d` |

---

## Teardown

```bash
docker compose down        # remove containers, keep metrics history
docker compose down -v     # also delete volumes — DESTROYS metrics history
```

`-v` is destructive: **confirm with the user before running it.** Neither command
touches vLLM or other workloads.

---

## Guardrails

- **Never** invent a CLI, wrapper script or installer for this stack. Deployment
  is `docker compose up -d` — nothing else.
- **No secrets belong in this repository.** It needs no Hugging Face token, no API
  key, and **no Grafana password** — the login form is disabled and anonymous
  access is on, so there is no credential at all. Never add one to a tracked
  file; if a user wants authentication, point them at a `.env` or secret store.
- **Confirm before mutating**: image pulls, container creation, restarts, and
  anything with `down -v`.
- **Report honestly.** If a target is down or a panel is empty, say so and say
  why. An empty `vllm` target with no server running is expected; an empty GPU
  panel is a real problem.
- **Do not add** load-test harnesses, benchmark scripts or extra tooling to this
  repository. It is a monitoring deployment, and its minimalism is the point.

---

## Known gaps (report these accurately; do not paper over them)

Telemetry breadth depends on **GPU firmware and kernel driver**, not on this repo's
config. Verified: some hosts export temperature and memory bandwidth from xpumd, others
do not. Report what you observe; do not "fix" the config to chase a missing field.

| Metric | Notes |
|--------|-------|
| GPU / memory temperature | From xpumd where firmware exposes it; `gpu_labels.yml` falls back to node-exporter hwmon otherwise. Check both: `curl -s localhost:9471/metrics \| grep -c hw_temperature` · `curl -s localhost:9100/metrics \| grep -c node_hwmon_temp_celsius` |
| Memory bandwidth utilization | Present on some hosts, absent on others. If absent the panel reads empty — that is a platform limitation, not a misconfiguration |
| Full telemetry breadth | Best with the `xe` kernel driver. Under `i915`, xpumd runs but reports fewer fields |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `no such file or directory` on a volume mount | Run `docker compose` from the repository root |
| `xpumd` exits immediately | `docker logs xpumd`; check `/dev/dri` and that the container keeps `SYS_ADMIN` |
| GPU metrics empty | `curl -s localhost:9471/metrics \| grep -c hw_gpu`; if 0, the host driver isn't exposing telemetry |
| Dashboard "No data" | Check targets: `curl -s 'localhost:9090/api/v1/targets?state=active'` |
| vLLM metrics missing | Server must be on a scraped port; `curl localhost:<port>/metrics \| grep vllm:` |
| Port conflict on 9471/9100/9090/3000 | Host networking — free the port; it cannot be remapped |
| GPU util > 100% or power in "TW" | Clamped by recording rules in `gpu_labels.yml`; stale samples age out within the 24 h retention window |

---

## Validated on

8× and 2× Intel Arc Pro B70 (32 GB VRAM, `xe` driver) · Ubuntu 24.04 / 25.10 ·
Docker 28.x / 29.x · XPU Manager v2.0.0 · vLLM 0.17.0-xpu / 0.19.0-xpu.

Human-facing documentation: [README.md](README.md).
