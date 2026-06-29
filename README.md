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

### Prerequisites

- Hermes Agent v0.17.0+ installed at `~/.hermes/hermes-agent`
- A `zai` provider configured (GLM Coding Plan API key)
- `git` and a working Python venv inside `hermes-agent/`
- No uncommitted changes to the 5 target files (see below)

### Human install (step by step)

**1 — Clone this repo**

```bash
git clone https://github.com/moreoronce/hermes-zcode-glm-patch.git /tmp/zcode-patch
```

**2 — Check for conflicts**

Before applying, verify the target files have no uncommitted local changes:

```bash
cd ~/.hermes/hermes-agent
git status --short -- agent/auxiliary_client.py agent/system_prompt.py run_agent.py \
  tests/agent/test_system_prompt.py tests/run_agent/test_provider_attribution_headers.py
```

If anything shows up, `git stash` or commit first.

**3 — Back up (safety net)**

```bash
git tag pre-zcode-patch
```

If something goes wrong later: `git reset --hard pre-zcode-patch` rolls everything back in one command.

**4 — Apply the patch**

```bash
git apply --check /tmp/zcode-patch/patches/zcode-glm-patch.diff   # dry-run first
git apply /tmp/zcode-patch/patches/zcode-glm-patch.diff           # apply for real
```

If `--check` reports conflicts, the upstream code has diverged. See [Upstream Sync Notes](#upstream-sync-notes) below, or fall back to [manual installation](#manual-install-fallback).

**5 — Verify**

```bash
source venv/bin/activate
python -m pytest tests/agent/test_system_prompt.py::TestZaiSystemPromptRewrite \
  tests/run_agent/test_provider_attribution_headers.py::test_zai_base_url_applies_zcode_client_fingerprint -v
```

All 3 tests should pass. Then restart Hermes (TUI / gateway / desktop app) so the new headers take effect — existing in-flight clients won't hot-reload the change.

**6 — Confirm the fix**

Start a new session and send any message. In the gateway log or dashboard, verify the request to `api.z.ai` carries `User-Agent: ZCode/3.1.8 ...` and `X-ZCode-App-Version: 3.1.8`. The 429/1305 errors should be gone.

#### Manual install fallback

If `git apply` fails due to upstream divergence, apply each section of `patches/zcode-glm-patch.diff` by hand:

1. **`agent/auxiliary_client.py`** — add `import uuid` to the top imports; add the `_ZCODE_APP_VERSION` / `_ZCODE_SESSION_ID` constants and the `build_zcode_headers()` function (follow the diff hunk `@@ ... @@` markers to find insertion points).
2. **`agent/system_prompt.py`** — in `build_system_prompt()`, after the `joined = ...` line and before the truncation-warning loop, insert the `zai`+`glm-5.2` brand-rewrite block.
3. **`run_agent.py`** — in `_apply_client_headers_for_base_url()`, add the `build_zcode_headers` import and the `elif provider == "zai"` branch.
4. **Test files** — add the test cases from the diff.

### Agent-assisted install

If you are an AI agent (e.g. another Hermes instance, Claude Code, Cursor) reading this repo to apply the patch on behalf of a user, see **[INSTALL-AGENT.md](./INSTALL-AGENT.md)** for a machine-readable install protocol.

### Upgrading the ZCode version

The version is hardcoded as `3.1.8` but can be overridden without editing code:

```bash
# In ~/.hermes/.env or your shell profile:
export ZCODE_APP_VERSION=3.2.0
```

To change the default, edit the fallback in `agent/auxiliary_client.py`:

```python
_ZCODE_APP_VERSION = os.environ.get("ZCODE_APP_VERSION", "3.1.8")  # ← change this
```

## Upstream Sync Notes

After merging Hermes upstream updates, re-check:

1. `auxiliary_client.py` and `run_agent.py` are high-churn files — watch the `_create_openai_client()` wrapper and header branch logic for conflicts.
2. `system_prompt.py` upstream rarely changes the assembly boundary, but verify.
3. ZCode version defaults to `3.1.8`; override at runtime with `ZCODE_APP_VERSION=3.2.0` without code changes.

## Credits

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research
- Root-cause analysis: [Deep Router — Hermes Agent 优化：解决GLM-5.2模型429(1305 overloaded)](https://deeprouter.org/article/hermes-agent-optimization-fix-glm-5-2-model-429-1305-overloaded)

<p align="center">
  <a href="https://x.com/moreoronce">
    <img src="https://img.shields.io/badge/X-Follow_@moreoronce-black?style=for-the-badge&logo=x&logoColor=white" height="40" alt="X @moreoronce">
  </a>
</p>

## License

MIT
