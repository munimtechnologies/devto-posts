## Why Does Open Innovation Matter?

Here it isn't a preference; it's the only design that works.

**You can't use a cloud API to find the secrets you don't want to send to a cloud API.** A hosted NER endpoint would mean shipping the client's names to a third party in order to hide the client's names from a third party. The detector has to run where the text already is: on your machine, in the tab, with weights you can download and inspect.

Openness buys three more things:

- **Verifiability.** "We process everything locally" is a claim. An MIT repo plus open-weight models plus a network tab you can watch is evidence. For a privacy tool, that difference is the product. The same goes for accuracy: the test set, its labels and the scoring code are all public, so the numbers above can be re-run by anyone instead of taken on trust.
- **Zero marginal cost.** Every mask runs on the user's CPU. A free tier costs nothing to serve, so the free version can stay genuinely useful instead of being a five-uses-a-day teaser.
- **Swappable parts.** Each model is one line. A multilingual NER model, a bigger Gemma, or one tuned on legal text can drop in without touching the masking logic, and the test page will say whether it does better. That is how the Gemma second look was added in the first place.

Where the open approach is weaker, honestly: a 2018-era BERT trained on news misses unusual names, lower-case names and non-English text, and a 270M Gemma is a small model with its own misses and false alarms. A large closed model would catch more, but you would have to send it the text, which defeats the point. The always-mask list is the answer for the names that matter most: add a client once and it is masked every time.

## Prize Categories

**Gemma**: Gemma 3 270M runs in the browser tab (Transformers.js, 4-bit ONNX) as Paste-Safe's optional second look for names, and the test page measures what it adds.

---

*Made by MV2 of Munim, Inc., with AI agents working in his name. Paste-Safe lowers the risk of pasting client text into AI tools; it does not make text anonymous and is not legal advice.*
