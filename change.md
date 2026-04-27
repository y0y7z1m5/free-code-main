# 兼容标准 OpenAI/DeepSeek 接口修改记录 (Change Log)

为了让 `free-code` 能够直接调用官方 DeepSeek (或任何兼容标准 OpenAI 协议的第三方 API)，我们对源码进行了底层拦截与协议适配。以下是所有修改的内容总结：

## 1. 认证逻辑适配 (auth.ts)
**文件**: `src/utils/auth.ts`
*   **修改内容**: 修改了 `getCodexOAuthTokens` 函数。
*   **原因**: 原代码针对的是 ChatGPT 的内部接口，强制要求提供 JWT 格式的 OAuth Token。
*   **实现**: 增加了对 `.env` 中 `OPENAI_API_KEY` 的读取。如果检测到此环境变量，则自动伪造并返回一个有效期 10 年的本地 Token（`accessToken` 为用户的 API Key，`accountId` 为 `openai_user`），从而**完美绕过**了强制登录校验，解决了卡在 `Not logged in` 报错的问题。

## 2. API 协议请求与响应翻译 (codex-fetch-adapter.ts)
**文件**: `src/services/api/codex-fetch-adapter.ts`
这是本次修改的核心，将适配器从“仅限内部 ChatGPT 协议”升级为了“通用 OpenAI 协议”。

*   **修改 API URL 支持**: 
    将硬编码的 `CODEX_BASE_URL` 修改为支持读取 `process.env.OPENAI_BASE_URL`。
*   **请求体 (Request) 翻译适配**: 
    在原有的请求发送逻辑中增加判断。如果配置了自定义的 `OPENAI_BASE_URL`，则抛弃 ChatGPT 内部使用的 `{"instructions":..., "input":...}` 结构，将用户的消息自动转换为符合 OpenAI 官方标准的 `{"model":..., "messages":...}` 结构。
*   **流式响应 (SSE Response) 翻译适配**: 
    因为标准的 OpenAI 流式返回数据位于 `choices[0].delta.content`，而原代码只监听 `response.output_text.delta`。我们增加了对标准 OpenAI 增量数据的解析，并将其无缝转化为 Claude 内部能够识别的 `content_block_delta`，让界面能实现逐字打印。
*   **跳过 Token 强校验**: 
    修改了 `extractAccountId` 方法。原方法会把 API Key 当成 JWT 进行解码，导致 `Failed to extract account ID from Codex token` 的报错。现在如果是自定义请求，会直接返回占位符而不报错。
*   **模型名称白名单**: 
    在 `CODEX_MODELS` 白名单数组中加入了 `{ id: 'deepseek-chat', label: 'DeepSeek Chat', description: 'DeepSeek V3' }`，这解决了启动时一直提示“模型不存在”并强制要求 `Run /model` 的拦截问题。

## 3. 环境配置更新 (.env)
**文件**: `.env`
*   **修改内容**:
    ```env
    CLAUDE_CODE_USE_OPENAI=1
    OPENAI_BASE_URL=https://api.deepseek.com
    OPENAI_API_KEY=sk-5ac6351652e74a848f7bb21f75b31962
    ANTHROPIC_MODEL=deepseek-chat
    ANTHROPIC_CUSTOM_MODEL_OPTION=deepseek-chat
    ```
*   **作用**: 启动修改后的 OpenAI 通道，并指向 DeepSeek 官方服务器。
