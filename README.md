# lmd-top

> **A terminal observability & operations tool for [llm-d](https://llm-d.ai) clusters.**
> The whole serving stack — Gateway, EPP routing, model servers, and heterogeneous accelerators — on one screen, in one static binary.

**English** · [한국어](README.ko.md)

![Rust](https://img.shields.io/badge/Rust-000?logo=rust&logoColor=white)
![single static binary](https://img.shields.io/badge/single%20static%20binary-no%20C%20deps-success)
![for llm-d](https://img.shields.io/badge/for-llm--d-8839ef)
![views](https://img.shields.io/badge/correlated%20views-10-89b4fa)

`lmd-top` correlates the four layers of an llm-d serving stack — `Gateway → EPP (Endpoint
Picker) → Model Server → Infrastructure` — for **heterogeneous accelerators**
(NVIDIA GPU · Rebellions RBLN · Furiosa RNGD · host CPU). It reads your existing
Prometheus + Kubernetes; it owns no data of its own.

## Demo

![lmd-top demo](docs/demo.gif)

<sub>soft (Catppuccin) theme · live braille timelines · cross-layer drill-down. Regenerate: `lmd-top --cast && agg docs/demo.cast docs/demo.gif`.</sub>

## Highlights

- **Four layers, one screen** — Gateway, EPP/InferencePool, model servers, and hardware, correlated: *which model runs where, how requests are routed, how load is distributed.*
- **Heterogeneous accelerators, unified** — GPU (`DCGM_*`) · RBLN (`RBLN_DEVICE_STATUS:*`) · RNGD (`furiosa_npu_*`) side by side; GPU model + VRAM **auto-detected**; **unified-memory** parts (GB10/GH200) marked `∪`; per-node **disk** too.
- **EPP-aware** — introspects the EPP `ConfigMap` (scorers/weights/picker), visualizes routing decisions & per-pod queues, and **diagnoses HTTPRoute→InferencePool vs bypass**.
- **Deploy lifecycle** — per-model **compiled variants** (TP/PP, quant, RBLN/Furiosa NPU options), which **node/disk** they live on, and free-capacity **deploy targets**.
- **Rich TUI** — LED device grid, stacked VRAM bar, braille timelines, active alerting (`A`), logs, `scale`, 4 themes, tasteful animations (`f`), zoom (`z`); **pure Rust, single static binary**.

## Views

Switch with number keys `0`–`9` or `Tab`.

| # | View | Shows |
|---|---|---|
| 0 | **Overview** | Cluster Σ · LED grid · VRAM bar · accelerators by kind/node · EPP path · models · one-line diagnosis |
| 1 | **Accel** | Per-device util / VRAM / temp / power + trend; `⏎` → util/VRAM timeline |
| 2 | **Models** | Per-model accel/node · ready · running/waiting · KV% · tok/s · route · status |
| 3 | **EPP** | Scorers & weights + picker + InferencePool endpoints + request distribution |
| 4 | **Flow** | Gateway → HTTPRoute → backend → pods, InferencePool/EPP/SLO, **EPP-bypass diagnosis**; `⏎` → backend model |
| 5 | **Pods** | `llm-serving` pods (ready / phase / node / restarts) |
| 6 | **Perf** | Per-device history + per-model p95 **QUEUE→PREFILL→DECODE→TPOT→E2E**, tok/s, queues; `⏎` → p50/95/99 + timelines |
| 7 | **Deploy** | **Model lifecycle** — compiled variants (family→build: opts, `@node /path`, status) · deploy targets (free capacity/node) · catalog feasibility |
| 8 | **Events** | k8s + llm-d events (newest first); `⏎` → full message |
| 9 | **Nodes** | Node health · CPU · mem · **disk** · load · devices per node; `⏎`, then `↑↓` picks a device |

## Install

**Prereqs** (audited — binary links only glibc; **no native/C-library deps**): a Rust
toolchain + a C linker (`cc`/`gcc`). Runtime: `kubectl` (kubeconfig) and reachability to
**Prometheus** — no SSH to accelerator nodes. A **truecolor** terminal with a
box-drawing/braille font is recommended (else `LMD_THEME=default`). `xdg-open` is optional
(for the `g` key).

```bash
git clone https://github.com/gspark-etri/lmd-top.git && cd lmd-top
./install.sh                 # install missing prereqs, then `cargo install`
#   ./install.sh --check     # report only   ·   --with-demo  also builds the GIF
# manual: cargo install --path .   (Rust crates are fetched by cargo automatically)
```

## Usage

```bash
lmd-top                      # TUI (permission mode: observe)
lmd-top --mode admin         # allow scale/rollout actions
lmd-top --json               # machine-readable agent state (JSON)
lmd-top --doctor             # survey Prometheus: exporters, metric coverage, gaps
lmd-top --snapshot | --render | --cast   # headless text · CI render · demo asciicast
LMD_PROM=10.0.0.5:30090 LMD_NS=my-ns lmd-top   # point elsewhere
```

**Permission modes** (`--mode`, header badge): `observe` (default, view) → `debug` (+logs `l`)
→ `admin` (+`scale`, with y/n confirm) → `danger` (reserved). **Keys:** `↑↓/kj` select ·
`Enter` drill-down · `←→` prev/next · `/` filter · `o` sort · `l` logs · `s` scale ·
`A` alerts · `t` theme · `f` animations · `z` zoom · `Space` pause · `g` Grafana · `?` help · `q` quit.

**Env:** `LMD_PROM` · `LMD_NS` (`llm-serving`) · `LMD_GRAFANA` · `LMD_THEME`
(`soft`/`default`/`high-contrast`/`colorblind`) · `LMD_W`/`LMD_H` (render size).
Optional YAML `~/.config/lmd-top/lmd-top.yaml` for column order.

**Colors** encode severity/identity; state is a separate glyph (`●` up · `○` idle · `◐` pending
· `⚠` throttle · `⊘` cordoned · `✗` down), so it stays legible in the colorblind theme.
Missing metrics render `–` and fill in once the workload is up.

## Data path

Reads your existing stack (owns no data):

| Layer | Source | Examples |
|---|---|---|
| Accelerators / host | Prometheus | `DCGM_FI_DEV_*`, `RBLN_DEVICE_STATUS:*`, `furiosa_npu_*`, `node_*` |
| Model server | Prometheus | `vllm:*_latency_seconds_bucket`, `vllm:num_requests_*`, `vllm:*kv_cache*` |
| EPP / Pool | Prometheus + ConfigMap | `inference_pool_*`, `inference_extension_*`, `llmd-router-epp` cm |
| Topology / status / actions | `kubectl` | Deployment, Pod, HTTPRoute, Gateway, InferencePool, InferenceObjective |

Two tiers: a ~1 s fast tier (accelerators + nodes) and a ~3 s full snapshot. Pure Rust —
Prometheus over raw `tokio` HTTP/1.0, Kubernetes via `kubectl`.

## Status & roadmap

**✅ Works now (no traffic):** all 10 views · GPU/RBLN/RNGD + node/disk monitoring (auto-detect,
unified-mem) · Flow topology + EPP-bypass diagnosis · EPP ConfigMap introspection · active
alerting · `scale`/`logs` actions · Deploy view (compiled variants, storage node, deploy
targets) · headless `--json`/`--doctor`/`--snapshot`/`--cast` · themes/animation/zoom/permission modes.

**🟡 Needs live EPP-path traffic + vLLM metrics:** per-model p95 latency breakdown, tok/s,
per-pod queue distribution, KV%/TTFT/E2E, EPP request distribution. (The EPP weight `+`/`-`
is a local weight-share simulation, not applied.)

**🔴 Planned:** applied control-plane actions (endpoint drain, traffic/policy-weight apply,
rollout — dry-run→confirm→audit) · EPP per-endpoint score debugger · **NPU compile & deploy
automation** — from the Deploy view, compile a model for RBLN/Furiosa (as a k8s Job with the
vendor toolchain) and deploy the artifact via ModelService, gated by permission mode. See
`ROADMAP.md` / `CHANGELOG.md`.

## Maturity

Verified against a live heterogeneous cluster (8 nodes; GB10 · RBLN · RNGD; EPP/routes/models
live). Experimental (0.x) — interfaces may change.
