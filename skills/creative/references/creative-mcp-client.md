# Creative MCP Client — 调用实现

> 与 `ads-tiktok-mcp` 的 `references/mcp-client.md` **对齐的设计模式**：容错/重试/校验/健康检查。

```python
import requests, json, os

CREATIVE_API_KEY = os.environ.get("CREATIVE_MCP_API_KEY", "")
CREATIVE_TOOLS_URL = "https://creative.vidau.ai/mcp"

def creative_mcp(name, args=None, timeout=60):
    """调用 VidAU Creative MCP（SSE/Streamable HTTP 模式）。

    注意：Creative MCP 使用标准 MCP SSE 端点（不同于 TikTok Ads 的 REST 端点）。
    实际调用方式取决于宿主环境的 MCP client 实现：
    - 若宿主支持原生 MCP client → 通过 tool name 直接调用（推荐，无需 Key）
    - 若需 HTTP fallback → POST JSON-RPC 到 CREATIVE_TOOLS_URL
      （此时 CREATIVE_MCP_API_KEY 可选：若设置了则作为 apiKey 查询参数/Bearer 头传入；
       若未设置，说明走平台会话鉴权，直接 POST 即可）

    **鉴权说明**：Creative MCP 标准 VidAU SaaS 部署走平台会话鉴权，通常无需独立 Key。
    仅私有部署要求独立 Key 时才需设置 CREATIVE_MCP_API_KEY。
    """
    # 方式一：原生 MCP client 调用（推荐，由宿主注入，无需 Key）
    # 实际执行时，宿主环境会将此函数替换为真实 MCP tool call
    # 此处为参考实现：

    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {"name": name, "arguments": args or {}},
        "id": 1
    }
    # CREATIVE_API_KEY 为可选；有则附带，无则走会话鉴权
    if CREATIVE_API_KEY:
        CREATIVE_TOOLS_URL_r = f"{CREATIVE_TOOLS_URL}?apiKey={CREATIVE_API_KEY}"
    else:
        CREATIVE_TOOLS_URL_r = CREATIVE_TOOLS_URL
    try:
        r = requests.post(
            CREATIVE_TOOLS_URL_r,
            json=payload,
            headers={"Content-Type": "application/json"},
            timeout=timeout
        )
    except requests.exceptions.Timeout:
        raise RuntimeError(f"Creative MCP 请求超时（>{timeout}s）：{name}")
    except requests.exceptions.RequestException as e:
        raise RuntimeError(f"Creative MCP 网络错误：{e}")

    try:
        resp = r.json()
    except ValueError:
        raise RuntimeError(f"Creative MCP 返回非 JSON（HTTP {r.status_code}）")

    if "error" in resp:
        err = resp["error"] or {}
        raise RuntimeError(f"Creative MCP 错误 [{err.get('code')}]：{err.get('message')}")

    result = resp.get("result")
    if not result:
        raise RuntimeError(f"Creative MCP 返回空 result：{name}")

    content = result.get("content") or []
    if not content:
        return result  # 某些工具返回无 content 的 result

    if isinstance(content, list) and len(content) > 0 and "text" in content[0]:
        try:
            return json.loads(content[0]["text"])
        except (ValueError, TypeError):
            return content[0]["text"]

    return result


def creative_mcp_retry(name, args=None, max_retries=3, base_timeout=60):
    """带退避的重试包装。

    重试条件：超时 / 429(限流) / 5xx。
    不重试：400/401/403（参数/鉴权错误，重试无效）。
    """
    import time
    for attempt in range(max_retries):
        try:
            return creative_mcp(name, args, timeout=base_timeout * (attempt + 1))
        except RuntimeError as e:
            err_msg = str(e)
            if any(kw in err_msg.lower() for kw in ["timeout", "429", "5xx", "gateway", "rate limit"]):
                if attempt < max_retries - 1:
                    wait = 2 ** attempt
                    time.sleep(wait)
                    continue
            raise
    return creative_mcp(name, args, timeout=base_timeout * max_retries)


def creative_mcp_healthcheck():
    """健康检查：验证 key 有效性 + 连通性。"""
    try:
        models = creative_mcp("creative_list_models", {}, timeout=10)
        return True, f"Creative MCP 正常，可用模型: {len(models) if isinstance(models, list) else 'ok'}"
    except RuntimeError as e:
        return False, f"Creative MCP 异常: {e}"
```

## 关键差异 vs Ads MCP Client

| 维度 | Ads MCP (`tiktok.vidau.ai`) | Creative MCP (`creative.vidau.ai`) |
|------|---------------------------|----------------------------------|
| 端点路径 | `/api/mcp/tools` (REST) | `/mcp` (SSE/Streamable HTTP) |
| 默认超时 | 15s（查询）/ 60s（sync） | 60s（生成类可能 2-5min） |
| 典型延迟 | ~1s/call | 图片 ~1-2min, 视频 ~2-5min, 脚本 10-30min |
| 重试策略 | 3 次，退避 2^n s | 3 次，更长 base_timeout |
| 异步模式 | 无（全部同步返回） | 有 job_id 追踪机制 |

## Prompt Gate 强制规则

> **所有图片/视频生成 MCP 调用前，必须先加载对应 Prompt Skill：**

| 目标 MCP | 必须先加载 | 产出 |
|----------|-----------|------|
| `creative_generate_image`, `batch_variants` | `creative-gpt-image2-prompt` | GPT Image 2 production prompt |
| `creative_generate_video`, `creative_image_to_video`, `creative_first_frame_to_video`, script2film shots | `creative-seedance2-prompt` | Seedance 2.0 production prompt |
| `creative_generate_script` | `creative-narrative-router` | narrative structure + beats |

**禁止**将用户原始文本直接作为 `prompt` 参数传入任何 Creative MCP 工具。
