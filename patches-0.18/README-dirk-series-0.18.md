# futureminds patch series for v0.18 (base: upstream tag v0.18.0-llamacpp-rc1)

Applied on top of `kvmem/kvmem-llama.cpp` tag **v0.18.0-llamacpp-rc1**.
**Apply in this order** (0008 and 0009 first, 0010 last):

| Patch | Purpose |
|---|---|
| 0008-capture-staging-slot-fit | reuse an idle D2H capture slot that already fits (upstream issue #55). Unchanged from the v0.17 series. |
| 0009-kvmem-pool-guard | reject a KVMem pool overrun (budget + gen_reserve cells) with HTTP 400 instead of cell exhaustion -> CUDA OOM -> ggml_abort (upstream issue #56). Ported to the v0.18 guard site. Tolerance via `KVMEM_POOL_GUARD_MARGIN_PCT`, default 150 %. |
| 0010-sleep-idle-seconds | v0.17 series 0005/0006/0007 in one patch: `--sleep-idle-seconds N` (unload/reload), drop the conversation stores on unload, retry the wake reload for up to 60 s. |

0010 is one patch instead of three because v0.18 moved the entire load path into
`main()` (model, context, lanes, conversation arming) and introduced
`ServerState`/`LlamaEngineState` with execution scopes, so the three concerns no
longer split into independent hunks. It must be applied **after** 0008/0009:
on v0.18 the guard hunks live in `tools/llama-kvmem-server.cpp`, the same file
this restructure rewrites.

Two pitfalls specific to v0.18, neither present in the v0.17 series:

* `main()` holds `kvmem_execution_scope startup_execution` for the whole startup,
  so adapter queries during startup see the execution state. The wake path is an
  HTTP worker thread and must establish that scope itself, otherwise
  `llama_kvmem_store_swap_supported()` answers for an empty execution state, the
  conversation arming fails (`KVMEM session disk cache requires supported session
  switching` right after `loading model`) and the reload dies half way.
* the idle monitor and the request path must be serialized: the monitor holds
  `sleep_mu` while unloading and the request takes the same lock before counting
  itself in, otherwise an arriving request reads `st.ctx`/`st.vision` while the
  monitor frees them (SIGSEGV on the second idle cycle). `st.mu` is not usable:
  v0.18 takes it only in the slot guard, not across the request. The unload also
  calls `llama_driver_clear(lane)` **before** `llama_free(ctx)`.

Verification (RTX 5060 Ti 16 GB, Qwen3.8-27B IQ3_S, budget 32768 + gen_reserve
16384, `--sleep-idle-seconds 45`): 4 idle/wake cycles including two ~58k-token
requests -- VRAM 15050 -> 1074 MiB on unload, wake in ~5 s with a correct
completion, process alive after every cycle, no log errors. The earlier v0.18
regression (`native layout scratch unavailable` at ~58k) did not appear with
budget 32768: 58k -> HTTP 200.

Build: `CMAKE_CUDA_ARCHITECTURES=120a-real scripts/build-cuda.sh` inside a
container that has the GPU (the test targets link against `libcuda.so.1`).
