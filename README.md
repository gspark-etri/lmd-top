# lmd-top

> **A terminal observability & operations tool for [llm-d](https://llm-d.ai) clusters.**
> `k9s`-style navigation + `all-smi`-style accelerator cards + **first-class understanding of the llm-d EPP routing architecture** — in one screen.

`lmd-top` is a TUI that **correlates all four layers** of an llm-d serving stack —
`Gateway → EPP (Endpoint Picker) → Model Server → Infrastructure` — for
**heterogeneous accelerator** fleets (NVIDIA GPU · Rebellions RBLN · Furiosa RNGD · host CPU).
It reads your existing Prometheus + Kubernetes; it owns no data of its own.

```
◐ lmd-top  llm-d · 5 nodes  ⌂ gw 10.254.184.233 ●   · updated 1s ago
GPU 0 RBLN 4 RNGD 8 · 8 busy │ vram 210/544GB │ models 2/6 │ ⚡390W   ⚠1 alert (A)
 0:Overview  1:Accel  2:Models  3:EPP  4:Topo  5:Pods  6:Perf  7:Launch  8:Events  9:Nodes
╭ Cluster ─────────────────────────────────────────────────────────────────────╮
│ Σ RBLN×4 RNGD×8 · util 41% · VRAM 210/544GB (39%) · 390W · models 2/6 · TTFT … │
│ VRAM  ████████░░░░░░░░░░░░░░░░  210/544GB used                                  │
│ RBLN  ● ● ● ●   RNGD  ● ● ○ ● ⚠ ● ● ●                                          │
╰────────────────────────────────────────────────────────────────────────────────╯
╭ Accelerators (by kind / node) ───────────────────────────────────────────────╮
│ ● RBLN×4 @node-a1        ██████░░░░  58%  56/68 GB   ▁▂▄▅▆▅▄▃                   │
│ ⚠ RNGD×8 @node-b2        ███░░░░░░░  31%  154/476GB  ▂▃▃▄▃▂▁▁                   │
╰────────────────────────────────────────────────────────────────────────────────╯
 ↑↓ sel  ⏎ detail  / filter  l logs  s scale  A alerts  t theme  z zoom  ? help  q quit
```

---

## Why lmd-top?

The llm-d ecosystem has **no live, operator-facing terminal tool** — only Grafana web
dashboards, benchmark harnesses, and `helm`/`kubectl`. `lmd-top` fills that gap, and
uniquely **observes and explains EPP routing decisions**.

| | Sees | llm-d / EPP awareness | Accelerators | K8s actions | Terminal |
|---|---|---|---|---|---|
| `k9s` | K8s objects | ❌ | ❌ | ✅ | ✅ |
| `all-smi` | Infra (accelerators) only | ❌ | ✅✅ | ❌ | ✅ |
| `llmtop` | single-host psutil | ❌ | ⚠️ | ❌ | ✅ |
| Grafana | all-layer metrics | ⚠️ | ✅ | ❌ | ❌ web |
| **lmd-top** | **4-layer correlation** | ✅✅ EPP `Filter→Score→Pick` | ✅ | ✅ | ✅ |

---

## Highlights

- **Four layers, one screen.** Gateway, EPP/InferencePool, model servers, and hardware
  are correlated so you can answer *"which model runs where, how requests are routed,
  and how load is distributed."*
- **Heterogeneous accelerators, unified.** NVIDIA GPU (`DCGM_*`), Rebellions RBLN
  (`RBLN_DEVICE_STATUS:*`), and Furiosa RNGD (`furiosa_npu_*`) are shown side by side —
  vendor identity by color, health by glyph.
- **EPP-aware.** Introspects the EPP `ConfigMap` (active scorers, weights, picker),
  visualizes routing decisions and per-pod queues, and **auto-diagnoses whether an
  HTTPRoute goes through the InferencePool (EPP) or bypasses it** (a common misconfig
  that leaves EPP metrics empty).
- **all-smi-style visuals.** Per-device gauges, inline sparklines, btop/nvtop-style
  **area-fill timelines**, an at-a-glance **LED device grid**, and a **stacked VRAM
  composition bar** (by vendor).
- **Active alerting.** Threshold/health conditions (throttle, not-alive, hot, node
  NotReady/cordon/pressure, pod restarts/Failed) trigger a summary-bar flash + a toast,
  and are collected into an **alert history** (`A`).
- **Operator ergonomics.** Row selection with scrollbars & position counters, substring
  filtering, sorting, drill-down detail, pod/model **logs overlay**, `scale` action,
  a **data-freshness clock**, responsive tabs, focus highlight on the active pane, a
  **zoom/focus** mode, and three themes (default / high-contrast / **colorblind-safe**).
- **Pure Rust, single static binary.** No TLS/heavy HTTP crates: Prometheus is queried
  over raw `tokio` HTTP/1.0, Kubernetes via `kubectl`. Nothing to install on GPU nodes.

---

## Views

Ten correlated views — switch with the top number keys (`0`–`9`) or `Tab`:

| # | View | What it shows |
|---|---|---|
| **0** | **Overview** | Cluster Σ summary · **LED device grid** · **VRAM composition bar** · accelerators by kind/node · EPP path & pools · models table · **one-line cross-layer diagnosis** |
| 1 | **Accel** | Per-device rows: util bar / VRAM / temp / power + inline util trend. GPU · RBLN · RNGD unified. `⏎` → full util/VRAM timeline |
| 2 | **Models** | Per-model accelerator/node · ready · running/waiting · KV% · tok/s · routing path · status |
| 3 | **EPP** | Active scorers & weights (ConfigMap introspect) + picker + InferencePool endpoints + **request distribution** (routing decisions) |
| 4 | **Topo** | **Whole topology at a glance** — Gateway → HTTPRoute → backend (model status/accelerator/node) + InferencePool/EPP/**SLO** (InferenceObjective) + autoscalers + **EPP-bypass diagnosis** |
| 5 | **Pods** | `llm-serving` pod status (ready / phase / node / restarts) |
| 6 | **Perf** | **For EPP policy tuning** — system timelines + per-model p95 latency broken down **QUEUE → PREFILL(P) → DECODE(D) → TPOT → E2E** + preemptions, tok/s, per-pod queue distribution. *Lists every launched model (idle ones show `–`).* |
| 7 | **Launch** | Model **catalog** × live accelerator inventory → placement feasibility (`✓` ready / `⚙` needs-artifact / `✗` no-capacity). Read-only; catalog = `catalog/models.yaml` |
| 8 | **Events** | Unified k8s + llm-d events (newest first), warnings highlighted |
| 9 | **Nodes** | Node health / placement — status · kubelet · CPU · load · memory · accelerators per node |

> **Topo** answers *where does each model run, how is it routed, and does traffic actually
> pass through the EPP?* **Perf** gathers the latency/token/distribution signals you need
> to design EPP scorer policy (populated once EPP-path traffic + vLLM metrics are present).

---

## Install

### Prerequisites

- **Rust** toolchain (`rustup`) + a **C linker** (`gcc`/`cc`), required for Rust linking:
  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  sudo apt-get install -y build-essential      # gcc/cc linker
  ```
- Runtime: `kubectl` (with kubeconfig access) + reachability to Prometheus.
  **No SSH to accelerator nodes is needed** — everything goes through Prometheus.

### Build & install

```bash
git clone https://github.com/gspark-etri/lmd-top.git && cd lmd-top
cargo install --path .        # → ~/.cargo/bin/lmd-top
# or just build:
cargo build --release         # → target/release/lmd-top
```

---

## Usage

```bash
lmd-top                       # launch the TUI (permission mode: observe)
lmd-top --mode admin          # allow scale/rollout actions (see Permission modes)
lmd-top --snapshot            # collect once, print text (headless / debug)   [alias: -s]
lmd-top --render              # render every view to text via TestBackend (CI / verification)

# point at a different cluster / namespace
LMD_PROM=10.0.0.5:30090 LMD_NS=my-ns lmd-top
```

### Permission modes

Mutating actions are gated by a startup mode (`--mode observe|debug|admin|danger`,
default `observe`), shown as a badge in the header. This prevents fat-finger accidents
in a shared cluster.

| Mode | Allows | Gated keys |
|---|---|---|
| **observe** *(default)* | view only | — |
| **debug** | + logs / dry-run | `l` |
| **admin** | + scale / rollout | `s` |
| **danger** | + delete / force | *(future)* |

Admin+ mutating actions (e.g. `s` scale) ask for a `y`/`n` confirmation before applying.

### Keybindings

| Key | Action |
|---|---|
| `0`–`9` | switch view |
| `Tab` | next view |
| `↑`/`↓` (or `k`/`j`), mouse scroll | select row |
| `Enter` | **drill-down detail** (accelerator / model / pod / node) |
| `←`/`→` | previous / next item |
| `/` | **filter** (substring) — type, then Enter/Esc |
| `o` | cycle **sort** (Accel: util/temp/mem/name · Models: name/status/ready · Pods: name/phase/restarts) |
| `l` | **logs** overlay for selected pod/model (scroll, `r` refresh) |
| `s` | **scale** selected model (desired 0↔1 toggle) |
| `A` | **alert history** overlay (threshold / health events) |
| `t` | cycle **theme** (default / high-contrast / colorblind-safe) |
| `g` | open **Grafana** dashboard in browser |
| `z` | **zoom / focus** (hide header + tabs, maximize body) |
| `Space` | **pause** updates (freeze data for reading) |
| `Esc` | **back only** (close detail / filter / zoom — does *not* quit) |
| `?` | help / color legend overlay |
| `q` | quit |

### Semantic colors & glyphs

Color encodes **severity** or **identity**; state is carried by a separate **glyph**
(so the two never collide, and it stays legible in the colorblind theme):

| Element | Meaning |
|---|---|
| 🟢 green | healthy / low load / serving |
| 🟡 yellow | warning / mid load / pending / throttling |
| 🔴 red | critical / high load / error / device down / **active alert** |
| 🔵 cyan | accent / headers / interactive values |
| ⚫ dark gray | idle / absent (`–`) / labels |
| vendor color | GPU = green · **RBLN = magenta** · RNGD = cyan |
| glyphs | `●` up/healthy · `○` idle/scaled-0 · `◐` pending · `⚠` throttle · `⊘` cordoned · `✗` down |
| thresholds | util `>85`🔴 `>60`🟡 · mem `>90`🔴 `>70`🟡 · temp `>80`🔴 `>60`🟡 |

Metrics that aren't present yet (workload off) render as `–`/`offline` and fill in
automatically once the workload comes up. The header shows a **freshness clock**
(`updated Ns ago`, turning yellow when data goes stale).

### Configuration (optional) — `~/.config/lmd-top/lmd-top.yaml`

Customize column visibility/order:

```yaml
columns:
  models: [name, accel, status, tps]   # only these columns, in this order (default: all)
```

### Environment variables

| Variable | Default | Meaning |
|---|---|---|
| `LMD_PROM` | `10.254.184.105:30090` | Prometheus `host:port` (plain HTTP) |
| `LMD_NS` | `llm-serving` | target namespace |
| `LMD_GRAFANA` | `http://10.254.184.105:30300` | Grafana base URL opened by `g` |
| `LMD_W` / `LMD_H` | `100` / `26` | render size for `--render` |

---

## Data path

lmd-top **owns no data** — it reads your existing stack and correlates it.

| Layer | Source | Example metrics / resources |
|---|---|---|
| Infra (accelerators) | Prometheus | `furiosa_npu_*`, `RBLN_DEVICE_STATUS:*`, `DCGM_FI_DEV_*`, `node_*` |
| Model server | Prometheus | `vllm:num_requests_running/waiting`, `vllm:*_latency_seconds_bucket`, `vllm:generation_tokens_total`, `vllm:kv_cache_usage_perc` |
| EPP / Pool | Prometheus + ConfigMap | `inference_pool_*`, `inference_extension_*`, `llmd-router-epp` cm |
| Topology / status / actions | `kubectl` | Deployment, Pod, HTTPRoute, Gateway, InferencePool, InferenceObjective |

Data flows in on two tiers: a **~1 s fast tier** (accelerators + nodes) and a
**~3 s full snapshot** (everything else). Per-model perf joins vLLM metrics by the
`service` label (= Deployment name), the same key the Models view uses.

> **To see every metric**, some exporters/ServiceMonitors may be required (RBLN and EPP;
> Furiosa is on by default). See the companion setup repo (`llm-d-setup`):
> `manifests/epp-servicemonitor.yaml`, `manifests/rbln-metrics-servicemonitor.yaml`.

---

## Architecture

```
 kubectl ─┐                                              ┌─ Overview ─┐
 Prom    ─┤→ collectors → Snapshot (metric bus) → panels ┤  Accel …   │ → ratatui → terminal
 (cm)    ─┘   (data IN, one-way)          (render OUT)    └─ Nodes ────┘
```

- **Pure Rust, no C-library deps.** Prometheus is queried directly over `tokio` TCP
  (HTTP/1.0, no TLS); Kubernetes via shelling out to `kubectl`. Result: a single static
  binary with no heavy TLS/HTTP crates.
- `collectors` **only write** to the `Snapshot`; `panels` **only read** it. New data =
  add a collector; new screen = add a panel.
- Dependencies: `ratatui`, `tokio`, `serde`/`serde_json`/`serde_yaml`, `anyhow`,
  `unicode-width`.

```
src/
  main.rs      entry point · event loop · --snapshot / --render
  collect.rs   Snapshot types + prom/kube collection
  prom.rs      pure-tokio HTTP/1.0 Prometheus client
  kube.rs      kubectl shelling + scale action
  app.rs       UI state (view / selection / history / alerts)
  ui.rs        ratatui rendering (header / tabs / views / footer)
```

---

## Roadmap

- **Phase 1 — Monitor** ✅ *(current)* — 10 correlated views, active alerting, logs, scale action.
- **Phase 2 — Control plane** *(next)* — permission modes (observe/debug/admin/danger), a
  machine-readable **agent JSON** state (`--snapshot --json`), and safe actions (endpoint
  drain / traffic & policy weight / rollout) with dry-run → confirm → audit.
- **Phase 3 — LLM-native depth** *(infra-gated; skeleton-first)* — PD-aware dashboard, EPP
  decision debugger with per-endpoint score table, KV/prefix cache locality, SLO/goodput
  diagnosis. Fills in once traffic goes through the EPP with vLLM/tracing metrics.
- **ModelService-native** — shift Models/Launch from raw Deployments to llm-d ModelService
  CRDs, enabling real deploy from the catalog.

See `ROADMAP.md` for the detailed plan and `CHANGELOG.md` for release history.

## Status

Verified against a live cluster (5 nodes, 12 accelerators, EPP/routes/models live).
Experimental (0.x) — interfaces may change.
