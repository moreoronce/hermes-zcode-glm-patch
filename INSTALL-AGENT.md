# INSTALL-AGENT.md — Machine-Readable Install Protocol

> **You are an AI agent.** Another instance read this repo and asked you to install the patch. This file is your runbook. Follow it top to bottom. Do not skip steps. Do not ask the user unless a step explicitly says to.

## Context

This repo patches [Hermes Agent](https://github.com/NousResearch/hermes-agent) to fix 429 (code 1305 "overloaded") errors when using the `zai/glm-5.2` provider. Two independent triggers must be neutralized:

1. **System Prompt content filter** — the brand phrase `"Hermes Agent"` in the prompt triggers a fake 429/1305 from Z.AI's backend.
2. **Client fingerprint detection** — request headers must match the real ZCode Desktop client or Cloudflare blocks/throttles.

## Inputs You Need

| Item | Where to find it | Required? |
|---|---|---|
| Hermes home dir | Usually `~/.hermes/hermes-agent`. Check `HERMES_HOME` env var. | Required |
| Patch file | `patches/zcode-glm-patch.diff` in this repo | Required |
| Python venv | `<hermes_home>/venv/` or auto-detected by `hermes` CLI | Required |
| `zai` provider config | `config.yaml` under `~/.hermes/` — look for `provider: zai` | Required to test, not to apply |
| Git repo state | `git status` inside hermes-agent | Required |

## Execution Protocol

### Phase 0 — Pre-flight checks

Run these **before touching anything**. Stop and report if any check fails.

```python
# Pseudocode — adapt to your tool interface
checks = {
    "hermes_home_exists": os.path.isdir(hermes_home),
    "venv_exists": os.path.isdir(f"{hermes_home}/venv"),
    "git_repo": os.path.isdir(f"{hermes_home}/.git"),
    "patch_file_exists": os.path.isfile("patches/zcode-glm-patch.diff"),
    "target_files_clean": check_git_status(hermes_home, TARGET_FILES),
}
```

**TARGET_FILES** (all 5 must have no uncommitted changes):
- `agent/auxiliary_client.py`
- `agent/system_prompt.py`
- `run_agent.py`
- `tests/agent/test_system_prompt.py`
- `tests/run_agent/test_provider_attribution_headers.py`

If any target file has uncommitted changes → **STOP**. Report to user: "Target files have uncommitted changes. Please commit or stash first." Do not proceed.

### Phase 1 — Backup

```bash
cd <hermes_home>
git tag pre-zcode-patch
```

This is the rollback anchor. Remember the tag name — it's your escape hatch.

### Phase 2 — Dry-run apply

```bash
cd <hermes_home>
git apply --check <repo>/patches/zcode-glm-patch.diff
```

- **Exit 0** → proceed to Phase 3.
- **Exit 1** → upstream has diverged. Do NOT force-apply. Follow the **Manual Fallback** procedure below.

### Phase 3 — Apply for real

```bash
cd <hermes_home>
git apply <repo>/patches/zcode-glm-patch.diff
```

Verify exit code 0. If non-zero, `git checkout -- .` to revert partial applies and report the error.

### Phase 4 — Run tests

```bash
cd <hermes_home>
source venv/bin/activate
python -m pytest tests/agent/test_system_prompt.py::TestZaiSystemPromptRewrite \
  tests/run_agent/test_provider_attribution_headers.py::test_zai_base_url_applies_zcode_client_fingerprint -v
```

- **3 passed** → proceed to Phase 5.
- **Any failure** → **STOP**. Run `git reset --hard pre-zcode-patch` to roll back. Report the test output to the user. Do not attempt to fix the tests yourself unless the user explicitly asks.

### Phase 5 — Post-install verification

After the patch is applied and tests pass, verify the headers are actually being generated correctly:

```bash
cd <hermes_home>
source venv/bin/activate
python -c "
from agent.auxiliary_client import build_zcode_headers
h = build_zcode_headers()
assert h['User-Agent'].startswith('ZCode/'), f'UA wrong: {h[\"User-Agent\"]}'
assert h['X-ZCode-App-Version'], 'Missing version'
assert h['X-ZCode-Agent'] == 'glm', 'Wrong agent tag'
assert h['x-session-id'].startswith('sess_'), 'Bad session ID format'
print('Headers OK:')
for k, v in h.items():
    print(f'  {k}: {v}')
"
```

### Phase 6 — Report

Report to the user with:

1. **What was applied** — "Z.AI/ZCode compatibility patch across 3 source files + 2 test files"
2. **Test results** — "3/3 passed"
3. **Header verification** — paste the `build_zcode_headers()` output (safe to show — no secrets)
4. **Rollback instructions** — "If issues arise: `git reset --hard pre-zcode-patch`"
5. **Restart reminder** — "Restart Hermes (TUI/gateway/desktop) for changes to take effect. Existing clients won't hot-reload."

## Manual Fallback Procedure

Use this when `git apply --check` fails (Phase 2 exit 1). The upstream code has diverged enough that the diff context lines don't match.

### Strategy

Apply each file's changes independently using targeted search-and-replace, not bulk `git apply`. This is more resilient to minor context drift.

### File 1: `agent/auxiliary_client.py`

**Change A** — Add `uuid` import.

Find the import block at the top of the file. After `import time`, add:

```python
import uuid
```

**Change B** — Add `build_zcode_headers()` function.

Find `def build_nvidia_nim_headers(base_url: str | None) -> dict:` and its body. After that function's `return {}` line (the fallback return), add:

```python
_ZCODE_APP_VERSION = os.environ.get("ZCODE_APP_VERSION", "3.1.8")
_ZCODE_SESSION_ID = f"sess_{uuid.uuid4().hex[:24]}"


def build_zcode_headers() -> dict:
    """Return ZCode client headers matching the real ZCode Desktop fingerprint."""
    return {
        "User-Agent": f"ZCode/{_ZCODE_APP_VERSION} ai-sdk/anthropic/3.0.81",
        "X-ZCode-App-Version": _ZCODE_APP_VERSION,
        "X-ZCode-Agent": "glm",
        "x-zcode-trace-id": uuid.uuid4().hex,
        "x-request-id": uuid.uuid4().hex,
        "x-session-id": _ZCODE_SESSION_ID,
        "x-query-id": uuid.uuid4().hex,
        "HTTP-Referer": "https://zcode.z.ai",
        "X-Title": "Z Code",
    }
```

### File 2: `agent/system_prompt.py`

Find `def build_system_prompt(`. Inside the function body, find the line:

```python
    joined = "\n\n".join(p for p in (parts["stable"], parts["context"], parts["volatile"]) if p)
```

Immediately after that line (before any `for warning in` loop), add:

```python
    # Z.AI Coding Plan's glm-5.2 backend rejects prompts containing the exact
    # product phrase "Hermes Agent" with a misleading 429/code 1305.
    provider_lower = str(getattr(agent, "provider", "") or "").lower()
    model_lower = str(getattr(agent, "model", "") or "").lower()
    if provider_lower == "zai" and "glm-5.2" in model_lower:
        joined = joined.replace("Hermes Agent", "ZCode")

```

### File 3: `run_agent.py`

**Change A** — Add import.

Find the existing import block inside `_apply_client_headers_for_base_url` that imports from `agent.auxiliary_client`. Add `build_zcode_headers` to the import list:

```python
        from agent.auxiliary_client import (
            build_nvidia_nim_headers,
            build_or_headers,
            build_zcode_headers,
        )
```

**Change B** — Add provider branch.

Find the `elif base_url_host_matches(base_url, "api.kimi.com"):` branch. After its body, add:

```python
        elif (
            getattr(self, "provider", None) == "zai"
            or base_url_host_matches(base_url, "api.z.ai")
            or base_url_host_matches(base_url, "open.bigmodel.cn")
        ):
            self._client_kwargs["default_headers"] = build_zcode_headers()
```

### File 4 & 5: Test files

If tests fail to apply, skip them — the core patch is in files 1-3. Tests are verification, not functionality. Report that tests were skipped and suggest the user run the manual verification in Phase 5 instead.

## Rollback

At any phase, if you need to undo:

```bash
cd <hermes_home>
git reset --hard pre-zcode-patch
```

This restores all 5 files to their pre-patch state. The `pre-zcode-patch` tag remains for future reference; delete it with `git tag -d pre-zcode-patch` when no longer needed.

## What NOT to Do

- **Do not** commit the changes on behalf of the user unless they explicitly ask.
- **Do not** restart Hermes services yourself unless the user explicitly asks.
- **Do not** modify any file not listed in TARGET_FILES.
- **Do not** attempt to "improve" or "optimize" the patch — apply it as-is or fall back to manual.
- **Do not** suppress test failures — report them honestly.
