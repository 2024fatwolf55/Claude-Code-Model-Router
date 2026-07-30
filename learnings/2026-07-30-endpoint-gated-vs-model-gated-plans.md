# Plan gating lives at two layers: the model, or the endpoint itself

## What happened

A user's Zhipu pay-as-you-go key (from a "glm-5.2 新手尊享包" tokens package)
returned `429 [1309] 您的GLM Coding Plan套餐已到期` on ccmr's GLM CN provider.
The instinct read — "API key, so it must be isolated from subscriptions" — was
wrong in an instructive way: the key was fine, the *endpoint* was gated.

`open.bigmodel.cn/api/anthropic` is Zhipu's ONLY domestic Anthropic-compatible
endpoint, and it is Coding-Plan-gated: the coding-plan FAQ states token
resource packages are unusable there and exhausted plans do NOT fall back to
pay-go billing. Domestic pay-go GLM exists only over the OpenAI protocol
(`/api/paas/v4`) — so a passthrough Anthropic gateway cannot offer pay-go GLM
at all. The provider was renamed `glm` → `glm-plan` (GLM_PLAN_API_KEY) to say
what it actually is.

## The two mirror failure shapes

- **Model-gated** (Qwen, 2026-07-20): shared endpoint accepts the key, one
  *model* is plan-only → `403 Model.AccessDenied` on that model, others work.
- **Endpoint-gated** (GLM, this note): the *endpoint* is plan-only → every
  request fails with a plan-status error (here `1309`), regardless of model,
  even with a valid, funded pay-go key.

## Diagnosis recipe (proved decisive here)

1. Control test the same key against the vendor's other-protocol pay-go
   endpoint (`/api/paas/v4` returned 200 and billed tokens) — this splits
   "bad key" from "gated door" in one request.
2. Trust the product FAQ over the generic protocol-compat doc. Zhipu's
   "Claude API 兼容" page implies any open-platform key works; the coding-plan
   FAQ ("Claude Code 中暂不支持使用其他资源包") describes actual behavior.
3. Probe for undocumented endpoint variants before declaring impossibility
   (all three candidate pay-go Anthropic paths 404'd) — then say "cannot be
   supported" with evidence instead of guessing a base_url into the config.

## Rule

When a plan/subscription error appears on a request made with a非订阅 key,
first ask WHICH layer is gated: run the same key against the vendor's pay-go
endpoint in its native protocol. If that succeeds, the endpoint is gated and
no key/config change on our side can fix it — the provider must be labeled a
plan provider (`-plan` naming, dedicated `*_PLAN_API_KEY`), and pay-go support
depends entirely on the vendor shipping a pay-go endpoint for our protocol.
