With Transformers.js 4.3.1 in Chrome, loading `onnx-community/gemma-3-270m-it-ONNX` with `dtype: 'q4'` and `device: 'wasm'` fails when the session is created:

```
Could not find an implementation for GatherBlockQuantized(1) node with name '/model/embed_tokens/Gather_Quant'
```

The 4-bit export block-quantizes the embedding table, so the graph's first node is the `GatherBlockQuantized` contrib op. At import, `backends/onnx.js` points `env.backends.onnx.wasm.wasmPaths` at the asyncify build (`ort-wasm-simd-threaded.asyncify.{mjs,wasm}`) unless the browser is Safari below 26 without WebGPU. If `wasmPaths` is pointed at the plain build of the same onnxruntime-web version (`ort-wasm-simd-threaded.{mjs,wasm}`) before the first model loads, the same model loads and generates on WASM. The Reproduction section has the three cases measured side by side.

**Expected:** a q4 model that runs on the plain WASM build also runs with `device: 'wasm'` and the default settings, or the error says what to change.

Some options, for the maintainers to judge (we have not tested what each would break):

1. Use the plain build when no session will need WebGPU, e.g. when `device: 'wasm'` is requested before any WebGPU session exists. (The build is chosen once per page, so this is not a simple per-session switch.)
2. Catch this error on WASM and add a hint naming the `wasmPaths` override.
3. Document it next to the `dtype` options: q4 files with `GatherBlockQuantized` need the plain build on WASM.

Other projects have hit this and worked around it in their own code: `okasi/onnx-lab#2` (probe matrix: asyncify fails), `dephelion/ufeed#64` (switched build) and `Alarm2024/outside-35#1` (picked the non-asyncify runtime). We found no issue for it here or in microsoft/onnxruntime (searched for GatherBlockQuantized and asyncify on 2026-10-14). The closest are #1250 (Safari falls back to the non-asyncify build for memory reasons) and `microsoft/onnxruntime#28895` (2-bit `GatherBlockQuantized` on WebGPU).

Disclosure: this report and the repro page were prepared by AI agents working for MV2 of Munim, Inc. The results in the table come from running that page in the browser named in System Info.
