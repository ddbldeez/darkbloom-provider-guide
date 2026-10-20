# Darkbloom Provider Guide

**Maximize finished tokens × success rate × hours online** — practical notes from running Apple Silicon providers in public alpha.

This repo is for other **Darkbloom providers** who already installed the CLI and want a clear, visual path to a reliable earnings setup. It is **not** official Darkbloom documentation.

| | |
|--|--|
| **Last verified** | 2026-10-20 · provider **0.9.22** · macOS 27 · M3 Ultra |
| **Full guide** | [`guide.md`](./guide.md) |
| **For coding agents** | [`AGENTS.md`](./AGENTS.md) — point Claude / Cursor / Codex / Grok at this repo |
| **Source graphics** | [`assets/Darkbloom-Provider-Guide-Graphics.tldraw`](./assets/Darkbloom-Provider-Guide-Graphics.tldraw) |
| **Darkbloom** | [darkbloom.dev](https://www.darkbloom.dev/) · [console](https://console.darkbloom.dev/) · [install](https://api.darkbloom.dev/install.sh) |

---

## Disclaimer

**Independent advice.** Not affiliated with Darkbloom ([darkbloom.dev](https://www.darkbloom.dev/)) or Eigen Labs ([eigenlabs.org](https://www.eigenlabs.org/)). Not official documentation. Darkbloom / Eigen Labs names and marks belong to their owners. Use at your own risk.

**Results vary by hardware.** Unified RAM is only one constraint. Memory **bandwidth / throughput**, thermals, and sustained power also decide finished tokens. A **stable, high-speed internet connection is critical** — weak or flaky links cause timeouts, reduced priority, and fewer jobs.

Concurrent bands and model picks below are **starting points from one fleet’s experience**, not guarantees. Catalog mins and pricing change — always re-check:

```bash
darkbloom models catalog
darkbloom doctor
```

Public alpha software. Not financial advice. Demand and policy change.

---

## How to use this page

Cards are in **order**. Skim the graphic, read the short notes, move on. Deep detail lives in [`guide.md`](./guide.md).

**Using a local coding agent?** Open this repo (or paste the path) and tell it to follow [`AGENTS.md`](./AGENTS.md). Example:

```text
Read AGENTS.md and guide.md. Apply the earnings-max profile to this Mac:
run darkbloom status/doctor/catalog, then propose a minimal provider.toml change.
```

---

## 01 · The one rule

![The one rule that matters](./assets/cards/01-one-rule.png)

**Maximize finished tokens × success rate × hours online.**

Not max concurrent, not max models, not “looks busy.” The coordinator routes to machines that **finish cleanly**. Aggressive concurrent that causes deadlines usually earns **less**.

---

## 02 · What actually pays

![What actually pays](./assets/cards/02-what-pays.png)

Three layers — fix **#1** before tuning concurrent:

1. **Eligibility** — on macOS 27, App Attest (`app_attest_verified`). `trust_level` may still say `self_signed` while you are serving. MDM is not the new-provider path. Console login, online, no sleep.
2. **Routable capacity** — warm catalog models your chip can run, current version, jobs actually arriving
3. **Organic demand** — real network jobs (the binding constraint)

No config turns a quiet network into full-time income.

**CLI vs console:** `darkbloom status` counters **reset on restart**. Console is 24h / lifetime truth.

---

## 03 · Serve what the catalog will route

![Serve only these](./assets/cards/03-models-only.png)

The picture above is the August 2026 two-model card. The text here is the 2026-10-20 check. Full table is in [`guide.md`](./guide.md).

`darkbloom models catalog` returned **13** models. Gemma is still listed. It is not the only model, and it is not the highest output sticker.

- Pin `enabled_models`. Do not advertise weights the catalog does not list.
- `EigenLabs/Qwen3.8-27B-4bit-mtp` and `glm-5.3-flash` require `apple_m5` and `mlx_nax`. Do not put them on an M3.
- On this M3 Ultra, a warm `qwen3.6-35b-a3b-vl-mtp-mxfp8` sat idle for 11 hours while `Qwen3.5-9B` took the jobs. Sticker is not demand. That is one machine, not an order for every provider.

```bash
darkbloom models catalog
darkbloom start --model Qwen3.5-9B --idle-timeout 0 --local-endpoint
```

---

## 04 · Concurrent caps (hard-won)

![Concurrent caps](./assets/cards/04-concurrent-caps.png)

The card still shows the August RAM bands. Those bands are historical heat caps from this fleet, not gemma-only rules, and not keys you must set on 0.9.22. The machine list is in [`guide.md`](./guide.md).

Doctor on 0.9.22 said unknown profiles keep default **4**, and **8** for automatically paged models on M5 Pro, Max, and Ultra. The working Studio config did not pin `engine_v2_max_concurrent`. Leave automatic until the box is cool and full priority. Do not jump to 16–24 because you have RAM.

See [Disclaimer](#disclaimer): RAM alone is not enough; bandwidth + network matter.

### Fan control (any supported Mac)

Opt-in experimental CLI — **not** in `provider.toml`. Not laptop-only; portables often need it most under continuous serve.

```bash
darkbloom fan diagnose          # check compatibility first
darkbloom fan enable            # admin password; defaults ~80% speed @ 45°C GPU
darkbloom fan status
# darkbloom fan disable         # restore macOS automatic
```

Full detail: [`guide.md` — Fan control](./guide.md#fan-control-thermals--all-apple-silicon-providers).

---

## 05 · Earnings-max profile

![Earnings-max profile](./assets/cards/05-earnings-profile.png)

Reliable baseline knobs:

- `idle_timeout_mins = 0` (`darkbloom idle keep-loaded`)
- `mtp_mode = "auto"` (do not use the old `beta enable mtp` step)
- `auto_update` + `auto_restart` on
- `enabled_models` pinned to catalog ids your chip can run
- concurrency left automatic (see §04)
- macOS 27: App Attest authorized, not a new MDM profile
- thermals: `darkbloom fan diagnose`, then optional `fan enable`  

Example TOML: [`config/examples/provider-earnings-max.toml`](./config/examples/provider-earnings-max.toml)

---

## 06 · Good vs unhealthy-busy

![Good vs unhealthy-busy](./assets/cards/06-good-vs-bad.png)

| Good | Bad |
|------|-----|
| doctor green | high concurrent + timeouts |
| warm list correct | advertised but not warm |
| full priority | restart thrashing |
| tokens climb without restarts | reduced priority for hours |
| boring thermals | looks busy, earns less |

If priority drops: cut concurrent, stay warm, **wait** — don’t config-thrash.

---

## 07 · Daily 10 minutes

![Daily 10 minutes](./assets/cards/07-daily-ops.png)

1. **status** — daemon online, warm + trust  
2. **doctor** — all green  
3. **warm list** — the models you meant to serve, and the idle ones are not pretending to earn  
4. **console** — 24h tokens & priority  

After an update: doctor → confirm warm → leave it alone while cache rebuilds. Don’t thrash restarts.

CLI = health **now**. Console = 24h / lifetime tokens & $.

---

## Quick commands

```bash
darkbloom status
darkbloom doctor
darkbloom verify
darkbloom update --check-only
darkbloom idle status
darkbloom restart
darkbloom models catalog
darkbloom models list
darkbloom fan diagnose
darkbloom fan status
```

---

## Repo layout

```text
README.md                 ← this page (cards + short text)
guide.md                  ← full operator guide
AGENTS.md                 ← rules for coding agents pointed at this repo
assets/cards/01…07.png    ← exported panels
assets/*.tldraw           ← editable source board
config/examples/          ← sample provider.toml
```

---

## Contributing

Catalog mins, model IDs, and best concurrent settings shift in alpha. PRs welcome when you’ve **verified** a change with `darkbloom models catalog` / live traffic — not calculator vibes.

Please **do not** open issues asking for earnings predictions or hardware buy advice beyond what’s documented.

---

## License

MIT — see [LICENSE](./LICENSE). Content is independent provider notes, shared as-is. Not affiliated with Darkbloom or Eigen Labs.
