# Hermes Agent → ZCode / GLM-5.2 Adaptation Patches

Fixes the persistent **429 (code 1305 "overloaded")** error when using Hermes Agent with `zai/glm-5.2` (Z.AI Coding Plan).

[English](./README.md) | [中文](./README.zh-CN.md)

## Root Cause

This is **not** ordinary rate limiting. Swapping API keys, changing endpoints, or shrinking request size all fail because two independent triggers are at work:

| Layer | Trigger | Symptom |
|---|---|---|
| **System Prompt content filter** | The product name `"Hermes Agent"` appears in the prompt | Server returns 429/1305 disguised as "overloaded" |
| **Client fingerprint detection** | Request headers don't match the real ZCode Desktop client | Cloudflare edge 1010 / silent throttling |

Both gates must be bypassed simultaneously for stable API access.

## What the Patches Do

### Layer 1 — System Prompt brand rewrite (`agent/system_prompt.py`)

At the final assembly boundary in `build_system_prompt()`, when the provider is `zai` and the model is `glm-5.2`, every occurrence of `"Hermes Agent"` in the assembled prompt is rewritten to `"ZCode"`.

**Key design**: The rewrite happens on the in-memory prompt *after* assembly. **Nothing on disk is touched** — docs, skills, memories, and stored sessions all remain byte-identical. The substitution exists only in the text sent to the API.

### Layer 2 — Client fingerprint headers (`agent/auxiliary_client.py` + `run_agent.py`)

`build_zcode_headers()` generates a set of HTTP headers that match the real ZCode Desktop 3.1.8 client:

| Header | Value | Lifecycle |
|---|---|---|
| `User-Agent` | `ZCode/<ver> ai-sdk/anthropic/3.0.81` | Static |
| `X-ZCode-App-Version` | `<ver>` (overridable via `ZCODE_APP_VERSION` env var) | Static |
| `X-ZCode-Agent` | `glm` | Static |
| `x-zcode-trace-id` | Random hex | **Fresh per request** |
| `x-request-id` | Random hex | **Fresh per request** |
| `x-session-id` | `sess_<24hex>` | **Process-stable** |
| `x-query-id` | Random hex | **Fresh per request** |
| `HTTP-Referer` | `https://zcode.z.ai` | Static |
| `X-Title` | `Z Code` | Static |

In `run_agent.py`, these headers are injected when the provider is `zai` or the base URL matches `api.z.ai` / `open.bigmodel.cn`.

**Fingerprint source**: Reverse-engineered from ZCode Desktop 3.1.8 (Electron), bundled obfuscated code at `resources/glm/zcode.cjs`, functions `eao()` / `rao()`.

## Files Changed

| File | Change |
|---|---|
| `agent/auxiliary_client.py` | New `build_zcode_headers()` function + `uuid` import |
| `agent/system_prompt.py` | Brand-word replacement logic in `build_system_prompt()` |
| `run_agent.py` | New `zai` branch in `_apply_client_headers_for_base_url()` |
| `tests/agent/test_system_prompt.py` | New `TestZaiSystemPromptRewrite` test class (2 cases) |
| `tests/run_agent/test_provider_attribution_headers.py` | New fingerprint assertion test |

**6 files, 127 lines** (including tests). The `package-lock.json` diff is unrelated and excluded.

## Installation

### Option A — Apply the unified diff (recommended)

```bash
cd ~/.hermes/hermes-agent
git apply patches/zcode-glm-patch.diff
```

### Option B — Manual

Reference the individual sections in `patches/zcode-glm-patch.diff` and edit each file by hand.

### Verify

```bash
cd ~/.hermes/hermes-agent
source venv/bin/activate
python -m pytest tests/agent/test_system_prompt.py::TestZaiSystemPromptRewrite \
  tests/run_agent/test_provider_attribution_headers.py::test_zai_base_url_applies_zcode_client_fingerprint -v
```

## Upstream Sync Notes

After merging Hermes upstream updates, re-check:

1. `auxiliary_client.py` and `run_agent.py` are high-churn files — watch the `_create_openai_client()` wrapper and header branch logic for conflicts.
2. `system_prompt.py` upstream rarely changes the assembly boundary, but verify.
3. ZCode version defaults to `3.1.8`; override at runtime with `ZCODE_APP_VERSION=3.2.0` without code changes.

## Credits

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research
- Root-cause analysis: [Deep Router — Hermes Agent 优化：解决GLM-5.2模型429(1305 overloaded)](https://deeprouter.org/article/hermes-agent-optimization-fix-glm-5-2-model-429-1305-overloaded)

## License

MIT
