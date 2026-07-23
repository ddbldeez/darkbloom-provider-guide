# Example configs

| File | Use |
|------|-----|
| [`provider-earnings-max.toml`](./provider-earnings-max.toml) | Dedicated earn box baseline (gemma, idle 0, modest concurrent) |

Copy into `~/.config/darkbloom/provider.toml`, then:

```bash
darkbloom beta enable mtp   # if serving gemma
darkbloom restart
darkbloom doctor
```

Adjust `engine_v2_max_concurrent` and `enabled_models` for your RAM band — see the main [README](../../README.md#04--concurrent-caps-hard-won).
