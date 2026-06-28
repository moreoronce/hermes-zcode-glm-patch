# Hermes Agent → ZCode/GLM-5.2 适配补丁

解决 Hermes Agent 使用 `zai/glm-5.2`（Z.AI Coding Plan）时频繁遭遇 **429 (code 1305 "overloaded")** 的问题。

[English](./README.md) | 中文

## 问题根因

这**不是**常规限流。换 API Key、换 endpoint、降请求长度全都无效，因为触发机制是双重的：

| 层级 | 触发条件 | 现象 |
|---|---|---|
| **System Prompt 内容过滤** | prompt 中出现 `"Hermes Agent"` 品牌词 | 服务端返回 429/1305 伪装成 "overloaded" |
| **客户端指纹检测** | 请求头与真实 ZCode Desktop 不匹配 | Cloudflare 边缘 1010 / 静默限速 |

两道关卡互相独立，**必须同时绕过**才能稳定调用。

## 补丁内容

### 第一层：System Prompt 品牌词替换

**文件**：`agent/system_prompt.py`

在 `build_system_prompt()` 的最终组装边界，当 provider 为 `zai` 且模型为 `glm-5.2` 时，将 prompt 全文中的 `"Hermes Agent"` 替换为 `"ZCode"`。

关键设计：在组装后的 prompt 上做替换，**不修改磁盘上的任何文件**（docs、skills、memories、stored sessions 全部原样保留）。替换只存在于发往 API 的内存文本中。

### 第二层：客户端指纹 Headers

**文件**：`agent/auxiliary_client.py` + `run_agent.py`

`build_zcode_headers()` 生成与 ZCode Desktop 一致的请求头指纹，包含身份标识、链路追踪 ID（每次请求新生成）、进程级稳定的 session ID、以及辅助归属头。

指纹来源：逆向自 ZCode Desktop 3.1.8 (Electron)，打包混淆代码 `resources/glm/zcode.cjs` 的 `eao()` / `rao()` 函数。

## 涉及文件

| 文件 | 改动 |
|---|---|
| `agent/auxiliary_client.py` | 新增 `build_zcode_headers()` + `uuid` import |
| `agent/system_prompt.py` | 品牌词替换逻辑 |
| `run_agent.py` | header 注入分支 |
| `tests/agent/test_system_prompt.py` | 2 个测试用例 |
| `tests/run_agent/test_provider_attribution_headers.py` | 1 个测试用例 |

**6 个文件，127 行改动**（含测试）。

## 安装

### 前置条件

- Hermes Agent v0.17.0+，安装路径 `~/.hermes/hermes-agent`
- 已配置 `zai` provider（GLM Coding Plan API Key）
- `git` 和 hermes-agent 内的 Python venv 可用
- 5 个目标文件没有未提交的本地改动（见下文）

### 人工安装（逐步）

**1 — 克隆本仓库**

```bash
git clone https://github.com/moreoronce/hermes-zcode-glm-patch.git /tmp/zcode-patch
```

**2 — 检查冲突**

应用前确认目标文件没有未提交的改动：

```bash
cd ~/.hermes/hermes-agent
git status --short -- agent/auxiliary_client.py agent/system_prompt.py run_agent.py \
  tests/agent/test_system_prompt.py tests/run_agent/test_provider_attribution_headers.py
```

有输出的话先 `git stash` 或 commit。

**3 — 备份（安全网）**

```bash
git tag pre-zcode-patch
```

后续出问题：`git reset --hard pre-zcode-patch` 一键回滚。

**4 — 应用补丁**

```bash
git apply --check /tmp/zcode-patch/patches/zcode-glm-patch.diff   # 先 dry-run
git apply /tmp/zcode-patch/patches/zcode-glm-patch.diff           # 正式应用
```

如果 `--check` 报冲突，说明上游代码已经偏离。参见下方[上游同步说明](#上游同步说明)，或使用[手动安装](#手动安装回退)。

**5 — 验证**

```bash
source venv/bin/activate
python -m pytest tests/agent/test_system_prompt.py::TestZaiSystemPromptRewrite \
  tests/run_agent/test_provider_attribution_headers.py::test_zai_base_url_applies_zcode_client_fingerprint -v
```

3 个测试全过。然后重启 Hermes（TUI / gateway / desktop app），让新 headers 生效——运行中的 client 不会热加载。

**6 — 确认修复**

开一个新 session 发条消息。在 gateway 日志或 dashboard 里，确认发往 `api.z.ai` 的请求带了 `User-Agent: ZCode/3.1.8 ...` 和 `X-ZCode-App-Version: 3.1.8`。429/1305 错误应该消失。

#### 手动安装回退

如果 `git apply` 因上游偏离失败，按 `patches/zcode-glm-patch.diff` 中的 diff hunk 逐文件手动编辑：

1. **`agent/auxiliary_client.py`** — 顶部 import 加 `uuid`；新增 `_ZCODE_APP_VERSION` / `_ZCODE_SESSION_ID` 常量和 `build_zcode_headers()` 函数（按 `@@ ... @@` 标记定位插入点）。
2. **`agent/system_prompt.py`** — 在 `build_system_prompt()` 中 `joined = ...` 之后、截断警告循环之前，插入 `zai`+`glm-5.2` 品牌词替换块。
3. **`run_agent.py`** — 在 `_apply_client_headers_for_base_url()` 中加入 `build_zcode_headers` import 和 `elif provider == "zai"` 分支。
4. **测试文件** — 按补丁添加测试用例。

### Agent 辅助安装

如果你是 AI Agent（另一个 Hermes 实例、Claude Code、Cursor 等），代替用户读取本仓库并应用补丁，参见 **[INSTALL-AGENT.md](./INSTALL-AGENT.md)** 的机器可读安装协议。

### 升级 ZCode 版本号

版本默认硬编码 `3.1.8`，但无需改代码即可覆盖：

```bash
# 在 ~/.hermes/.env 或 shell profile 中：
export ZCODE_APP_VERSION=3.2.0
```

修改默认值则编辑 `agent/auxiliary_client.py`：

```python
_ZCODE_APP_VERSION = os.environ.get("ZCODE_APP_VERSION", "3.1.8")  # ← 改这里
```

## 上游同步说明

上游合并后重新检查：

1. `auxiliary_client.py` 和 `run_agent.py` 是高频变动文件，关注 `_create_openai_client()` wrapper 和 header 分支逻辑。
2. `system_prompt.py` 上游极少改组装边界，但确认一下。
3. ZCode 版本号默认 `3.1.8`，可通过 `ZCODE_APP_VERSION` 环境变量覆盖，无需改代码。

## 致谢

## License

MIT
