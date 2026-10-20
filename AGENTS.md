# AGENTS.md — Darkbloom Provider Guide

Instructions for coding agents (Claude Code, Cursor, Codex, Grok, etc.) working in this repo or pointed at it by a Darkbloom provider operator.

## What this repo is

- **Unofficial, independent** operator notes for running a [Darkbloom](https://www.darkbloom.dev/) **provider node** on Apple Silicon.
- Goal: maximize **finished tokens × success rate × hours online** — not theoretical concurrent max.
- **Not affiliated** with Darkbloom (darkbloom.dev) or Eigen Labs (eigenlabs.org). Not official docs.
- Content is experience-based (public alpha). Catalog mins, pricing, and demand change — **verify live** before asserting current facts.

## Repo map

| Path | Role |
|------|------|
| [`README.md`](./README.md) | Homepage: sequential cards (01–07) + short text |
| [`guide.md`](./guide.md) | Full operator guide (depth) |
| [`assets/cards/`](./assets/cards/) | Exported panel PNGs for README |
| [`assets/*.tldraw`](./assets/) | Editable graphics source |
| [`config/examples/`](./config/examples/) | Sample `provider.toml` baselines |
| [`AGENTS.md`](./AGENTS.md) | This file — agent operating rules |

**Canonical depth:** prefer `guide.md` for full rules; keep `README.md` as the scannable tour. Do not invent a third conflicting guide.

## When a user points you here

Typical asks:

1. **Apply the guide** to their Mac (edit `~/.config/darkbloom/provider.toml`, start flags, concurrent, models).
2. **Diagnose** no traffic / reduced priority / cold models / timeouts.
3. **Update** this repo (README cards, guide text, example toml) after they verify a change.
4. **Explain** model/RAM/concurrent choices without rewriting Darkbloom itself.

Default behavior:

1. Read `README.md` disclaimer + relevant card section, then `guide.md` for the same topic.
2. On their machine, prefer **live checks** over this repo’s last-verified snapshot:
   ```bash
   darkbloom --version
   darkbloom status
   darkbloom doctor
   darkbloom models catalog
   darkbloom update --check-only
   ```
3. Label claims: **observed** (this session) vs **from guide** (may be stale) vs **assumed**.
4. Give the **usable config or command first**, then short caveats.

## Non-negotiable product rules (do not invert)

These are load-bearing; if the live catalog contradicts them, **say so** and follow the catalog.

- **Allowlist catalog models your chip can run.** The August "only gemma + gpt-oss" rule is stale as of provider 0.9.22 (13 catalog models on 2026-10-20). Gemma is still valid. It is not the only model and not the top output sticker.
- **Do not advertise** `EigenLabs/Qwen3.8-27B-4bit-mtp` or `glm-5.3-flash` on M1–M4. Catalog requires `apple_m5` and `mlx_nax`.
- **Sticker is not demand.** A warm model with hours of idle time is not earning. Prefer `darkbloom switch` over a restart loop.
- **Concurrent:** 0.9.22 doctor keeps unknown profiles at 4 (8 for automatically paged models on M5 Pro, Max, and Ultra). Do not set the old global `engine_v2_max_concurrent` just because the August guide did. Do not jump to 16–24.
- **Dedicated earn boxes:** `idle_timeout_mins = 0` (`darkbloom idle keep-loaded`).
- **MTP:** `mtp_mode = "auto"`. Do not require `darkbloom beta enable mtp`.
- **macOS 27 trust:** App Attest. `trust_level: self_signed` can still be a paying node when the authorization path is `app_attest`. Do not install a new MDM profile. Do not run `darkbloom unenroll` unless the operator asked.
- **Reliability > throughput cosplay.** Deadlines and reduced priority lose tomorrow’s jobs.
- **Fan control (optional, all hardware):** experimental `darkbloom fan` — **not** a `provider.toml` key. Not laptop-only; portables often benefit most. Always `darkbloom fan diagnose` before enable; admin password for root helper.

## Hardware / network honesty (always surface)

When recommending hardware, concurrent, or dual-warm:

- Results **vary by hardware**.
- **RAM is not enough** — memory **bandwidth/throughput**, thermals, and sustained power matter.
- **Stable, high-speed internet is critical** — flaky links → timeouts → reduced priority.
- Guide concurrent bands and Mac examples are **starting points**, not guarantees.
- **No earnings promises** and no “buy this Mac and make $X.”
- Under heat / reduced priority: cut concurrent **and** consider `darkbloom fan diagnose` → optional `fan enable` (if supported). Do not invent TOML fan settings.

### Fan CLI (quick)

```bash
darkbloom fan diagnose
darkbloom fan enable --speed 80 --temperature 45   # admin prompt; defaults similar
darkbloom fan status
darkbloom fan configure --speed 75 --temperature 50
darkbloom fan disable
# darkbloom fan uninstall
```

See `guide.md` § Fan control for full caveats.

## Config paths (provider machine)

Typical Darkbloom provider install (may differ by version):

| Item | Path / command |
|------|----------------|
| Config | `~/.config/darkbloom/provider.toml` |
| State / logs | `~/.darkbloom/` (`daemon-state.json`, `provider.log`, …) |
| LaunchAgent | `~/Library/LaunchAgents/io.darkbloom.provider.plist` |
| Example baseline | `config/examples/provider-earnings-max.toml` in this repo |

**Gotcha:** `darkbloom start` can rewrite the LaunchAgent and strip custom env (e.g. `DARKBLOOM_MLX_MEMORY_RESERVE_GB`). After start/restart, verify the live process env if the guide’s MLX reserve advice is in play.

Backup `provider.toml` before edits. Prefer surgical TOML changes + `darkbloom restart` over thrashing restarts while recovering reputation.

## Diagnosis playbook (short)

| Symptom | First moves |
|---------|-------------|
| Zero jobs but doctor green | Quiet demand or reputation lag — wait; confirm console fleet; avoid restart thrash |
| Warm empty / `model_not_loaded` | Preload, idle 0, exact model id, local kick via `--local-endpoint` |
| Reduced priority / 120s deadlines | Cut concurrent; drop second model; cool machine; `fan diagnose` / optional enable; stay online |
| After update looks colder | Stay warm; cache epoch may rebuild — don’t thrash restarts |

Always re-check: `darkbloom doctor`, `darkbloom status`, console priority / 24h tokens (CLI counters **reset on restart**).

## Editing this repo

If the user asks you to change the guide for others:

1. Update **`guide.md`** for full accuracy first.
2. Mirror the same rules in the relevant **README** card section (keep card copy short).
3. If visuals change, re-export panels from the `.tldraw` source into `assets/cards/01…07.png` (or ask the user to export) so README images match text.
4. Bump **Last verified** date and provider version when claims were re-checked live.
5. Keep affiliation disclaimer intact in README Disclaimer + guide header.
6. A dated sticker snapshot is allowed when it names the endpoint and the check date. Do **not** add “expected $/mo” or tell anyone to buy a Mac for a rate.
7. Prefer PR-quality commits: complete sentences, why the change.

### Card order (README)

1. One rule  
2. What actually pays  
3. Models only  
4. Concurrent caps (+ hardware examples, no costs)  
5. Earnings-max profile  
6. Good vs unhealthy-busy  
7. Daily 10 minutes  

## What not to do

- Claim affiliation with Darkbloom / Eigen Labs or present this as official docs.
- Invent live pricing, floor rewards, or demand levels — fetch or tell the user to check console/catalog.
- Recommend non-catalog mega-models for public fleet serving “because they fit in RAM.”
- Raise concurrent without a reliability story.
- Commit secrets: auth tokens, API keys, `~/.darkbloom/auth_token`, payout details, personal machine hostnames if the user wants privacy.
- “Fix” earnings by restarting repeatedly.

## How users should invoke you

Examples they can paste:

```text
Read AGENTS.md and guide.md in this repo. Apply the earnings-max profile
to my Mac: check darkbloom status/doctor/catalog, then propose a minimal
provider.toml diff for my RAM and current version.
```

```text
Using this repo’s AGENTS.md + guide.md, diagnose why my provider is
hardware-trusted but getting no jobs. Prefer live CLI checks over guessing.
```

```text
Update this repo’s guide for provider version X after I verify these catalog
lines: … Keep README cards in sync; do not invent pricing.
```

## Tone

Be direct, practical, and honest about uncertainty. Challenge bad ideas (e.g. concurrent 24 on a Studio “because RAM”). Prefer evidence from the operator’s machine over vibes.
