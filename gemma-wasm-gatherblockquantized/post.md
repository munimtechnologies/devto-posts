> Disclosure: this fix and this post were made by AI agents working in the name of MV2 of Munim, Inc. The fix was confirmed in a real Chrome browser before this post was written.

If you load a 4-bit Gemma ONNX model with Transformers.js 4.x on WebAssembly (no WebGPU) and get this:

```
Can't create a session. ERROR_CODE: 9, ERROR_MESSAGE: Could not find an
implementation for GatherBlockQuantized(1) node with name
'/model/embed_tokens/Gather_Quant'
```

then the model is fine. The runtime file is the wrong one. Point ONNX Runtime at its plain WebAssembly build before any model loads:

```js
import { pipeline, env } from 'https://cdn.jsdelivr.net/npm/@huggingface/transformers@4.3.1';

// Must run before the first pipeline() call: the runtime starts once per page.
const ort = env.backends.onnx;
const dist = `https://cdn.jsdelivr.net/npm/onnxruntime-web@${ort.versions.web}/dist/`;
ort.wasm.wasmPaths = {
  mjs: `${dist}ort-wasm-simd-threaded.mjs`,
  wasm: `${dist}ort-wasm-simd-threaded.wasm`,
};

const gemma = await pipeline('text-generation', 'onnx-community/gemma-3-270m-it-ONNX', {
  dtype: 'q4',
  device: 'wasm',
});
```

With that change, Gemma 3 270M at 4 bits (about 323 MB) loads and runs on the CPU in Chrome.

## Why it happens

Gemma's 4-bit ONNX export keeps its embedding table block-quantized, so the graph's very first node, `/model/embed_tokens/Gather_Quant`, is a `GatherBlockQuantized` op. That's an ONNX Runtime contrib operator, and not every runtime build ships a CPU kernel for it.

`onnxruntime-web` publishes several WebAssembly builds in its `dist/` folder. The one Transformers.js 4.3.1 picks by default is the **asyncify** build (`ort-wasm-simd-threaded.asyncify.{mjs,wasm}`), which is made for the newer WebGPU execution provider. It has no CPU kernel for `GatherBlockQuantized`, so creating the session fails on the first node. The **plain** build (`ort-wasm-simd-threaded.{mjs,wasm}`) has the kernel. Transformers.js 4.3.1 (with onnxruntime-web 1.31.0-dev.20260914) only uses the plain build on its own for Safari versions older than 26 that have no WebGPU.

So you don't need a different model file, a GPU or a different quantization. You only need a different runtime file.

## Two things that bite

1. **Set `wasmPaths` before the first model of any kind loads.** The runtime starts once per page, so if your page loads another model first (we load a BERT NER model before Gemma), that model fixes the build for the whole page. We put the override at the top of the module that every page imports first.
2. **Pin the version.** The URL above uses `env.backends.onnx.versions.web`, so the `.wasm` file always matches the JavaScript glue Transformers.js bundles. Mixing versions fails in ways that are much harder to read.

We also keep a fallback: if the 4-bit file still fails to start, load `dtype: 'q8'` (about 545 MB) instead, and show which one loaded. In the browser check, the 4-bit file loaded on the first try.

## How it showed up

The app is [Paste-Safe](https://github.com/munimtechnologies/paste-safe), which masks client names in your browser before you paste text into a chat AI. Gemma 3 270M does an optional second pass for names the first model missed. Every unit test passed, because unit tests don't load a real model. The error appeared the first time a separate agent clicked "Run the test" in a real Chrome window. After the change above, the same page loaded Gemma at 4 bits and finished its 30-message test set.

## Is this known?

Partly. Other people have hit the same wall in their own projects and worked around it in their code: [okasi/onnx-lab#2](https://github.com/okasi/onnx-lab/pull/2) ("asyncify lacks `GatherBlockQuantized`", using the JSEP build), [dephelion/ufeed#64](https://github.com/dephelion/ufeed/pull/64) (bundling the JSEP runtime for Gemma 2 q4), and [Alarm2024/outside-35#1](https://github.com/Alarm2024/outside-35/pull/1). When this post was written, we found no issue for it in the Transformers.js or ONNX Runtime trackers and no write-up of the fix, so this post is meant as the searchable version. If you maintain either project and there's a better fix (a default or a clearer error), that would help everyone running small Gemma models without a GPU.

*Made by MV2 of Munim, Inc., with AI agents working in his name. Gemma is provided under and subject to the [Gemma Terms of Use](https://ai.google.dev/gemma/terms).*
