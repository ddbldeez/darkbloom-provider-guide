# Darkbloom Provider Guide: Maximize Earnings (Without Breaking Your Node)

**Audience:** other Apple Silicon providers who already installed Darkbloom and want to earn more reliably.  
**Based on:** a two-machine fleet (Mac Studio M3 Ultra 512 GB + MacBook Pro M5 Max 128 GB) run through public alpha (v0.7.x through **0.9.22**).  
**Last verified:** 2026-10-20 on Studio M3 Ultra 512 GB, macOS 27. Provider **0.9.22** (`darkbloom update --check-only` said up to date). Live checks: `models catalog`, `status`, `doctor`, `beta list`, `idle status`, `enroll --help`, and `GET https://api.darkbloom.dev/v1/pricing`. The previous pass was 2026-08-06 on provider **0.8.1**.

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

- **Serving authorization:** on macOS 27 this is **App Attest**, not a new MDM profile. Doctor should say the connection is authorized. `trust_level` can still read `self_signed` after that approval. The field that matters is the authorization path (`app_attest`) and reason (`app_attest_verified` when current).
- **Do not treat `coordinator_acknowledged: false` as offline** if `lifecycle.outcome` is `serving` and jobs are paying. That combination was observed on 0.9.22.
- **MDM:** `darkbloom enroll` on 0.9.22 says new providers need macOS 27 or later and coordinator-qualified App Attest. It says Darkbloom MDM will be deactivated, and MDM-only providers receive no base rewards. Do not install a new Darkbloom MDM profile on macOS 27. Do not run `darkbloom unenroll` unless you mean to drop MDM. Grandfathered pairs can still serve on older macOS for a while.
- **Console session logged in** (APNs code-identity needs a real user session)
- **Automatic login** on dedicated earn boxes (reboot, then auto login, then attest without a human)
- **No idle auto-logout** killing the console session
- **Sleep prevented** while serving (provider should self-caffeinate; still check Energy settings)
- **SIP / authenticated root / hardened runtime** healthy (doctor flags these)
- **Up to date:** `darkbloom update --check-only` (current was **0.9.22** on 2026-10-20)
- A device-wide `devicecheckd` "Should fetch CD hash" doctor warning was present and did not block pay. Doctor labels it as not attributable to Darkbloom.

### Shared / laptop gotchas

| Problem | Why it kills earnings | Fix |
|--------|------------------------|-----|
| No user logged in after reboot | Can’t attest / stay trusted | Enable auto-login for the provider user on always-on machines |
| Lid closed / sleep | Offline or reduced readiness | Keep plugged in, prevent sleep, Power Nap is not enough |
| Heavy local work competing for RAM/GPU | Timeouts → reduced priority | Cap concurrent; park browsers/AI apps; leave headroom |
| App Attest not qualified (`app_attest_qualification_required`) | Online but not paid | Leave the daemon up. Login, enroll, or reinstall does not qualify a build the coordinator has not approved. A new install creates a new Apple key and a cooldown. |
| MDM-only on a Mac that should be on App Attest | No base rewards, per `darkbloom enroll` help | Move to macOS 27 App Attest. Do not download a new MDM profile to "fix" it. |

---

## 3. Model strategy (catalog, then what actually gets routed)

The August 2026 playbook said serve only `gemma-4-26b-qat-4bit` and `gpt-oss-20b`. That is stale. On 2026-10-20 `darkbloom models catalog` returned **13** active text models. Gemma is still in the catalog. It is not the only model, and it is not the highest output sticker.

**Serve models your chip can keep warm, and that this machine is actually being sent.** A higher sticker that sits idle earns zero. Pin `enabled_models`. Do not leave the allowlist empty if other MLX weights are on disk.

### Chip gates (from catalog `required_provider_capabilities`)

| Model ID | Catalog min RAM | Weights | Gate |
|----------|-----------------|---------|------|
| `EigenLabs/Qwen3.8-27B-4bit-mtp` | 36 GB | ~16.3 GB | `apple_m5` and `mlx_nax` |
| `glm-5.3-flash` | 256 GB | ~182 GB | `apple_m5` and `mlx_nax` |

Do not advertise those two on an M1, M2, M3, or M4. This M3 Ultra cannot run them. They can still show up in an **account-wide** earnings list when another machine on the account serves them.

`mimo-v2.6-flash-mopd` has no capability gate in the catalog. Floor is 256 GB, weights about 173 GB. It is a large-RAM model, not an M5-only model.

### Catalog floors (2026-10-20)

Re-check with `darkbloom models catalog`. Weights are the catalog `size_gb`.

| Model ID | Min RAM | Weights |
|----------|---------|---------|
| `EigenLabs/Ternary-Bonsai-2-9B-MLX-2bit` | 24 GB | ~3.5 GB |
| `gpt-oss-20b` | 24 GB | ~12.1 GB |
| `Qwen3.5-9B` | 24 GB | ~6.1 GB |
| `ternary-bonsai-2-27b` | 24 GB | ~8.6 GB |
| `qwen3.6-35b-a3b-vl-mtp-mxfp8` | 32 GB | ~21.3 GB |
| `EigenLabs/Qwen3.8-27B-4bit-mtp` | 36 GB | ~16.3 GB |
| `gemma-4-26b` | 36 GB | ~28.0 GB |
| `gemma-4-26b-qat-4bit` | 36 GB | ~15.6 GB |
| `qwen3.5-35b-a3b` | 36 GB | ~20.9 GB |
| `nvidia-nemotron-3.5-lightning` | 48 GB | ~19.1 GB |
| `gemma-4-26b-8bit` | 64 GB | ~28.0 GB |
| `glm-5.3-flash` | 256 GB | ~182 GB |
| `mimo-v2.6-flash-mopd` | 256 GB | ~173 GB |

`GET /v1/pricing` the same day also listed `qwen3.8-flash-next` and `qwen3-vl-30b-a3b-instruct`. They were **not** in this machine's catalog response. Do not advertise an id the catalog does not list.

### Sticker is not demand

Prices are per 1M tokens from `GET https://api.darkbloom.dev/v1/pricing` on 2026-10-20. They move. Refetch before you act. Fallback that day was $0.05 input / $0.20 output.

Highest **output** stickers that day:

| Model | Input | Output | Who can run it |
|-------|-------|--------|----------------|
| `EigenLabs/Qwen3.8-27B-4bit-mtp` | $0.035 | $2.20 | M5 + NAX only |
| `qwen3.5-35b-a3b` | $0.065 | $0.75 | catalog min 36 GB, no chip gate |
| `qwen3.6-35b-a3b-vl-mtp-mxfp8` | $0.050 | $0.70 | catalog min 32 GB, no chip gate |
| `ternary-bonsai-2-27b` | $0.075 | $0.50 | catalog min 24 GB |
| `gemma-4-26b-qat-4bit` (same sticker as the other two gemma ids) | $0.042 | $0.22 | catalog min 36 GB |
| `Qwen3.5-9B` | $0.080 | $0.13 | catalog min 24 GB |
| `gpt-oss-20b` | $0.018 | $0.09 | catalog min 24 GB |

Gemma is no longer the best-paid workhorse on sticker. A cheap model that is actually routed still beats an expensive model that stays idle.

### What this M3 Ultra was doing (one process, not a rule)

Observed 2026-10-20, provider 0.9.22, process up 11h21m:

- Allowlist and warm set: `Qwen3.5-9B` and `qwen3.6-35b-a3b-vl-mtp-mxfp8`.
- `Qwen3.5-9B` was the most recently used model.
- `qwen3.6-35b-a3b-vl-mtp-mxfp8` stayed resident and idle for that whole process.
- In the last 500 account earnings rows (about 13:27 to 16:52 UTC), this process's current session was 141 jobs, all `Qwen3.5-9B`, about $0.06. That window is not the full 11 hours.
- Other provider ids on the same account were paid in that window for `qwen3.6-35b-a3b-vl-mtp-mxfp8` and `EigenLabs/Qwen3.8-27B-4bit-mtp`.

Read that as routing, not as "everyone should serve the 9B." If `darkbloom status` shows a warm model idle for hours while a smaller one is taking the jobs, the idle model is not earning on that box. Change the set with `darkbloom switch` instead of restarting to "clear" it.

### Pick by what the catalog allows

| Your box | Start here | Why |
|----------|------------|-----|
| Under the model's catalog min | Do not advertise it | Doctor and the coordinator will not treat it as capacity |
| M1–M4 | Any catalog model **without** `apple_m5` / `mlx_nax`, inside your RAM | The two gated models above will not run |
| 24–32 GB | A 24 GB-floor model you can keep warm (`Qwen3.5-9B`, `gpt-oss-20b`, or a Bonsai id) | Gemma's floor is still 36 GB |
| 36 GB and up | One model you can keep healthy. Add a second only if both stay warm **and** both get jobs | Dual-warm that never routes wastes RAM and reputation |
| 256 GB and up | Large models (`mimo-v2.6-flash-mopd`, and `glm-5.3-flash` only on M5 + NAX) are eligible by RAM | RAM does not create demand |

Rules of thumb:

- Pin the allowlist (see §4).
- Prefer one model that finishes cleanly over every model that fits.
- `gemma-4-26b` and `gemma-4-26b-8bit` are heavier than `gemma-4-26b-qat-4bit` at the same or higher floor. They are not the default gemma path.
- There is no `darkbloom earnings` command. Session counters in `darkbloom status` reset on restart. Account pay is the console, or `GET /v1/provider/account-earnings` with the local device token. The default page is short and **account-wide**. `?limit=500` returned 500 rows on 2026-10-20. Match your session if you need this Mac only. Do not commit the token.

### Start command pattern

`--model` is repeatable. `--idle-timeout 0` keeps models loaded. Change a running set with `darkbloom switch --model <id>` so you do not drop the coordinator session.

```bash
# One catalog model you can keep warm. Swap the id for your RAM and chip.
darkbloom start \
  --model Qwen3.5-9B \
  --idle-timeout 0 \
  --local-endpoint
```

**Why `--local-endpoint`:** the same loaded models also serve a local OpenAI endpoint. Useful for a warm-up completion when the warm list is empty. `darkbloom start --help` on 0.9.22 says the local port default is **8000**. The provider backend port in TOML is still **8100**. Use the port `darkbloom local` prints.

```bash
# Replace KEY, port, and model from `darkbloom local` and your allowlist.
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Authorization: Bearer KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"Qwen3.5-9B","messages":[{"role":"user","content":"ping"}],"max_tokens":16}'
```

---

## 4. Config profile that actually works

Edit: `~/.config/darkbloom/provider.toml`

### Earnings-max (reliable) — recommended baseline

```toml
[backend]
# Allowlist only. Empty can auto-select other local weights.
# One model until status shows that model actually receiving work.
enabled_models = ["Qwen3.5-9B"]
engine_v2_kv_backend = "auto"
engine_v2_kv_checkpoint_quantization = "balanced"
engine_v2_kv_quantization = "native"
# Leave the per-model maps empty unless you have measured a reason to pin them.
engine_v2_max_concurrent_by_model = {}
idle_timeout_mins = 0
max_model_slots = 1
mtp_mode = "auto"
port = 8100
preload_models = []
startup_preload = true
startup_preload_timeout_secs = 120
startup_selftest = true
startup_selftest_fail_closed = false

[coordinator]
heartbeat_interval_secs = 5
private_only = false
url = "wss://api.darkbloom.dev/ws/provider"

[gemma_optimizations]
prefill_layer18 = true
weighted_r1 = true

[provider]
auto_restart = true
auto_update = true
memory_reserve_gb = 4
name = "your-provider-name"
update_jitter_seconds = 300
```

A working 0.9.22 config on 2026-10-20 did **not** contain `engine_v2_max_concurrent`, `kv_quant`, or `mtp = true`. Those 0.8.1 keys are not the baseline anymore. MTP is `mtp_mode = "auto"`. Idle is `idle_timeout_mins = 0`, which matches `darkbloom idle` reporting "always ready". The CLI for that policy is `darkbloom idle keep-loaded` or `darkbloom idle unload-after <minutes>`, then restart.

Autopilot (`darkbloom autopilot`, and a `model_autopilot` block) was installed and **not activated** on that check: status said experimental, waiting, no coordinator control. Do not turn it on to chase earnings.

Then:

```bash
darkbloom restart
darkbloom doctor
darkbloom status
```

### Concurrent caps by machine class (hard-won)

Grouped by **unified RAM** (what the catalog cares about), with example Mac configs. **No pricing** — configs change; verify About This Mac.

#### 36–128 GB · concurrent **2–4** (historical reliability band, not a gemma rule)

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

Notes: higher concurrent mainly heats. Demand is often only a few jobs.

#### 192–512 GB · concurrent **6–8** (historical reliability band)

| Product | Chip examples | RAM configs in this band |
|---------|---------------|--------------------------|
| Mac Studio | M2 Ultra | 192 GB |
| Mac Studio | M3 Ultra | 256 · 512 GB |
| Mac Pro | M2 Ultra | 192 GB |

Notes: **8** was stable for long token runs on this fleet under 0.8.x. On 0.9.22 the working Studio config did not pin a global concurrent number. Doctor said unknown profiles keep default **4**, and **8** for automatically paged models on M5 Pro, Max, and Ultra. Leave the automatic default until a box is cool and priority-stable.

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

### Beta features (0.9.22)

```bash
darkbloom beta list
```

Observed `darkbloom beta list` on 0.9.22:

- `gemma-prefill-layer18` default on
- `gemma-weighted-r1` default on
- `mtp` = **auto**. Help text: force MTP on for supported targets. Embedded Qwen heads and Gemma 4 26B QAT default on automatically.

`darkbloom beta enable mtp` is no longer the setup step. Leave `mtp_mode = "auto"` unless you have a measured reason to force it. Confirm with `darkbloom status`: both warm models on this Mac showed `mtp=enabled, active`.

### KV backend (0.8.1 note is historical)

| When | `engine_v2_kv_backend = "auto"` on the machine we checked |
|------|--------------------------------------------------------------|
| 0.8.1 (2026-08-06) | resolved **contiguous**. That release turned paged back off after fleet `token_budget_exhausted` ([#600](https://github.com/Layr-Labs/d-inference/pull/600)). |
| 0.9.22 (2026-10-20, M3 Ultra) | resolved **paged** for `Qwen3.5-9B` and `qwen3.6-35b-a3b-vl-mtp-mxfp8` (`native-paged-full-recurrent-v1`). Jobs were paying on that posture. |

Do not copy the 0.8.1 "force contiguous" fix onto 0.9.22. Read `darkbloom status` and doctor. Doctor on this box said unknown profiles keep concurrent default **4**, and **8** for automatically paged models on M5 Pro, Max, and Ultra.

### Prefix cache

The 0.8.1 contiguous build did not construct the SSD prefix cache, on purpose, because cold and adopted streams diverged ([#600](https://github.com/Layr-Labs/d-inference/pull/600)). That note is about 0.8.1, not a 0.9.22 procedure. After any upgrade, a colder first hour is not a reason to restart in a loop.

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
| Advertising a model your chip cannot run (M5/NAX ids on an M3) | `model_not_loaded` and no pay |
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
- Disk: remove local weights you are not willing to keep out of `enabled_models` (`darkbloom models remove <id>`).
- Confirm auto-login still works after macOS updates. On macOS 27, confirm App Attest is still the authorization path. Do not reinstall an MDM profile to "fix" trust.

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
3. Confirm the model id matches the catalog and is in `enabled_models`. On an M3, that list must not include an `apple_m5` / `mlx_nax` model.

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
- Extra RAM mostly buys catalog eligibility (36 GB gets you gemma and the 35B-class models, 256 GB gets you MiMo). It does not buy M5-only models onto an M3, and it does not create demand.
- **Always-on desktop/Studio** beats a laptop that sleeps, leaves home, or thermal-throttles.
- Alpha-era floors / subsidies (if any) are **temporary policy**, not a product guarantee. Never finance hardware on subsidy math.
- Best second node historically for raw $/bandwidth: used high-bandwidth Ultra-class — **only after** your first node shows real utilization for weeks.

---

## 9. Copy-paste profiles

### Profile A — Dedicated Studio (large RAM, M1–M4)

- Models: one catalog model your chip can run. Add a second only after the first is getting jobs. Do not add `EigenLabs/Qwen3.8-27B-4bit-mtp` or `glm-5.3-flash` on an M3.
- This M3 Ultra on 2026-10-20 was routed `Qwen3.5-9B` while a warm `qwen3.6-35b-a3b-vl-mtp-mxfp8` sat idle. That is an observation, not an order to delete the larger model on every Studio.
- Concurrent: leave the 0.9.22 automatic default until the box is cool and priority-stable
- Idle: **0** (`darkbloom idle keep-loaded`)
- MTP: **`mtp_mode = auto`**
- `memory_reserve_gb`: **4**
- MLX env reserve: **~32** on 512 GB if you still set `DARKBLOOM_MLX_MEMORY_RESERVE_GB` (see the LaunchAgent gotcha)
- Auto login: **on**
- Auto update / restart: **on**
- App Attest: required on macOS 27
- Fan control: optional (`darkbloom fan diagnose` first)

### Profile B — Laptop / mini that clears a 36 GB model

- Model: one model at or under your RAM. Gemma qat still fits at 36 GB. So do `qwen3.5-35b-a3b` and, from 32 GB, `qwen3.6-35b-a3b-vl-mtp-mxfp8`.
- Concurrent: automatic default. The old 2–4 band is a heat cap, not a TOML key you must set.
- Idle: **0** if always plugged in; `darkbloom idle unload-after 15` to `30` if you need the RAM back
- MTP: auto
- Auto login: on if this is a dedicated lid-open box
- Fan control: `darkbloom fan diagnose`, then optional `fan enable`

### Profile C — Small Mac 24–32 GB

- Model: a 24 GB-floor catalog id (`Qwen3.5-9B`, `gpt-oss-20b`, or a Bonsai id). Do not force gemma. Its floor is still 36 GB.
- `qwen3.6-35b-a3b-vl-mtp-mxfp8` starts at 32 GB.
- Concurrent: leave automatic. If deadlines appear, you are already too wide.
- Idle: **0** if dedicated

### Profile D — Shared family Mac

- One small catalog model
- Concurrent: do not raise it
- Idle unload: 15–30 minutes so people can use the Mac
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
darkbloom idle status
darkbloom switch --model <catalog-id>
darkbloom models list          # advertised set should match enabled_models, not every weight on disk
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

> Stay App Attest authorized on macOS 27 (`self_signed` in trust_level can still be a paying node). Pin catalog models your chip can run. Keep them warm. Leave MTP on auto. Leave concurrency on the automatic default until you are cool and full priority. A higher sticker that sits idle earns nothing. Finished tokens win. Failed jobs lose tomorrow's routing.

---

## Assumptions & verification

- **Observed (2026-10-20, provider 0.9.22, M3 Ultra, macOS 27):** 13 catalog models and the floors in §3. `GET /v1/pricing` stickers in §3. App Attest path `app_attest`, reason `app_attest_verified`, `trust_level` still `self_signed`, lifecycle `serving` with `coordinator_acknowledged` false, and jobs paying. `engine_v2_kv_backend=auto` resolved paged on both warm Qwen models. `mtp_mode=auto` and both slots MTP active. Doctor concurrent text: unknown profiles default 4; 8 for automatically paged models on M5 Pro, Max, and Ultra. No `darkbloom earnings` command. Account earnings endpoint is account-wide. This process: 929 requests and 1,363,276 tokens in 11h21m. In the last 500 account rows, this session was 141 `Qwen3.5-9B` jobs and the warm 35B-class Qwen was idle the whole process.
- **Observed earlier (2026-08-06, provider 0.8.1):** two-model playbook, stock auto resolved contiguous, MTP was a beta switch, MDM was the trust story. Kept above only as history.
- **Derived:** do not advertise M5/NAX models on this M3. Do not treat sticker rank as demand.
- **Assumed / re-check live:** next week's demand mix, whether paged stays the stock auto resolution on other chips, and any base-reward or floor policy. Refetch catalog and `/v1/pricing` after upgrades.
- Re-check after every provider release: `darkbloom doctor`, `darkbloom beta list`, and your console. The PNG cards in the README are still the August graphics. The text is the current source.

**Independent advice — not affiliated with Darkbloom or Eigen Labs; not official docs.**  
**Not financial advice. Public alpha software. Demand and policy change.**
