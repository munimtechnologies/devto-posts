Live page with the same code: https://munimv2.github.io/gbq-repro/ (source: https://github.com/munimv2/munimv2.github.io/tree/main/gbq-repro). Open cases A, B and C in turn. Each is a fresh page load, because ONNX Runtime picks its WASM build once per page.

Minimal code:

```html
<script type="module">
import { pipeline, env } from 'https://cdn.jsdelivr.net/npm/@huggingface/transformers@4.3.1';

// Workaround: uncomment to load the plain build instead of asyncify; then the q4 model loads.
// const dist = `https://cdn.jsdelivr.net/npm/onnxruntime-web@${env.backends.onnx.versions.web}/dist/`;
// env.backends.onnx.wasm.wasmPaths = {
//   mjs: `${dist}ort-wasm-simd-threaded.mjs`,
//   wasm: `${dist}ort-wasm-simd-threaded.wasm`,
// };

const generator = await pipeline('text-generation', 'onnx-community/gemma-3-270m-it-ONNX', {
  dtype: 'q4',
  device: 'wasm',
});
const out = await generator([{ role: 'user', content: 'Reply with one word: hello' }], {
  max_new_tokens: 8,
  do_sample: false,
});
console.log(out[0].generated_text.at(-1).content);
</script>
```

Steps:

1. Serve the snippet over https (or open the live page) in Chrome.
2. With the workaround commented out, session creation fails with the `GatherBlockQuantized(1)` error.
3. Uncomment the workaround, reload: the model loads and replies.

Results from the live page:

