# categorical-correctness

A 50-line probe that measures whether your LLM is *functorial*. Run it on Colab in 45 seconds; the output is a single number — the **functor-law violation rate** — that tells you what fraction of compositional queries the model confabulated through.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/coproduct-opensource/categorical-correctness/blob/main/categorical_compositionality_probe.ipynb)

> **New here?** Read **[INTRO.md](./INTRO.md)** first — a patient walkthrough of the problem, the existing measurement toolkit, and what the categorical move actually buys you. No prior category theory required. Then come back here for the technical pitch.

## Who this is for

1. **Practicing engineers** shipping LLM features into production, who are tired of "feels-fine" eval loops and want a 5-minute correctness signal that means something specific.
2. **Anyone trying to reduce confabulation** — particularly the failure mode where the model produces two locally-plausible answers that *don't compose into a globally-coherent one*. That failure has a name in category theory; this notebook makes it measurable.
3. **People forming an opinion** on what AI correctness should look like in 2026. There is no single answer yet. The categorical lens is one strong option, this notebook lets you kick the tires on it in an afternoon, and the README links out to the load-bearing literature so you can disagree productively.

## What it measures

For each entity `x`, the probe asks the model three questions:

- `g(x)`        — e.g. "What is the capital of France?" → "Paris"
- `f(g(x))`     — e.g. "What language is primarily spoken in Paris?" → "French"
- `(f ∘ g)(x)`  — e.g. "What language is primarily spoken in the capital of France?" → should also be "French"

If the model is a faithful functor, the chained answer `f(g(x))` and the composed answer `(f ∘ g)(x)` agree. Disagreement means the model produced an answer to one form that contradicts its own answer to the other — a *categorical confabulation*. The notebook reports the rate at which this happens across 8 hand-picked triples.

The categorical claim isn't that the model needs to know any of these facts. It's that whatever the model does know, it should *know consistently across phrasings*. Failure of this is structural, not stochastic.

## Why it matters

Most factuality benchmarks score *outputs*. This one scores *the relationship between two outputs*. That's a strictly harder constraint, and it's the constraint that any system claiming to compose facts (RAG over chained retrieval, agentic tool use, multi-hop reasoning) has to satisfy in order to be trusted.

The single-number readout has a clean interpretation:

- **violation rate = 0**: the model agrees with itself on every compositional query. Necessary but not sufficient for correctness.
- **violation rate > 0**: each violation is a verifiable factual contradiction, by the model's own lights, on a query the user is plausibly going to ask. These rows are the most actionable bug reports an LLM eval can produce — every one is a confabulation with a receipt.

Empirically, frontier reasoning models have been getting *worse* on long-chain factuality (o3-series at 33–51% hallucination on PersonQA / SimpleQA, vs ~16% for o1; see [Scott Graffius's analysis](https://www.scottgraffius.com/blog/files/ai-hallucinations-2026.html)). The functor-law framing predicts this: longer compositional chains have more places to drop a type. This notebook is a way to measure that prediction directly on whatever model you care about.

## How to run

**One-click Colab**: click the badge above. You'll need an Anthropic API key (~$0.20 in spend at default settings). Drop it in `Tools → User data` as `ANTHROPIC_API_KEY` and the notebook reads it automatically.

**Locally**:

```bash
pip install anthropic pandas
jupyter lab categorical_compositionality_probe.ipynb
```

The notebook runs three model calls per triple (24 total) plus 8 judge calls = 32 API calls, plus 8 sanity-check judge calls = 40 if you keep the symmetric-vs-asymmetric diagnostic enabled.

## What's interesting about how it's built

Two design choices worth pointing at because they're load-bearing for taking the result seriously:

1. **Asymmetric judge by default.** The notebook tests `claude-haiku-4-5` and judges the equivalence of two outputs with `claude-sonnet-4-6`. v1 used the same model for both and discovered, the hard way, that a model judging its own outputs has a systematic self-agreement bias that *understates* its own hallucination rate. The asymmetric judge cuts that confound. The notebook keeps the symmetric judge around for one cell so you can see the gap directly — the difference IS the bias term.
2. **The judge gets two answers, not the truth.** The probe doesn't need a ground-truth answer for any query. It only checks self-consistency across two phrasings. That makes it cheap to extend to any domain where you can write `(g, f)` template pairs — no labeled dataset, no expert annotators, no benchmark contamination concerns.

## Limitations

Listed plainly because the next move depends on which limitation you care about most:

- **8 triples is small.** The headline number on this sample has wide confidence intervals. Real benchmarks would pull hundreds of triples from Wikidata via SPARQL — any entity + two SPARQL-derivable predicates gives you a free `(g, f)` pair.
- **The judge is still an LLM.** A judge with chain-of-thought + a third "I DON'T KNOW" option (Lawvere-Tierney `j` if you want the categorical name) cuts both false positives and false negatives. The next notebook in this repo will do that.
- **No abstention modeling on the system under test.** A model that says "I don't know" to one of `g(x)`, `f(g(x))`, `(f ∘ g)(x)` shouldn't count as a violation; the current judge probably calls these inconsistencies. Easy to fix; not yet fixed.
- **Anthropic-only.** Swapping in OpenAI / Gemini / DeepSeek is a 5-line change to the `ask` helper and would let you do cross-model comparisons. PRs welcome.

## What the categorical claim actually is

If you want the long version, read the [companion essay](https://coproduct.one/blog/categorical-correctness). Short version: a category is a set of objects with a typed composition rule (`g: A → B` and `f: B → C` compose to `f ∘ g: A → C` only when types match). A functor between categories preserves composition: `R(f ∘ g) = R(f) ∘ R(g)`. Treating an LLM's question-answering as a (claimed) functor from queries to answers gives you exactly one law to check, and that law is what this notebook checks.

If the model is a functor, it's structurally consistent. If it isn't, every violation is a fact it invented during composition that it would have rejected if asked directly.

## Resources for forming an opinion

In rough order of return-on-time:

- [Floridi, Jia, Tohmé — *A Categorical Analysis of Large Language Models and Why LLMs Circumvent the Symbol Grounding Problem*](https://arxiv.org/abs/2512.09117) (December 2025). The current best formal account of *why* LLMs hallucinate, in categorical terms.
- [Fong & Spivak — *Seven Sketches in Compositionality*](http://brendanfong.com/fong_spivak_an_invitation.pdf) (free PDF). Read chapter 1 for the vocabulary; it's enough to follow the rest of the AI-correctness literature in this lens.
- [Google DeepMind FACTS Grounding](https://deepmind.google/blog/facts-grounding-a-new-benchmark-for-evaluating-the-factuality-of-large-language-models/) + [the dataset on HuggingFace](https://huggingface.co/datasets/google/FACTS-grounding-public). The most rigorous public factuality benchmark as of late 2025; 1,719 examples, breaks factuality into 4 dimensions instead of collapsing to one score.
- [HalluLens (ACL 2025)](https://arxiv.org/abs/2504.17550). The first benchmark that distinguishes *intrinsic* (training-data inconsistency) from *extrinsic* (input-context deviation) hallucination — categorically these are different bugs requiring different fixes.
- [CatColab](https://topos.institute/work/catcolab/) (Topos Institute). Collaborative formal categorical modeling, currently at v0.5: Sandpiper. The fastest way to *play* with compositional models without writing them yourself.

## License

MIT.

## Maintainers

- [@brandon-coproduct](https://github.com/brandon-coproduct) — `brandon@coproduct.one`

PRs welcome, especially for: more triples, more model providers, the abstention-aware judge, the Wikidata SPARQL extractor.
