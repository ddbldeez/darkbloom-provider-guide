# Darkbloom Provider Guide: Maximize Earnings (Without Breaking Your Node)

**Audience:** other Apple Silicon providers who already installed Darkbloom and want to earn more reliably.  
**Based on:** a two-machine fleet (Mac Studio M3 Ultra 512 GB + MacBook Pro M5 Max 128 GB) run hard through public alpha (v0.7.x → **0.8.1**).  
**Last verified:** 2026-08-06 on Studio M3 Ultra 512 GB — provider **0.8.1**, live `catalog` / `status` / `doctor` / `beta list` / `update --check-only`. Pricing not re-fetched this pass.

**Independent advice.** Not affiliated with Darkbloom (darkbloom.dev) or Eigen Labs (eigenlabs.org). Not official documentation. Darkbloom / Eigen Labs names and marks belong to their owners. Use at your own risk.

---

## 0. The one rule that matters

**Maximize finished tokens × success rate × hours online.**

Not:

- max concurrent jobs
- max models loaded
- max GPU cache size
- “looks busy” in Activity Monitor

The coordinator routes work to machines that **finish cleanly**. Deadlines, mid-stream cancels, OOM, tool-call failures, and “reduced priority” all cut your share of future traffic. Aggressive concurrent that causes failures usually **earns less** than a boring, always-warm, full-priority node.

**Disclaimer — results vary by hardware.** Unified RAM is only one constraint. Memory **bandwidth / throughput**, thermals, and sustained power also decide finished tokens. A **stable, high-speed internet connection is critical** — weak or flaky links cause timeouts, reduced priority, and fewer jobs. Numbers and concurrent bands in this guide are starting points from one fleet’s experience, **not guarantees**.

---

## 1. What actually pays (mental model)

Three layers, in practice:

1. **Eligibility** — hardware trust, MDM/attestation, console login, online, not sleeping.  
   If this fails, you earn ~nothing. Fix this before tuning models.
2. **Routable capacity** — correct network models warm, not cold after idle; version current; reputation healthy.  
   This is most of what “earnings max” optimizes.
3. **Organic token work** — real inference demand.  
   Demand is still the binding constraint in public alpha. No config turns a quiet network into a full-time job.

**CLI vs console:**  
`darkbloom status` counters **reset on every restart**. Use the web console for 24h / lifetime tokens and $. CLI is for “is this box healthy right now?”

**Pricing / floors change.** Check live pricing and your own console before making hardware decisions. Do **not** buy a Mac expecting calculator best-case.

Hardware and network still bound layer 2: more RAM does not fix low memory bandwidth, hot chassis, or a bad uplink. See the disclaimer under §0 and the concurrent / hardware sections.

---

## 2. Prerequisites checklist (eligibility first)

Run until green:

```bash
darkbloom doctor
darkbloom verify
darkbloom status
```

You want roughly:

- **Trust:** `hardware / online` (or the current top tier that says eligible for traffic)
- **MDM / enrollment** present if required for your account
- **Console session logged in** (APNs code-identity needs a real user session)
- **Automatic login** on dedicated earn boxes (reboot → auto login → attest without a human)
- **No idle auto-logout** killing the console session
- **Sleep prevented** while serving (provider should self-caffeinate; still check Energy settings)
- **SIP / Secure Boot / hardened runtime** healthy (doctor flags these)
- **Up to date:** `darkbloom update --check-only` → current (**0.8.1+** as of this writing)

### Shared / laptop gotchas

| Problem | Why it kills earnings | Fix |
|--------|------------------------|-----|
| No user logged in after reboot | Can’t attest / stay trusted | Enable auto-login for the provider user on always-on machines |
| Lid closed / sleep | Offline or reduced readiness | Keep plugged in, prevent sleep, Power Nap is not enough |
| Heavy local work competing for RAM/GPU | Timeouts → reduced priority | Cap concurrent; park browsers/AI apps; leave headroom |
| Unenrolled / software-only trust | Less or no private/fleet traffic | Complete MDM enroll + keep profile installed |

---

## 3. Model strategy (only two models)

**Serve only these. Nothing else.**

### Catalog + recommended RAM (verify with `darkbloom models catalog`)

Live catalog floors and practical “runs well warm” guidance (**2026-08-06**, `darkbloom models catalog`):

| Model ID | Role | Weights (est.) | Catalog min RAM | **Recommended unified RAM** | Notes |
|----------|------|----------------|-----------------|----------------------------|--------|
| **`gemma-4-26b-qat-4bit`** | **Primary earner** | ~15.6 GB | ≥ **36 GB** | **≥ 48 GB** (sweet: **64–128 GB+**) | Catalog shows ~15.6 GB weights. Primary when you meet min RAM. Live $/1M: check console (not re-verified this pass). |
| **`gpt-oss-20b`** | **Small-box / fill-in** | ~12.1 GB | ≥ **24 GB** | **≥ 32 GB** (fine on **24–36 GB** if that’s all you have) | Catalog shows ~12.1 GB weights. Fits machines that can’t run gemma. Dual-warm on big boxes for extra job types. |

Also in catalog but **not** this playbook’s defaults:

| Model ID | Catalog min RAM | Why skip for max-earn default |
|----------|-----------------|-------------------------------|
| `gemma-4-26b` | ≥ 36 GB | Larger (~28 GB weights); qat-4bit is the usual network path |
| `gemma-4-26b-8bit` | ≥ **64 GB** | Rollback path; heavier than qat-4bit |

**Do not invert this:** gpt-oss is the **smaller-RAM** model. Gemma is the **better-paid workhorse** once you clear ~36 GB (ideally 48 GB+ for headroom).

### Pick by machine size

| Unified RAM | Serve | Why |
|-------------|-------|-----|
| **24–32 GB** | **`gpt-oss-20b` only** | Below gemma’s 36 GB catalog floor |
| **36–47 GB** | **`gemma-4-26b-qat-4bit` only** (preferred) *or* gpt-oss if gemma is unstable | Meets gemma min; dual-warm is tight |
| **48–128 GB** | **`gemma-4-26b-qat-4bit` primary**; add **gpt-oss** only if dual-warm stays clean | Gemma earns more per token; gpt-oss is optional fill |
| **192–512 GB** | **gemma + gpt-oss** dual-warm when healthy | Big iron buys concurrent + dual job types — still **gemma-first** |

Do **not** download, warm, advertise, or leave `enabled_models` open to any other weights for fleet serving. Extra models waste disk, risk bad loads, and do not help earnings under this playbook.

Rules of thumb:

- Pin the allowlist in config (see §4) so auto-select cannot pick strays.
- **Default = gemma** whenever RAM ≥ 36 GB (recommend ≥ 48 GB).
- **gpt-oss** = small machines *or* second warm model on large machines — not “big RAM only.”
- If disk is tight, remove every other local MLX weight.

### Start command pattern (earnings-max)

```bash
# ≥36–48 GB+ — gemma primary, stay warm forever
darkbloom start \
  --model gemma-4-26b-qat-4bit \
  --idle-timeout 0 \
  --local-endpoint
```

```bash
# 24–32 GB only (or gemma won’t load) — gpt-oss only
darkbloom start \
  --model gpt-oss-20b \
  --idle-timeout 0 \
  --local-endpoint
```

```bash
# Large box (≈128 GB+) — dual-warm when both stay healthy
darkbloom start \
  --model gemma-4-26b-qat-4bit \
  --model gpt-oss-20b \
  --idle-timeout 0 \
  --local-endpoint
```

**Why `--local-endpoint`:** keeps a local OpenAI endpoint on the same loaded models. Useful for:

- self-tests / “warm kick” if Warm list is empty but models are advertised
- verifying the engine without waiting for fleet traffic

Kick example (after `darkbloom local` for the key):

```bash
# Replace KEY and port from `darkbloom local`
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Authorization: Bearer KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gemma-4-26b-qat-4bit","messages":[{"role":"user","content":"ping"}],"max_tokens":16}'
```

---

## 4. Config profile that actually works

Edit: `~/.config/darkbloom/provider.toml`

### Earnings-max (reliable) — recommended baseline

```toml
[backend]
# Allowlist only — do not leave empty (empty can auto-select other local weights).
# ≥48 GB+ dual-warm example:
enabled_models = ["gemma-4-26b-qat-4bit", "gpt-oss-20b"]
# 36–128 GB gemma-only (most machines that clear gemma min):
# enabled_models = ["gemma-4-26b-qat-4bit"]
# 24–32 GB small box (below gemma catalog floor):
# enabled_models = ["gpt-oss-20b"]
engine_v2_kv_backend = "auto"  # 0.8.1+ resolves contiguous (see §4 KV notes)
engine_v2_max_concurrent = 8   # Studio-class operator pick; fleet default after 0.8.1 is 4
idle_timeout_mins = 0          # NEVER unload when quiet
kv_quant = false               # still not a free win as of 0.8.x
max_model_slots = 2            # 1 if single-model
mtp = true                     # or: darkbloom beta enable mtp  (Gemma)
port = 8100
startup_preload = true
startup_preload_timeout_secs = 180
startup_selftest = true
startup_selftest_fail_closed = false

[coordinator]
heartbeat_interval_secs = 5
private_only = false
url = "wss://api.darkbloom.dev/ws/provider"

[provider]
auto_restart = true
auto_update = true
memory_reserve_gb = 4          # toml reserve — keep modest
name = "your-provider-name"
update_jitter_seconds = 300
```

Then:

```bash
darkbloom beta enable mtp
darkbloom restart
darkbloom doctor
```

### Concurrent caps by machine class (hard-won)

Grouped by **unified RAM** (what the catalog cares about), with example Mac configs. **No pricing** — configs change; verify About This Mac.

#### 36–128 GB · gemma primary · concurrent **2–4**

| Product | Chip examples | RAM configs in this band |
|---------|---------------|--------------------------|
| Mac mini | M4 Pro | 48 GB |
| MacBook Pro | M3 Pro | 36 GB |
| MacBook Pro | M3 Max | 36 · 48 · 64 · 128 GB |
| MacBook Pro | M4 Max | 36 · 48 · 64 · 128 GB |
| MacBook Pro | M5 Pro | 48 GB |
| MacBook Pro | M5 Max | 48 · 64 · 128 GB |
| Mac Studio | M1 Max | 64 GB |
| Mac Studio | M1 Ultra | 64 · 128 GB |
| Mac Studio | M2 Max | 64 · 96 GB |
| Mac Studio | M2 Ultra | 64 · 128 GB |
| Mac Studio | M4 Max | 36 · 64 · 128 GB |
| Mac Studio | M3 Ultra | 96 GB |
| Mac Pro | M2 Ultra | 64 GB |

Notes: recommend **≥48 GB** for comfortable gemma headroom (catalog min 36 GB). Higher concurrent mainly **heats**; demand often only 3–4 jobs.

#### 192–512 GB · gemma · concurrent **6–8**

| Product | Chip examples | RAM configs in this band |
|---------|---------------|--------------------------|
| Mac Studio | M2 Ultra | 192 GB |
| Mac Studio | M3 Ultra | 256 · 512 GB |
| Mac Pro | M2 Ultra | 192 GB |

Notes: **gemma-4-26b-qat-4bit** only for this concurrent band. **8** was stable for long token runs.

**Disclaimer — results vary by hardware.** Unified RAM is only one constraint. Memory **bandwidth / throughput**, thermals, and sustained power also decide finished tokens. A **stable, high-speed internet connection is critical** — weak or flaky links cause timeouts, reduced priority, and fewer jobs. Concurrent bands above are starting points from one fleet’s experience, not guarantees.

If the console shows **reduced priority** or logs show `request exceeded 120s deadline`:

1. Drop concurrent one step (8→4 or 4→2).
2. Stay online and warm 30–60+ minutes.
3. Do **not** thrash restarts — restarts reset session stats and can slow recovery.

### Two different “memory reserve” knobs

| Knob | Where | What it does |
|------|--------|--------------|
| `memory_reserve_gb` | `provider.toml` | Provider-side reserve; keep modest (e.g. **4**) for max serve budget |
| `DARKBLOOM_MLX_MEMORY_RESERVE_GB` | **process environment** | MLX ceiling / how much RAM to leave for the OS and other apps. **Higher = less MLX budget** |

Example env targets used successfully:

- Studio 512 GB: `DARKBLOOM_MLX_MEMORY_RESERVE_GB=32` (~lots of cache headroom still available)
- Laptop 128 GB: `DARKBLOOM_MLX_MEMORY_RESERVE_GB=40` (protect the rest of the machine)

**Gotcha:** `darkbloom start` rewrites the LaunchAgent and can **strip** `EnvironmentVariables`. After start, verify the live process:

```bash
pgrep -lf darkbloom
# inspect env on the pid:
ps eww -p <PID> | tr ' ' '\n' | grep DARKBLOOM_MLX
```

If the env keeps disappearing, wrap ProgramArguments with `/usr/bin/env DARKBLOOM_MLX_MEMORY_RESERVE_GB=…` in the LaunchAgent (backup the plist first).

### Beta features

```bash
darkbloom beta list
darkbloom beta enable mtp      # Gemma 4 drafter-assisted decode — faster, token-identical
# kv-quant: leave OFF until doctor/beta says it actually works for your version
```

As of **0.8.1**: **mtp = on is a real speed win** for Gemma (`darkbloom beta list` → `mtp`); **kv-quant still not a free earnings boost**.

### KV backend + concurrency (0.8.0 → 0.8.1)

| Release | Stock default (`engine_v2_kv_backend = "auto"`) | Concurrent default |
|---------|-----------------------------------------------|--------------------|
| **0.8.0** | resolved **paged** | raised toward **8** |
| **0.8.1** | resolved **contiguous** again | fleet migrated back toward **4** |

**Why the revert (from release notes / [#600](https://github.com/Layr-Labs/d-inference/pull/600)):** under auto-paged, fleet KV capacity was far smaller than contiguous grants → high `token_budget_exhausted`, slow TTFT, client cancels, worse uptime. Paged is still available if you set `engine_v2_kv_backend = "paged"` explicitly (refusal if unbuildable, not silent degrade). Crash-loop backend guard and parity harness remain in the binary.

This guide’s concurrent bands (2–4 mid / 6–8 Studio) are still the reliability playbook. Fleet default 4 is the safe stock path; only raise when the box stays cool and priority-stable.

Confirm live posture: `darkbloom status` / `doctor` → `kv=contiguous|paged` and concurrent.

### Prefix / SSD cache (0.8.1 reality)

On **0.8.1 stock auto → contiguous**, the product **does not construct** the SSD prefix cache on that slot: measured cold-vs-adopted token streams **diverge** on contiguous for the production catalog models, so construction is gated off to avoid wrong answers ([#600](https://github.com/Layr-Labs/d-inference/pull/600)).

- Exact prefix adoption remains a **paged**-backend property for flagship models when you opt into paged.
- After any upgrade: first hours can look colder for other reasons (warm-up, demand, reputation). Don’t thrash restarts.
- Older “0.7.12 restored contiguous hybrid SSD reuse” notes are **superseded** by the 0.8.1 gate.

---

## 5. Reliability = reputation = more work

### Do these

- Stay on **auto_update** (watchdog + predecessor rollback is the safety net).
- Keep models **warm** (`idle_timeout_mins = 0` on dedicated earn boxes).
- Prefer **fewer concurrent jobs finishing fast** over many jobs timing out.
- Keep the machine cool enough that you don’t thermal-throttle mid-stream (`darkbloom fan diagnose` / optional `fan enable` — see below).
- After any bad spell: **wait online**, don’t config-thrash.

### Avoid these

| Anti-pattern | Why it hurts |
|--------------|--------------|
| Concurrent 16–24 “because I have RAM” | Deadlines / reduced priority |
| Advertising models that never load | `model_not_loaded` self-routes, wasted reputation |
| Default idle unload (60m) on a quiet network | Every new job pays cold-start tax; you lose TTFT races |
| Any model besides gemma-4-26b-qat-4bit / gpt-oss-20b | Disk waste, load risk, no benefit under this playbook |
| Leaving `enabled_models` empty with other weights on disk | Auto-select can advertise junk you never warm |
| Restarting to “clear cache” without a live failure | Loses warm state + session metrics |
| Leaving laptop sleep / no auto-login | Invisible offline hours |
| Ignoring reduced priority for days | You silently fall out of the good queue |

### Thermals

Elevated fans under sustained inference are normal. Sustained heat + thermal throttle often shows up as timeouts and **reduced priority** — lower concurrent, improve airflow/ventilation, stay plugged in, and consider fan control below.

### Fan control (thermals) — all Apple Silicon providers

Darkbloom ships **opt-in experimental** temperature-based fan control. It is **not** laptop-only: any supported Mac can use it. **Portables often benefit most** (thinner chassis, more throttle under continuous warm serve); Studio/mini/desktop boxes can enable it too when automatic fans lag under load.

| Fact | Detail |
|------|--------|
| Default | **Off** — macOS automatic fan control |
| Not in | `provider.toml` (CLI + privileged helper only) |
| Enable | Installs a narrowly scoped **root helper** — admin password required |
| When it acts | Only while a **signed Darkbloom provider is active** and a validated **GPU sensor is hot enough** |
| Knobs | `--speed` 60–90% of each fan’s max (default **80**); `--temperature` °C engage threshold (default **45**) |
| Undo | `darkbloom fan disable` (restore automatic) · `darkbloom fan uninstall` (remove helper) |

**Always diagnose first** (compatibility varies by chip/sensor map):

```bash
darkbloom fan diagnose
# or: darkbloom fan status
```

If `Compatibility: supported` (or equivalent) and you want more aggressive cooling while earning:

```bash
# defaults: --speed 80 --temperature 45  (admin prompt)
darkbloom fan enable

# or explicit:
darkbloom fan enable --speed 80 --temperature 45

darkbloom fan status
```

Tune after enable:

```bash
darkbloom fan configure --speed 75 --temperature 50
```

Back to stock macOS fans:

```bash
darkbloom fan disable
# fully remove helper:
# darkbloom fan uninstall
```

**Caveats**

- Experimental — can be **louder**; not required if thermals and priority stay healthy on automatic.
- Does **not** replace modest concurrent, good ventilation, or a plugged-in always-on setup.
- Re-check after provider upgrades: `darkbloom help fan` (subcommand surface can move in alpha).
- Policy on disk (root): `/Library/Application Support/Darkbloom/fan-policy.json` when installed.

---

## 6. Daily / weekly ops (10 minutes)

### Daily

```bash
darkbloom status
darkbloom doctor
# optional: darkbloom logs | tail
```

Green flags:

- Daemon running, trust eligible
- Warm models list non-empty (for your intended models)
- Requests/tokens climbing over hours (not stuck at 0 after a long quiet period *and* console also dead)

### After every update

```bash
darkbloom update --check-only
darkbloom doctor
darkbloom status
# confirm warm + hardware trust
```

Watchdog promotes only after a stabilization window (~10 min). Predecessor binaries live under `~/.darkbloom/recovery/` for rollback.

### Weekly

- Console: 24h tokens, success rate, priority, payouts.
- Disk: remove every local model that is not gemma-4-26b-qat-4bit or gpt-oss-20b (`darkbloom models remove <id>`).
- Confirm auto-login / MDM still healthy after macOS updates.

---

## 7. Troubleshooting playbook

### A) Online + trusted + warm, but zero jobs

Often **quiet demand** or **temporary reputation lag**, not bad TOML.

1. Confirm console fleet is also quiet (not just you).
2. Wait 15–60 minutes without restarting.
3. Local kick once via `--local-endpoint` if Warm is empty but you expected load.
4. Only then dig logs for deadline / tool-call / OOM patterns.

### B) Advertised but Warm empty / `model_not_loaded`

1. Check `startup_preload = true` and idle timeout 0.
2. Local chat completion to force load.
3. Confirm model id is exactly `gemma-4-26b-qat-4bit` or `gpt-oss-20b` (and is in `enabled_models`).

### C) Reduced priority / many 120s deadlines

1. Cut concurrent (half it).
2. Drop second model if dual-warm.
3. Reduce competing local GPU/RAM apps.
4. Stay warm; let reputation recover.

### D) Doctor says “model fits in RAM FAIL” but status is warm and serving

Doctor RAM estimate can lag reality. Prefer live `status` + console if traffic is flowing.

### E) After `darkbloom start`, settings “forgot”

- TOML is source of truth for most knobs.
- LaunchAgent args + MLX env can be rewritten by start — re-verify plist and process env.

---

## 8. Hardware honesty (read before buying another Mac)

**Results vary.** RAM gets you into the model band; **memory bandwidth / throughput** is what usually drives tokens/sec; thermals and power decide whether you sustain it. **Stable, high-speed internet is critical** — the coordinator cannot route finished work through a flaky link. Same chip + same RAM can earn very differently on different networks or duty cycles.

- **Organic $ scales with real demand and usable bandwidth**, not “I have 512 GB.”
- Extra RAM mostly buys: **gemma eligibility** (36 GB floor, 48 GB+ recommended), dual-warm gemma + gpt-oss, more concurrent/KV cache, higher **floor tiers** (if floors still exist) — not free money from random other models.
- **Always-on desktop/Studio** beats a laptop that sleeps, leaves home, or thermal-throttles.
- Alpha-era floors / subsidies (if any) are **temporary policy**, not a product guarantee. Never finance hardware on subsidy math.
- Best second node historically for raw $/bandwidth: used high-bandwidth Ultra-class — **only after** your first node shows real utilization for weeks.

---

## 9. Copy-paste profiles

### Profile A — Dedicated Studio ≥128 GB (max reliable tokens)

- Models: **`gemma-4-26b-qat-4bit` + `gpt-oss-20b`** (gemma-first dual-warm) — concurrent bands in this guide are gemma-first; dual-warm only if healthy
- Recommended RAM: **192–512 GB** class
- Concurrent: **8**
- Idle: **0**
- MTP: **on** (gemma)
- `memory_reserve_gb`: **4**
- MLX env reserve: **~32** on 512 GB (tune to leave OS comfortable)
- Auto login: **on**
- Auto update / restart: **on**
- **Fan control:** optional if automatic cooling lags under continuous load (`darkbloom fan diagnose` first)

### Profile B — Laptop / mini ≥48 GB (earn without melting)

- Model: **`gemma-4-26b-qat-4bit` only** (add gpt-oss only if dual-warm stays clean)
- Recommended RAM: **48–128 GB** (catalog min for gemma is 36 GB)
- Concurrent: **2–4**
- Idle: **0** if always plugged and docked; else 15–30 if you need RAM back when idle
- MTP: **on**
- MLX env reserve: **higher** (e.g. 40 on 128 GB) so the machine stays usable
- Auto login: strongly recommended if it’s a dedicated earn lid-open box
- **Fan control:** `darkbloom fan diagnose` → optional `fan enable` (experimental; highly useful on continuous laptop serve; also fine on desktops)

### Profile C — Small Mac 24–32 GB (gpt-oss path)

- Model: **`gpt-oss-20b` only** (gemma catalog floor is 36 GB)
- Recommended RAM: **≥ 32 GB** (catalog min 24 GB)
- Concurrent: **1–2**
- Idle: **0** if dedicated; else 15–30 if shared
- Don’t force gemma below 36 GB

### Profile D — Shared family Mac (minimize friction)

- Model: **gemma** if ≥48 GB, else **gpt-oss**
- Concurrent: **1–2**
- Idle unload: **15–30** so the machine frees RAM for humans
- Don’t fight thermals or UI lag for a few extra tokens
- Accept lower earnings for lower household pain

---

## 10. Minimal command card

```bash
darkbloom status
darkbloom doctor
darkbloom verify
darkbloom logs
darkbloom report          # upload logs to coordinator when stuck
darkbloom local          # local endpoint + API key
darkbloom update
darkbloom beta enable mtp
darkbloom restart
darkbloom models list          # should only show gemma / gpt-oss for this playbook
```

Config: `~/.config/darkbloom/provider.toml`  
State: `~/.darkbloom/daemon-state.json`  
Logs: `~/.darkbloom/provider.log`, unified logging via `darkbloom logs`  
LaunchAgent: `~/Library/LaunchAgents/io.darkbloom.provider.plist`  
Watchdog: `~/Library/LaunchAgents/io.darkbloom.watchdog.plist`

---

## 11. What “good” looks like

Healthy earning node:

- `doctor` all green on trust + traffic readiness
- Warm list shows the model(s) you intend to serve
- Version current; watchdog not flapping versions
- Console priority normal / full (not stuck reduced)
- Tokens accumulate over the day **without** constant restarts
- Thermals boring; fans may spin, machine doesn’t crash

Unhealthy “busy” node:

- High concurrent, many timeouts
- Models advertised but never warm
- Frequent restarts to “fix” things
- Reduced priority for hours after a bad config experiment

---

## 12. Shareable one-liner for other providers

> Stay hardware-trusted. Gemma-4-26b-qat-4bit is the primary earner (catalog ≥36 GB, recommend ≥48 GB). gpt-oss-20b is for 24–32 GB boxes or dual-warm fill on large machines—not “big RAM only.” Keep models warm (idle 0), enable MTP on gemma, modest concurrent (≈2 small / ≈4 laptop / ≈8 desktop), auto-update. Finished tokens win; failed jobs lose tomorrow’s routing.

---

## Assumptions & verification

- **Observed (2026-08-06, provider 0.8.1):** catalog mins (gpt-oss ≥24 GB / ~12.1 GB weights, gemma-qat ≥36 GB / ~15.6 GB, gemma full ≥36 GB / ~28 GB, gemma-8bit ≥64 GB); stock `auto` → `kv=contiguous`; MTP beta on; doctor all green / hardware-trusted; concurrent bands from author fleet; auto-login; concurrent-24 reduced priority historically; LaunchAgent MLX env strip; 0.8.1 disables SSD prefix construction on contiguous slots.
- **Derived:** “recommended RAM” headroom bands (48 GB+ for comfortable gemma; 32 GB+ for gpt-oss); dual-warm only on large machines.
- **Assumed / re-check live:** demand mix week to week, floors/subsidies, live $/1M prices, paged opt-in safety on your chip — run `darkbloom models catalog` and console after upgrades.
- Re-check after every major provider release: `darkbloom beta list`, release notes on `Layr-Labs/d-inference`, and your console priority.

**Independent advice — not affiliated with Darkbloom or Eigen Labs; not official docs.**  
**Not financial advice. Public alpha software. Demand and policy change.**
