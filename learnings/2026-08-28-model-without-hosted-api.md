# A requested "model" may be open weights only; conflicting vendor docs need a freshness check

**Problem (one line):** Asked to add `glm-5.3-flash`, `qwen3.8-flash`, `qwen3.8-flash-next` to ccmr with CN/global and subscription/pay-go variants checked — one of the three had no hosted API at all, and two official pages disagreed on whether Token Plan covers `qwen3.8-flash`.

## Approach

1. **Before any edit, fill the Phase 0 fact table per model** from vendor pages only: model id, Anthropic-compatible base_url per region, billing mode per endpoint, context/max output. Read the model page *and* the vendor's own Claude Code integration page — the latter is the proof that the id is served on the Anthropic endpoint (`glm-5.3-flash` appeared verbatim in both bigmodel's and Z.ai's Claude Code env-var examples).
2. **For each vendor, map the model onto the providers ccmr already has** instead of inventing new ones: GLM → `glm-plan` (CN Coding Plan) + `glm-global` (Z.ai); Qwen → `qwen` (Bailian pay-go) + `qwen-plan` (Token Plan). A region/billing line that has no existing provider (Qwen international) is reported, not silently added.
3. **Prove a hosted endpoint exists before treating a name as a routable model.** For `qwen3.8-flash-next`: vendor model page 404, three first-party catalogs plus OpenRouter lack the id, and the HF card states the production API is `qwen3.8-flash`. Four independent negatives → it is open weights only → do not add; encode the decision as a test asserting absence so the next session sees *why*.
4. **TDD at the public seam** (`ConfigManager`): write the RED tests with the doc URL in a comment above each expectation, then edit Copy 1 + Copy 2, then README/version; template-consistency test guards drift.
5. **Doctor with a real key, then isolate the failing layer with a control.** `qwen-plan-3.8-flash` → 403 `AccessDenied.Unpurchased`. Running the *already-shipped* `qwen-plan-3.8-max` with the same key gave the same 403 → the subscription is inactive, so the error says nothing about the new variant.
6. **When two official pages conflict, date them by their contents.** The Token Plan tier table still listed retired `glm-5.1`/`glm-5`/`kimi-k2.5`, so it is stale; the latest-model page shipped with the 3.8-flash launch. Keep the variant, cite both pages, state "unverified" verbatim in config comment + README.

## Judgment calls (deliberately not done)

- **No alias `qwen3.8-flash-next → qwen3.8-flash`.** Silently routing a different model under a requested name is worse than passthrough; the user gets a clear "not found" and the README explains the substitute.
- **No new `qwen-intl` provider.** It would be a new key (Copies 3 + 5) and a scope change beyond the three requested models; reported as a follow-up instead.
- **Defaults untouched** (`glm`, `glm-global`, `qwen`, `qwen-plan` still resolve to the flagship). Flash tiers are additions, not the new flagship — unlike the 5.3 launch where the newest flagship took the default.
- **No workspace-scoped Bailian URL** (`{WorkspaceId}.cn-beijing.maas.aliyuncs.com`) as default even though docs call it "recommended": it needs a per-user id, and the legacy `dashscope.aliyuncs.com/apps/anthropic` is still documented and returned 200 live.
- **No commit / no live-yaml sync**: neither was requested this turn; both were separate explicit requests last time.

## Reusable rule

A model name in a request is a hypothesis, not a fact: confirm a first-party *hosted* endpoint serves that exact id (vendor Claude Code page or a live 200) before adding it, and when a doctor check fails, run the same key against an already-shipped sibling to tell account state from endpoint error before touching config.
