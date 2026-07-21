# Supported Models

PRIOR runs **one model per container**, and you supply the generation model as a **GGUF**. It's
mounted at runtime, not baked into the image. This page covers which models are supported, how PRIOR
grades them, and the exact command to fetch a recommended quickstart model.

> We **don't distribute models**. You pull them from Hugging Face by their standard filename. Keep the
> filename shown below so PRIOR's registry matches it to the right calibration.

---

## Why support is model-specific

The **gate** (policy detection) works on any model you load. **Steering is different**: it operates on
a model's internal geometry, so its quality is **model- and architecture-specific** and must be
validated per model. PRIOR ships a **registry** (baked into the engine) that maps each supported GGUF
to its calibrated steering parameters and grades it:

| Tier | What it means | Steering |
|---|---|---|
| **Certified** | Calibration swept and validated. | Reliable. This is the published behavior. |
| **Beta** | Runs, but calibration is default or not yet swept. | Weaker/less predictable; the gate still works. |
| **Unsupported** | Some architectures don't steer well (or at all) yet. | Not recommended for the safety use-case. |

The grade is the honest signal: a **certified** model steers cleanly, while an unsupported one
effectively doesn't (the gate still routes and flags, but forced refusals won't hold). **Don't assume an
arbitrary GGUF off Hugging Face will steer.** Start from the certified list below.

---

## Recommended models

Certified picks (4-bit `Q4_K_M`):

| Model | Approx. size | Grade | Best for |
|---|---|---|---|
| **Llama-3.2-3B-Instruct** | ~2 GB | Certified | The quickstart default: fast, modest GPU, cleanest steering. |
| **Qwen3-8B** | ~5 GB | Certified | More capable, still light on VRAM. |
| **Qwen3-4B-Instruct-2507** | ~2.5 GB | Certified | A strong 4B middle ground. |
| **Meta-Llama-3.1-8B-Instruct** | ~5 GB | Certified · reference | Our **primary judged model**; pick this to reproduce the cais-judged safety numbers. Some headline results are measured on other models (StrongREJECT and attack-success on Mistral-Nemo-12B, faithfulness on Qwen3-8B); each is named where it's quoted. |

Also certified: **Llama-3.2-1B** (~0.8 GB) for the smallest footprint, both **Mistral-Nemo-12B** builds,
the **Qwen3-Next** hybrid class (including the **27B Qwen3.5**), **Phi-4** / **Phi-4-mini**, **Ministral-3**,
a **DeepSeek-R1 14B distill**, and the **Gemma** builds at 12B and **31B** (**Gemma-3-12B**, **Gemma-4-12B**,
and **Gemma-4-31B**, the largest calibrated build). **Qwen3-14B**, the smaller **Gemma-3/4**
builds, **SmolLM3**, and the **abliterated Qwen3-4B-Instruct-2507** build are currently **beta** for
steering, so load them for the gate, not for reliable refusals. (The stock, non-abliterated
Qwen3-4B-Instruct-2507 in the table above is certified; the registry tags the two apart.) The full
roster (**24 builds across 9 architecture families**, 19 certified and 5 beta, each graded) lives at
**[eagle-logic.com/models](https://eagle-logic.com/models)**
(machine-readable at **[/model-registry.json](https://eagle-logic.com/model-registry.json)**).

---

## Fetch a model (one command)

Use the Hugging Face CLI to grab the **quickstart default** (Llama-3.2-3B, ~2 GB):

```bash
pip install -U "huggingface_hub[cli]"

huggingface-cli download bartowski/Llama-3.2-3B-Instruct-GGUF \
  Llama-3.2-3B-Instruct-Q4_K_M.gguf --local-dir ./models
```

Or the more capable Qwen3-8B (~5 GB):

```bash
huggingface-cli download Qwen/Qwen3-8B-GGUF \
  Qwen3-8B-Q4_K_M.gguf --local-dir ./models
```

> Repos occasionally move. If a download 404s, search Hugging Face for the model's official GGUF
> release or a reputable uploader (e.g. `bartowski/…-GGUF`), and keep the **exact filename** shown above
> so the registry match holds.

Then run it per the [Quickstart](./README.md#quickstart). Confirm PRIOR resolved it as **certified** via
the WebUI or the `/admin/model` endpoint (see the [docs](https://eagle-logic.com/docs/)).

---

## Bringing your own model

Not certified yet? PRIOR can **auto-calibrate** a new model in the WebUI. If you opt in, it can also share
the (non-PII) calibration result with us so we can verify and add it to the certified registry for
everyone. Details in the [docs](https://eagle-logic.com/docs/).

---

<sub>Full model & deployment reference: https://eagle-logic.com/docs/ · More at https://eagle-logic.com</sub>
