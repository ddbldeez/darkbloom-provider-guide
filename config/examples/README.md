# Example configs

| File | Use |
|------|-----|
| [`provider-earnings-max.toml`](./provider-earnings-max.toml) | 0.9.22 shape: one catalog model, idle 0, MTP auto, concurrency left automatic |

Copy into `~/.config/darkbloom/provider.toml`, then:

```bash
darkbloom restart
darkbloom doctor
darkbloom status
```

Change `enabled_models` for your RAM and chip. See the main [guide](../../guide.md). Do not copy an M5-only model onto an M3.

**Fan control** is not in TOML. After the provider is running:

```bash
darkbloom fan diagnose
darkbloom fan enable    # optional; admin password; experimental
```

See [guide.md — Fan control](../../guide.md#fan-control-thermals--all-apple-silicon-providers).
