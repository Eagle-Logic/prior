# Benchmarks: Capabilities & Limitations

An honest account of what PRIOR does, what it doesn't, and how we measured it. Every number below
comes from **matched-pair tests**. That means the same model, seed, and harness, run with steering off
and on, on named models and hardware, with an independent judge where noted. We publish provenance with every
figure, and **we don't publish numbers we can't defend**.

## How to read these results

PRIOR is two parts, and it's worth keeping them separate when you read any number:

- **The actuator**: the golden-layer steering that forces the outcome (a refusal, or a grounded
  fact) *once a decision has been made*.
- **The gate**: the detector that decides *whether* to act on a given request.

The short version: **where the gate routes a request to RED, the actuator does its job essentially
perfectly.** The frontier of ongoing work is the gate's *recall*, meaning catching more phrasings of
the same intent. That work is on the gate, not the actuator. We call it out explicitly in the limitations.

## How we measured

- **Fleet:** twelve models from 1B to 14B, across six architecture families: Llama-3.2, Qwen3
  (including an abliterated build), Gemma-3, Gemma-4, Mistral-Nemo, and SmolLM3.
- **Hardware:** a single NVIDIA RTX 3060 (12 GB). Every fleet result is reproduced per model.
- **Judging:** safety is scored by the independent **cais/HarmBench classifier**, with an independent
  LLM-judge cross-check (**93.6% agreement, n=800**) that rated our reported figures as *conservative
  floors*. In other words, the real numbers are likely better, not worse.

---

## Capabilities

### Safety: the actuator flips what the gate catches

| Result | What it means |
|---|---|
| **100%** of gate-routed harmful completions refused (**79/79**, cais-judged, across 11 models) | When the gate fires, the intervention holds, and it's independently verified. |
| Mistral-Nemo-12B attack-success **44.6% → 28.9%** | On the highest-headroom model, PRIOR roughly halves the residual attack surface. |
| **96%** jailbreak recovery via span-scoring, at **0** benign false-blocks | Holds up against obfuscated/jailbroken phrasings without taxing legitimate traffic. |
| Generalization to unseen attacks: **AUC ~0.93** | Coverage converges on a shared harm manifold. It is not endless string-chasing. |

> **Honest framing:** most *per-model* attack-success drops are small (1–7pp), because modern instruct
> models already refuse most obvious harm at baseline. PRIOR's safety value is largest on high-headroom
> or abliterated models where the base guardrails are weak. And everywhere, it flips **100%** of what
> the gate routes to RED, deterministically, regardless of the model's own inclination.

### Zero capability tax when no policy fires

When a request isn't in policy scope, PRIOR is a no-op. Not "a small delta," but byte-for-byte the
base model.

| Result | What it means |
|---|---|
| **GSM8K byte-identical across all 12 models** (exact-match 1.0), validated to Qwen3-14B (87%) | Math reasoning is untouched by steering. |
| **MMLU 67.25%**, baseline = steered, Δ 0 | Knowledge is untouched. |
| **HumanEval pass@1 59.8%**, byte-identical steered vs base | Code generation is untouched. |
| **0 induced over-refusals** (XSTest, 12 models, exact-match 1.0) | PRIOR does not make the model more prudish on safe prompts. |
| Overhead **+0.0%** (no policy fires) · **+0.2%** (intervention fires) | Negligible cost, measured on saturated batches. |

### Contextual grounding & anti-poison

| Result | What it means |
|---|---|
| Under poisoned retrieval, regression to the false value **eliminated across all 4 models tested** (3B–8B) | A grounded value holds even when a source in the context contradicts it. |
| Most-vulnerable Qwen3-8B faithfulness **50% → 77% (+27pp)** | The more capable the model, the more it "reasons into" a confident lie, and grounding closes that gap. |
| A grounded fact costs **0 prompt tokens** at query time | Ground a verified value in the model's latent state without spending context window. |

---

## Limitations

We'd rather you learn these here than in production.

### The current frontier is gate recall, not the actuator

Where the gate routes a request to RED, the actuator refuses it ~100% of the time. The open work is
getting the **detector to catch more phrasings of the same intent**. The shipping detector can under-fire on
short or novel phrasings that lack obvious trigger language. Two levers you control soften this:

- **Recall is a tunable operating point.** The default is precision-first (zero benign false-blocks),
  which trades away some recall. The **YELLOW confirm band** already catches most ambiguous cases
  without blocking anything, and you can tune thresholds or add your own detection packs.
- **A stronger intent encoder is the structural lever.** Detection accuracy is bounded by the embedding
  encoder; upgrading it lifts recall at every operating point. This is on the roadmap.

### Some attacks are out of scope by design

Take **absent-intent dual-use composition**, e.g. "fix these vulnerabilities," then diff the patches to
reconstruct the exploit. It carries no harmful signal in the prompt, the output, or the model's internal
state (the model is genuinely doing something benign). **No model-layer guardrail can detect it**, and
we don't claim to. Mitigating this is an operational/platform concern (usage-pattern monitoring,
provenance, attribution), not something any inference-time control solves. We state it rather than
imply coverage we don't have.

### What has and hasn't been independently judged yet

- Independent cais-judging is complete across the **1B–8B** fleet (Llama-3.1-8B fully judged).
  Independent judging on the **12–14B** models is **pending cloud hardware**.
- **Multi-turn:** payload-split attacks are hard-blocked; compliance-grooming is routed to
  human-confirm (YELLOW). Across-turn composition is validated at **≤8B**; above 8B is pending.
- **Faithfulness** figures are for **adversarial / poisoned retrieval**. On a clean single-source
  pipeline, modern models are already faithful, and that isn't where PRIOR earns its keep.

### Steering is calibrated per model

The steering geometry is a property of each model and must be calibrated before it's trusted. Models
in the registry are graded **certified** (calibration swept and validated) or **beta** (sensible
defaults, not yet swept). Bringing a new model means a one-time calibration; policies carry over
unchanged. See **[MODELS.md](./MODELS.md)**.

---

## Provenance

Judged model: **Llama-3.1-8B-Instruct** · Fleet (1B–14B): **Llama-3.2, Qwen3 (incl. abliterated),
Gemma-3, Gemma-4, Mistral-Nemo, SmolLM3** · Hardware: **NVIDIA RTX 3060, 12 GB** · Safety judge:
**independent cais/HarmBench classifier** · Cross-check: **independent LLM-judge, 93.6% agreement
(n=800), scored as conservative floors**. Frontier-scale (12–14B) independent judging and across-turn
composition above 8B are in progress.

---

<sub>Questions about methodology? Get in touch at https://eagle-logic.com · Full docs: https://eagle-logic.com/docs</sub>
