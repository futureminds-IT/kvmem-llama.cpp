# futureminds patch series 0005-0009 (base: upstream tag v0.17.0)

Applied on top of `kvmem/kvmem-llama.cpp` tag **v0.17.0**
(commit 37805065ab5b2e311cb556005eaae40b3476416f), in this order:

| Patch | Purpose |
|---|---|
| 0005-sleep-idle-seconds | `--sleep-idle-seconds N`: unload the model from VRAM after N idle seconds |
| 0006-sleep-drop-conversation-stores | drop multi-conversation stores and disk snapshots on unload (required with `--kvmem-conversations > 1`; otherwise every post-wake request returns HTTP 503 "session switch requires a committed active conversation") |
| 0007-sleep-wake-reload-retry | retry the wake reload for up to 60 s instead of returning 503 immediately |
| 0008-capture-staging-slot-fit | reuse an idle D2H capture slot that already fits (upstream issue #55). Verified A/B on RTX 5060 Ti 16 GB: unpatched = 2x "KVMEM D2H gpu staging: out of memory" plus slot collapse; patched = 0 OOM, slots stable at 32+224 MiB |
| 0009-kvmem-pool-guard | reject a KVMem pool overrun (budget + gen_reserve cells) with HTTP 400 instead of cell exhaustion -> CUDA OOM -> ggml_abort (upstream issue #56). Tolerance via `KVMEM_POOL_GUARD_MARGIN_PCT`, default 150 % |

Build notes: `scripts/apply-patches.sh`, then `git apply` 0005 .. 0009 in that order, then
`CMAKE_CUDA_ARCHITECTURES=120a-real scripts/build-cuda.sh`.
The applied-code commits live on branch `dirk-patches-v0.17`.
Note: upstream's own `patches/` directory contains an unrelated `0005-hip-rdna2-...` patch,
so avoid globbing `0005*`.
