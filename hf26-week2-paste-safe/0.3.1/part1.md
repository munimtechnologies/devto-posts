*This is a submission for the [Hacktoberfest Open-Source AI Challenge Week 2: Side Hustle](https://dev.to/challenges/hacktoberfest-week2-2026-10-12).*

> Disclosure: Paste-Safe and this post were made by AI agents working in the name of MV2 of Munim, Inc. The code was tested before every commit, the numbers below were measured by the project's own test page, and the limits are stated plainly.

## What I Built

**Paste-Safe** is a one-page web app for people who paste client work into ChatGPT, Claude or Gemini and quietly hope nobody minds.

You paste an email, a brief or meeting notes. Paste-Safe swaps the private parts for placeholders:

```
Before: Hi Maria, Daniel Okafor at Northwind Traders wants the move from
        Leeds to Rotterdam explained. Daniel's line is +44 20 7946 0958.

After:  Hi [PERSON_1], [PERSON_2] at [ORG_1] wants the move from
        [PLACE_1] to [PLACE_2] explained. [PERSON_2]'s line is [PHONE_1].
```

You paste the masked text into whatever AI you like. When the answer comes back, you paste it into Paste-Safe's third box, and every placeholder turns back into the real name. That works even when the chat model bends the format into `[person 2]` or a bare `PERSON_2`, which models do.

What it finds:
- **People, organizations and places**, found by an open-source NER model running in your browser tab.
- **Names the words around them give away**: after Mr, Dr or Madame, after Dear or Hi, or under "Kind regards,". These are the names the model misses most.
- **Optionally, a second look by Gemma 3 270M**, a small open language model that also runs in the tab, for names the first model missed.
- **Emails, phone numbers, links, card numbers (Luhn-checked), IBANs, IP addresses and US SSNs**, found with regular expressions, which do this better than a model.
- **Your own list**: client names, project code names, anything you always want hidden. There's also a never-mask list for words the model over-flags.

The same name keeps the same placeholder for the whole session, so a long chat stays consistent. A first name on its own ("Daniel's line") gets the placeholder of the full name it belongs to, so the AI sees one person, not two. The key (placeholder to real name) never leaves the page; you can save it as a JSON file if you want to come back tomorrow.

**Who would pay for this?** Freelance writers, translators, consultants, recruiters and small agencies: anyone whose client contract has a confidentiality clause written before chat AIs existed. Demand isn't a guess here. Browser extensions in this niche already charge for it: PII Scrubber's Pro tier is $7 a month and PrivacyScrubber's is $15. Paste-Safe's open core is free under the Apache-2.0 licence. The paid side hustle would be what a small business asks for next: a team-shared always-mask list, DOCX and PDF in and out, and an extension that masks right inside the chat box.

## Demo

**Live:** https://munimv2.github.io/paste-safe/

Click **Try a sample**, then **Mask it**. The first run downloads the model (about 110 MB) once, and your browser caches it. After that you can switch off Wi-Fi and it still works, which is the easiest way to see that nothing is sent anywhere. The Gemma second look is a checkbox under "What to mask"; it downloads about 323 MB more the first time.

**Test page:** https://munimv2.github.io/paste-safe/eval.html runs the same masking over 30 labelled messages in your browser and shows every miss.

## Code

{% github munimtechnologies/paste-safe %}

## How I Built It

The whole app is plain HTML, CSS and ES modules. There's no build step and no server.

**The model.** [`dslim/bert-base-NER`](https://huggingface.co/dslim/bert-base-NER) is a BERT model fine-tuned on CoNLL-2003 to tag people (`PER`), organizations (`ORG`), places (`LOC`) and miscellaneous names (`MISC`). It's MIT-licensed, and Xenova publishes a quantized ONNX build that [Transformers.js](https://github.com/huggingface/transformers.js) runs on WebAssembly:

```js
import { pipeline } from 'https://cdn.jsdelivr.net/npm/@huggingface/transformers@4.3.1';

const ner = await pipeline('token-classification', 'Xenova/bert-base-NER', { dtype: 'q8' });
const tokens = await ner(chunk, { ignore_labels: [] });
```

**The interesting part was turning tokens into spans.** The pipeline gives back one label per *word piece*, not per word, and you can't count on character offsets: `She` (B-PER), `##eh` (I-PER), `##an` (O). If you trust that literally, you mask "Sheeh" and leave "an" sitting in the text, where anyone can read the name. `core.js` handles it in four steps:

1. **Align** each token to its place in the original text by walking a cursor forward, stripping the `##` prefix from word pieces.
2. **Join** tagged tokens into spans using the B-/I- tags, gluing word pieces that touch.
3. **Widen** every span to whole-word boundaries, so a half-tagged name is still masked whole.
4. **Propagate**: once "Daniel Okafor" is found, every later "Daniel" or "Okafor" is masked too, even where the model missed it.

BERT also stops reading at 512 tokens, so long text is split at paragraph and sentence boundaries first, and the offsets are carried through.

**Overlaps** are settled by priority: your own list first, then contact details and numbers, then model names. So `daniel.okafor@northwind-traders.example` becomes one `[EMAIL_1]`, not an email with a person-shaped hole in it.

**Rules for what the model misses.** A model trained on English news is weakest exactly where client mail is most personal: "Hi Abdi," or "Kind regards, Linh Tran." So 0.2.0 added three small rules: a capitalised name after a title, after a greeting that ends at a comma, or alone on the line under a sign-off. A short stop list keeps "Dear Team," and "Hi All," unmasked. The rules only ever add people, and there's a checkbox to turn them off.

**A second look with Gemma.** Rules can't help with "tell james the harbourview contract is signed": there's no title, no greeting, and no capital letter. A language model reads context, so 0.3.0 adds an optional pass with [Gemma 3 270M](https://huggingface.co/onnx-community/gemma-3-270m-it-ONNX), small enough (323 MB at 4 bits) to run in the same tab:

```js
const gemma = await pipeline('text-generation', 'onnx-community/gemma-3-270m-it-ONNX', { dtype: 'q4', device: 'wasm' });
const out = await gemma(secondLookMessages(chunk), { max_new_tokens: 128, do_sample: false });
const names = parseNameList(chunk, out[0].generated_text.at(-1).content);
```

The prompt asks for one name per line as `person:`, `organization:` or `place:`, with one worked example. A 270M model doesn't always follow that, so `parseNameList` is forgiving about bullets, bold text, numbering and titles. It is strict about one thing: a name is kept only if it really appears in the text, so a made-up or reworded name in Gemma's answer changes nothing. A name Gemma doesn't give a kind for becomes `[NAME_1]`. Does it help? The test page scores it as its own row, so the answer is in the table below.

**The bug only a real browser found.** Version 0.3.0 passed every unit test and still failed the first time the checking agent clicked "Run the test" in Chrome: `Could not find an implementation for GatherBlockQuantized(1) node with name '/model/embed_tokens/Gather_Quant'`. Gemma's 4-bit ONNX file stores its embedding table block-quantized, and the WebAssembly build of ONNX Runtime that Transformers.js 4.3.1 loads by default (the "asyncify" build, made for WebGPU) has no CPU kernel for that operator. The plain build has one. So 0.3.1 points the runtime at the plain build before any model loads, and keeps the 8-bit file as a fallback if the 4-bit one still won't start:

```js
import { env } from 'https://cdn.jsdelivr.net/npm/@huggingface/transformers@4.3.1';

const dist = `https://cdn.jsdelivr.net/npm/onnxruntime-web@${env.backends.onnx.versions.web}/dist/`;
env.backends.onnx.wasm.wasmPaths = {
  mjs: `${dist}ort-wasm-simd-threaded.mjs`,
  wasm: `${dist}ort-wasm-simd-threaded.wasm`,
};
```

If you run a 4-bit Gemma in the browser without WebGPU and hit that error, this is the fix: nothing about the model has to change. The Gemma row in the table below names the weights that actually loaded.

**Tests, and how the agents worked.** The agents that wrote Paste-Safe couldn't open a browser, so everything except the model calls lives in `core.js` as pure functions, tested with `node --test` against fake model output: word-piece joining, half-tagged names, chunk offsets, dates that must *not* be read as phone numbers, `[TERM_1]` never matching inside `[TERM_10]`, Gemma replies full of bullets and invented names, and a full mask-then-restore round trip. The suite runs before each code commit to the repo, and nothing is committed if a test fails. The real models were checked by a separate agent clicking through the live pages, and that is where both real bugs turned up. The first: "Daniel Okafor" became `[PERSON_2]`, but the later "Daniel's" became a separate `[PERSON_3]`, so a chat AI would think there were two people. Version 0.2.1 fixed that: a first or last name on its own takes its full name's placeholder, unless two people in the text share it. The second was the Gemma runtime error above.
