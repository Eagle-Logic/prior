# PRIOR

**Policy-enforcing inference for open-weight LLMs.** Self-hosted, OpenAI-compatible, your prompts and model never leave your box.

PRIOR wraps an open-weight model (you supply the GGUF) and enforces *your* policy on every request.
It doesn't use a keyword filter. It works at the **residual-stream level**, where the constraint survives
obfuscation, role-play, and jailbreak wrappers. When a request is in-policy-scope it's a **byte-for-byte
no-op**; when it isn't, PRIOR steers the outcome deterministically, regardless of the base model's own
inclination.

- 🌐 **Website:** https://eagle-logic.com
- 📚 **Docs:** https://eagle-logic.com/docs/
- 🚀 **Free 30-day trial** (full capabilities, no node lock): https://eagle-logic.com/get-prior#trial
- ✉️ **Sales & enterprise:** sales@eagle-logic.com

> This repository is the public overview: what PRIOR is, how it measures up, and which models it
> supports. The engine ships as a container on GHCR (below); the source is licensed, not public.

---

## What PRIOR is

Two parts, worth keeping separate:

- **The gate** is a tri-state detector (GREEN / YELLOW / RED) that decides *whether* a request is in
  policy scope. GREEN passes through untouched; YELLOW routes ambiguous cases to human-confirm; RED
  triggers an intervention.
- **The actuator** is **golden-layer steering** in the residual stream that forces the outcome (a
  refusal, or a grounded fact) *once the gate has decided*. The steering direction is calibrated per
  model and runs either as a pre-distilled static vector or computed live per request, so the intervention is a
  property of the model's own geometry, not a bolt-on text filter.

It's **OpenAI-compatible** (`/v1/chat/completions`), so any OpenAI client works. PRIOR's telemetry
rides on `x-prior-*` response headers (zone, pack, injections, decision id) and its behavior is tuned
per-request via the `prior_options` vendor extension. Governance (packs, keys, audit) lives under a
role-gated `/admin` surface.

**Policy is a document, not code.** Ship your own policy by pasting a governance document; PRIOR
compiles it into a detection pack. See the [docs](https://eagle-logic.com/docs/).

---

## Quickstart

The trial image is public, no login needed. You supply a certified GGUF (see **[MODELS.md](./MODELS.md)**)
and your trial `license.bin` (emailed when you start a trial). This repo ships a ready-to-run
[`docker-compose.yml`](./docker-compose.yml) that brings up the **engine** plus the **console** (a web
dashboard to watch the gate, author policy, manage keys, and read the audit log):

```bash
cp .env.example .env          # set MODELS_DIR and MODEL
# drop your license.bin next to the compose file (or upload it later in the console)
docker compose up
```

- **Engine** → `http://localhost:8089/v1`: OpenAI-compatible, point your app here
- **Console** → `http://localhost:8080`: open it, paste the engine URL + the admin key

**See it work:** open the console's **Playground** and click a prebuilt test prompt. A benign one routes
**GREEN** (`x-prior-zone: GREEN`, `x-prior-injections: 0`) and is answered bit-for-bit as the base model
would; a harmful one routes **RED**, and the refusal holds even when you obfuscate the request. Full
walkthrough: **[Getting Started](https://eagle-logic.com/docs/)**.

The compose file and [`.env.example`](./.env.example) are commented inline for the knobs you'll actually
touch: the **admin key** (printed once in the engine logs on first run; `PRIOR_RESET_ADMIN_KEY=1` rotates
it), the **policy switch** (`PRIOR_POLICY=off` for byte-for-byte passthrough), and **CPU-only** mode
(`PRIOR_NATIVE_NGL=0`, delete the `deploy:` block).

**Picking an image.** `prior:slim` is the CUDA 12.4 build: it covers Turing, Ampere, and Ada (T4,
A100, RTX 30/40) and wants driver 550. Two alternates, same engine binary, different GPU reach:
`prior:slim-cuda12.8` for **Blackwell / RTX 50 series** (`sm_120`, driver 570), which 12.4 does not
cover, and `prior:slim-cpu` if you have no GPU and no driver. Pin a release with `1.0.0-cuda12.4`,
`1.0.0-cuda12.8`, or `1.0.0-cpu`. Full table: **[Choosing an image](https://eagle-logic.com/docs/getting-started#choosing-an-image)**.

<details>
<summary><b>Engine only, no console</b> (a 30-second smoke test)</summary>

```bash
docker run -d --name prior -p 127.0.0.1:8089:8089 \
  -e PRIOR_NATIVE_MODEL=/models/Llama-3.2-3B-Instruct-Q4_K_M.gguf \
  -v "$(pwd)/models:/models:ro" \
  -v "$(pwd)/license.bin:/app/license.bin:ro" \
  ghcr.io/eagle-logic/prior:slim

curl -s http://localhost:8089/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"prior","messages":[{"role":"user","content":"Write a haiku about the ocean."}]}' -i
```
</details>

The engine verifies its license **locally** against a baked-in public key (no activation call to start),
and your prompts and model data never leave your environment. Connected tiers make a lightweight periodic
license check-in (license id + node id only, never your data); an **air-gapped** tier that makes no
outbound calls at all is available for offline deployments.

---

## Benchmark highlights

Matched-pair tests (same model, seed, and harness; steering off vs on), independently judged by the
**cais/HarmBench classifier**. Full results, provenance, and limitations: **[BENCHMARKS.md](./BENCHMARKS.md)**.

| Result | Meaning |
|---|---|
| Forbidden-prompt harm down **86%** on Mistral-Nemo-12B (StrongREJECT, the benchmark's own classifier: mean harm 0.600 to 0.084) | Less than half the residual harm a strong safety system prompt leaves (0.184). |
| **100%** of gate-routed harmful completions refused (**79/79**, cais-judged, 11 models) | When the gate fires, the intervention holds. |
| **96%** jailbreak recovery at **0** benign false-blocks | Holds up against obfuscated phrasings without taxing legit traffic. |
| Generalization to unseen attacks **AUC ~0.93** | Coverage converges on a shared harm manifold, not string-chasing. |
| **GSM8K byte-identical**, **MMLU Δ 0**, **0 induced over-refusals** | Zero capability tax when no policy fires. |
| Overhead **+0.0%** (no policy) / **+0.2%** (intervention) | Negligible cost. |

We publish provenance with every figure and **don't publish numbers we can't defend**. That includes an
honest account of where the frontier still is (gate recall) and what's out of scope by design.

---

## Supported models

Steering is **calibrated per model**, so PRIOR ships a **registry** grading each supported GGUF
**certified** (calibration swept and validated) or **beta** (sensible defaults). The gate works on any
model; reliable steering needs a supported one. The current roster is **22 calibrated builds across 9
architecture families**. Recommended quickstart: **Llama-3.2-3B** (certified, ~2 GB). Full list + fetch
commands: **[MODELS.md](./MODELS.md)**.

---

## How it's licensed

- **Self-hosted, node-locked** subscription, billed annually. The engine verifies an ED25519-signed
  `license.bin` **locally** against a baked-in public key (no activation call to run). Connected tiers
  make a periodic license check-in (license id + node id only, never your data); an **air-gapped** tier
  with no outbound calls is available for offline deployments.
- **Free 30-day trial:** fully featured, no node lock. The same file slot later takes a paid license,
  with no reinstall and no migration.
- **Distribution:** `ghcr.io/eagle-logic/prior:slim` (public, default packs, start here) and a
  private full-pack image for paid/enterprise. See **[pricing & plans](https://eagle-logic.com/get-prior)**.

---

<sub>PRIOR is a product of **Eagle Logic** (State Space Systems LLC). The engine is proprietary and
licensed; this repository is public documentation. © 2026 State Space Systems LLC.</sub>
