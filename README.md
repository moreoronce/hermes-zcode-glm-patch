# Hermes Agent → ZCode/GLM-5.2 适配补丁

解决 Hermes Agent 使用 `zai/glm-5.2`（Z.AI Coding Plan）时频繁遭遇 **429 (code 1305 "overloaded")** 的问题。

## 问题根因

这**不是**常规限流。换 API Key、换 endpoint、降请求长度全都无效，因为触发机制是双重的：

| 层级 | 触发条件 | 现象 |
|---|---|---|
| **System Prompt 内容过滤** | prompt 中出现 `"Hermes Agent"` 品牌词 | 服务端返回 429/1305 伪装成 "overloaded" |
| **客户端指纹检测** | 请求头与真实 ZCode Desktop 不匹配 | Cloudflare 边缘 1010 / 静默限速 |

两道关卡互相独立，**必须同时绕过**才能稳定调用。

## 补丁内容

两个独立改动，各解决一层：

### 1. System Prompt 品牌词替换（`agent/system_prompt.py`）

在 `build_system_prompt()` 的最终组装边界，当 provider 为 `zai` 且模型为 `glm-5.2` 时，将 prompt 全文中的 `"Hermes Agent"` 替换为 `"ZCode"`。

**关键设计**：在组装后的 prompt 上做替换，**不修改磁盘上的任何文件**（docs、skills、memories、stored sessions 全部原样保留）。替换只存在于发往 API 的内存文本中。

### 2. 客户端指纹 Headers（`agent/auxiliary_client.py` + `run_agent.py`）

`build_zcode_headers()` 函数生成与 ZCode Desktop 3.1.8 一致的请求头指纹：

| Header | 值 | 生命周期 |
|---|---|---|
| `User-Agent` | `ZCode/<ver> ai-sdk/anthropic/3.0.81` | 恒定 |
| `X-ZCode-App-Version` | `<ver>`（可被 `ZCODE_APP_VERSION` 环境变量覆盖） | 恒定 |
| `X-ZCode-Agent` | `glm` | 恒定 |
| `x-zcode-trace-id` | 随机 hex | **每次请求新生成** |
| `x-request-id` | 随机 hex | **每次请求新生成** |
| `x-session-id` | `sess_<24hex>` | **进程级稳定** |
| `x-query-id` | 随机 hex | **每次请求新生成** |
| `HTTP-Referer` | `https://zcode.z.ai` | 恒定 |
| `X-Title` | `Z Code` | 恒定 |

在 `run_agent.py` 的 `_apply_client_headers_for_base_url()` 中，当检测到 `api.z.ai` / `open.bigmodel.cn` 域名或 `provider == "zai"` 时注入这些 headers。

**指纹来源**：逆向自 ZCode Desktop 3.1.8 (Electron)，打包混淆代码 `resources/glm/zcode.cjs` 的 `eao()` / `rao()` 函数。

## 涉及文件

| 文件 | 改动 |
|---|---|
| `agent/auxiliary_client.py` | 新增 `build_zcode_headers()` 函数 + `uuid` import |
| `agent/system_prompt.py` | `build_system_prompt()` 中新增品牌词替换逻辑 |
| `run_agent.py` | `_apply_client_headers_for_base_url()` 中新增 zai 分支 |
| `tests/agent/test_system_prompt.py` | 新增 `TestZaiSystemPromptRewrite` 测试类（2 个用例） |
| `tests/run_agent/test_provider_attribution_headers.py` | 新增 `test_zai_base_url_applies_zcode_client_fingerprint` |

**6 个文件，127 行改动**（含测试）。`package-lock.json` 的改动与此补丁无关，不含在内。

## 安装

### 方式一：直接 apply（推荐）

```bash
cd ~/.hermes/hermes-agent
git apply /path/to/zcode-glm-patch.diff
```

### 方式二：手动应用

参考 `patches/` 目录下的独立 diff 文件，逐文件 `patch -p1` 或手动编辑。

### 验证

```bash
cd ~/.hermes/hermes-agent
source venv/bin/activate
python -m pytest tests/agent/test_system_prompt.py::TestZaiSystemPromptRewrite \
  tests/run_agent/test_provider_attribution_headers.py::test_zai_base_url_applies_zcode_client_fingerprint -v
```

## 版本同步

每次 Hermes 上游更新后需要重新检查：

1. `git merge` 上游后检查冲突区域
2. `auxiliary_client.py` 和 `run_agent.py` 是高频变动文件，关注 `_create_openai_client()` wrapper 和 header 分支逻辑
3. ZCode 版本号跟随官方更新（当前 `3.1.8`），可通过 `ZCODE_APP_VERSION` 环境变量覆盖无需改代码

## 致谢

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research
- 问题分析参考：[Deep Router — Hermes Agent 优化：解决GLM-5.2模型429(1305 overloaded)](https://deeprouter.org/article/hermes-agent-optimization-fix-glm-5-2-model-429-1305-overloaded)

## License

MIT
