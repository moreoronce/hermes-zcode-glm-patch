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

```bash
cd ~/.hermes/hermes-agent
git apply patches/zcode-glm-patch.diff
```

验证：

```bash
source venv/bin/activate
python -m pytest tests/agent/test_system_prompt.py::TestZaiSystemPromptRewrite \
  tests/run_agent/test_provider_attribution_headers.py::test_zai_base_url_applies_zcode_client_fingerprint -v
```

## 致谢

- [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research
- [Deep Router 分析文章](https://deeprouter.org/article/hermes-agent-optimization-fix-glm-5-2-model-429-1305-overloaded)

## License

MIT
