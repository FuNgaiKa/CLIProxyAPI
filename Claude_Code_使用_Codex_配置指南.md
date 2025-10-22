# Claude Code 使用 Codex (88code.org) 配置指南

## 快速开始

### 1. 修改 CLIProxyAPI 配置

编辑 `config.yaml`：

```yaml
# Server port
port: 8317

# Enable debug logging (推荐，方便排查问题)
debug: true

# Codex API keys - 88code.org 使用 Responses API
codex-api-key:
  - api-key: "你的88code API key"
    base-url: "https://www.88code.org/openai/v1"
```

### 2. 修改 Claude Code 配置

编辑 `~/.claude/settings.json`（Windows: `C:\Users\你的用户名\.claude\settings.json`）：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8317",
    "ANTHROPIC_AUTH_TOKEN": "sk-dummy"
  },
  "model": "gpt-5-codex",
  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}
```

**重要提示**：
- ⚠️ JSON 不支持注释，请勿在 JSON 文件中使用 `//` 或 `/* */` 注释
- ⚠️ 必须使用 Codex 模型名称（如 `gpt-5-codex`），而不是 Claude 模型名称

### 3. 启动服务

```bash
# 启动 CLIProxyAPI
./cli-proxy-api.exe

# 完全关闭并重启 Claude Code
```

### 4. 验证

在 Claude Code 中发送 "hello"，如果成功则配置完成！

---

## 问题背景

当你想让 Claude Code 通过 CLIProxyAPI 调用 88code.org 的 Codex 服务时，可能会遇到各种错误。本指南基于实际踩坑经验，帮助你正确配置。

### 架构流程

```
Claude Code → http://127.0.0.1:8317/v1/messages → CLIProxyAPI → 88code.org Codex
```

---

## 常见错误及解决方案

### ❌ 错误 1: `400 unknown provider for model`

**错误示例**：
```
400 unknown provider for model claude-3-5-sonnet-20241022
```

**原因**：
- 模型没有在注册表中注册
- 使用了 Claude 模型名称，但配置的是 `codex-api-key`（不支持别名映射）

**解决方案**：
1. **方案 A（推荐）**：使用 Codex 原生模型名称

   在 Claude Code 的 `settings.json` 中：
   ```json
   {
     "model": "gpt-5-codex"
   }
   ```

2. **方案 B**：使用 `openai-compatibility` 配置（但 88code 不适用，见错误 3）

**验证**：
- 启动 CLIProxyAPI，查看日志是否显示模型已注册：
  ```
  [debug] Registered new model gpt-5-codex from provider codex
  ```

---

### ❌ 错误 2: `500 auth_unavailable: no auth available`

**错误示例**：
```
500 auth_unavailable: no auth available
```

**原因**：
- 之前的错误请求导致 API key 被临时禁用
- 认证管理器将失败的 key 标记为 unavailable

**解决方案**：
1. **重启 CLIProxyAPI 服务**（最简单）
   ```bash
   # Ctrl+C 停止服务
   ./cli-proxy-api.exe
   ```

2. 或等待重试时间（通常 5 分钟）

**验证**：
- 查看日志，确认 key 已重新加载：
  ```
  [info] full client load complete - 1 clients (1 Codex keys)
  ```

---

### ❌ 错误 3: `404 No static resource openai/chat/completions`

**错误示例**：
```
{"error":{"code":404,"type":"Not Found","message":"No static resource openai/chat/completions"}}
```

**原因**：
- 88code.org 使用的是 **OpenAI Responses API**，而不是 Chat Completions API
- 使用了 `openai-compatibility` 配置（硬编码使用 `/chat/completions` 端点）

**解决方案**：
- ✅ 使用 `codex-api-key` 配置（支持 Responses API）
- ❌ 不要使用 `openai-compatibility` 配置（不适用于 88code）

**正确配置**：
```yaml
codex-api-key:
  - api-key: "你的API key"
    base-url: "https://www.88code.org/openai/v1"
```

---

### ❌ 错误 4: `401 API key is disabled`

**错误示例**：
```
401 API key is disabled
```

**原因**：
- API key 因为连续失败被临时禁用
- API key 本身无效或已过期

**解决方案**：
1. **重启 CLIProxyAPI**（清除禁用状态）
2. **验证 API key 是否有效**：
   ```bash
   curl -X POST https://www.88code.org/openai/v1/chat/completions \
     -H "Authorization: Bearer 你的API_key" \
     -H "Content-Type: application/json" \
     -d '{"model":"gpt-5-codex","messages":[{"role":"user","content":"test"}]}'
   ```

---

### ❌ 错误 5: JSON 解析失败

**错误示例**：
- Claude Code 启动失败
- 配置文件无法加载

**原因**：
- `settings.json` 中包含了注释（`//` 或 `/* */`）
- JSON 格式不合法

**解决方案**：
```json
// ❌ 错误 - JSON 不支持注释
{
  "model": "gpt-5-codex",  // 这是注释
  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}

// ✅ 正确 - 移除所有注释
{
  "model": "gpt-5-codex",
  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}
```

---

## 完整配置示例

### CLIProxyAPI 配置 (config.yaml)

```yaml
# Server port
port: 8317

# Management API settings
remote-management:
  allow-remote: false
  secret-key: ""
  disable-control-panel: false

# Authentication directory
auth-dir: "C:/Users/你的用户名/.cli-proxy-api"

# API keys for authentication (留空表示不需要认证)
api-keys: []

# Enable debug logging (推荐开启，方便排查问题)
debug: true

# Logging configuration
logging-to-file: false

# Usage statistics
usage-statistics-enabled: false

# Proxy URL
proxy-url: ""

# Request retry
request-retry: 3

# Quota exceeded behavior
quota-exceeded:
  switch-project: true
  switch-preview-model: true

# Codex API keys - 用于 88code.org
codex-api-key:
  - api-key: "88_你的完整API_key"
    base-url: "https://www.88code.org/openai/v1"
```

### Claude Code 配置 (settings.json)

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:8317",
    "ANTHROPIC_AUTH_TOKEN": "sk-dummy"
  },
  "model": "gpt-5-codex",
  "permissions": {
    "defaultMode": "bypassPermissions"
  }
}
```

---

## 88code.org 支持的 Codex 模型

根据 CLIProxyAPI 启动日志，88code.org 支持以下模型：

- `gpt-5` - 基础模型
- `gpt-5-minimal` - 最小配置
- `gpt-5-low` - 低推理力度
- `gpt-5-medium` - 中推理力度
- `gpt-5-high` - 高推理力度
- `gpt-5-codex` - **推荐用于编程**
- `gpt-5-codex-low` - 低推理力度编程
- `gpt-5-codex-medium` - 中推理力度编程
- `gpt-5-codex-high` - 高推理力度编程
- `codex-mini-latest` - 轻量级模型

**推荐**：对于 Claude Code，建议使用 `gpt-5-codex` 或 `gpt-5-codex-high`。

---

## 验证步骤

### 1. 验证 CLIProxyAPI 配置

启动 CLIProxyAPI 后，检查日志：

```bash
./cli-proxy-api.exe
```

**期望的日志输出**：
```
[info] full client load complete - 1 clients (0 auth files + 0 GL API keys + 0 Claude API keys + 1 Codex keys + 0 OpenAI-compat)
[debug] Registered new model gpt-5 from provider codex
[debug] Registered new model gpt-5-codex from provider codex
[debug] Registered client codex:apikey:... from provider codex with 10 models
```

**关键检查点**：
- ✅ `1 Codex keys` - Codex API key 已加载
- ✅ `Registered new model gpt-5-codex` - 模型已注册
- ✅ 没有错误信息

---

### 2. 验证 Claude Code 配置

在 Claude Code 中输入 `/status`：

```
Anthropic base URL: http://127.0.0.1:8317  ✅
```

---

### 3. 测试请求

在 Claude Code 中发送简单请求：

```
> hello
```

**成功的日志**：
```
[debug] Use API key 88_7...482e for model gpt-5-codex
[info] [GIN] | 200 | ...ms | 127.0.0.1 | POST "/v1/messages?beta=true"
```

---

## 为什么不能使用 openai-compatibility 配置？

很多人可能想使用 `openai-compatibility` 配置来实现模型别名映射（Claude 模型名 → Codex 模型名），但这在 88code.org 上**不可行**。

### 原因分析

1. **API 端点不同**：
   - `openai-compatibility` 硬编码使用 `/chat/completions` 端点
   - 88code.org 使用 `/responses` 端点（OpenAI Responses API）

2. **代码证据**：
   ```go
   // internal/runtime/executor/openai_compat_executor.go:56
   url := strings.TrimSuffix(baseURL, "/") + "/chat/completions"
   ```

3. **88code 官方推荐配置**：
   ```toml
   [model_providers.88code]
   base_url = "https://www.88code.org/openai/v1"
   wire_api = "responses"  # 关键！使用 Responses API
   ```

### 解决方案

**接受现实，直接使用 Codex 模型名称**：
- ✅ 简单直接
- ✅ 避免配置错误
- ✅ 性能更好（无需别名转换）

---

## 故障排查流程

### Step 1: 启用 debug 模式

在 `config.yaml` 中：
```yaml
debug: true
```

### Step 2: 重启服务

```bash
# 停止 CLIProxyAPI
Ctrl+C

# 重新启动
./cli-proxy-api.exe
```

### Step 3: 查看启动日志

检查以下关键信息：
1. `1 Codex keys` - API key 已加载
2. `Registered new model gpt-5-codex` - 模型已注册
3. 没有错误或警告

### Step 4: 重启 Claude Code

**必须完全关闭**后重新启动（而不是刷新）。

### Step 5: 发送测试请求

在 Claude Code 中输入简单文本，查看日志：

**成功标志**：
```
[debug] Use API key 88_7...482e for model gpt-5-codex
[info] [GIN] | 200 | ...ms | 127.0.0.1 | POST "/v1/messages?beta=true"
```

**失败标志**：
```
[error] [GIN] | 400 | ...
[error] [GIN] | 500 | ...
```

---

## 常见问题 FAQ

### Q1: 为什么必须使用 Codex 模型名称？

**A**: 因为：
1. `codex-api-key` 配置不支持模型别名映射
2. `openai-compatibility` 配置不支持 88code.org 的 Responses API
3. 直接使用 Codex 模型名最简单可靠

---

### Q2: 可以使用 Claude 模型名称吗？

**A**: 理论上可以通过修改 CLIProxyAPI 源码实现，但**不推荐**：
1. 需要修改 `openai_compat_executor.go` 支持 Responses API
2. 维护成本高
3. 性能开销大
4. 直接用 Codex 模型名更简单

---

### Q3: base-url 应该如何设置？

**A**: 对于 88code.org：
```yaml
# ✅ 正确 - 包含完整路径
base-url: "https://www.88code.org/openai/v1"

# ❌ 错误 - 缺少路径
base-url: "https://www.88code.org"

# ❌ 错误 - 路径错误
base-url: "https://www.88code.org/openai"
```

**验证方法**：
```bash
# 完整端点应该是
https://www.88code.org/openai/v1/responses
```

---

### Q4: 多个 API key 如何配置？

**A**:
```yaml
codex-api-key:
  - api-key: "88_key1..."
    base-url: "https://www.88code.org/openai/v1"
  - api-key: "88_key2..."
    base-url: "https://www.88code.org/openai/v1"
```

CLIProxyAPI 会自动进行负载均衡。

---

### Q5: 如何测试 API key 是否有效？

**A**:
```bash
curl -X POST https://www.88code.org/openai/v1/chat/completions \
  -H "Authorization: Bearer 88_你的API_key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5-codex",
    "messages": [{"role": "user", "content": "test"}]
  }'
```

如果返回正常响应（而不是 401/403），说明 key 有效。

---

### Q6: Claude Code 必须完全重启吗？

**A**: 是的！
- ❌ 刷新页面 - 不够
- ❌ 重新加载配置 - 不够
- ✅ **完全关闭后重新启动** - 必须

---

### Q7: 配置文件修改后需要重启吗？

**A**:
- **CLIProxyAPI**: 不需要，支持热重载
- **Claude Code**: 需要完全重启

---

## 技术说明

### CLIProxyAPI 架构

```
Claude Code (Claude API 格式)
  ↓
CLIProxyAPI (/v1/messages 端点)
  ↓
格式转换器 (Claude → Codex/Responses)
  ↓
Codex 执行器 (/responses 端点)
  ↓
88code.org
```

### 关键代码文件

- `sdk/api/handlers/claude/code_handlers.go` - Claude API 处理器
- `internal/translator/codex/claude/codex_claude_request.go` - Claude → Codex 请求转换
- `internal/translator/codex/claude/codex_claude_response.go` - Codex → Claude 响应转换
- `internal/runtime/executor/codex_executor.go` - Codex 执行器
- `internal/registry/model_registry.go` - 模型注册表

### 支持的功能

- ✅ 流式响应 (SSE)
- ✅ 非流式响应
- ✅ Tool use / Function calling
- ✅ 多轮对话
- ✅ 系统提示词
- ✅ 思维链 (Chain of Thought)

---

## 注意事项

1. ⚠️ **JSON 不支持注释**：`settings.json` 中不能使用 `//` 或 `/* */`
2. ⚠️ **必须完全重启 Claude Code**：修改配置后必须完全关闭再启动
3. ⚠️ **使用 Codex 模型名称**：不能使用 Claude 模型名称
4. ⚠️ **base-url 必须正确**：必须包含 `/openai/v1`
5. ⚠️ **debug 模式很有用**：出问题时启用 `debug: true`
6. ✅ **配置文件热重载**：CLIProxyAPI 的 `config.yaml` 修改后自动生效
7. ✅ **ANTHROPIC_AUTH_TOKEN 可以是任意值**：代理不验证此 token

---

## 总结

### 核心要点

1. ✅ 使用 `codex-api-key` 配置（不要用 `openai-compatibility`）
2. ✅ `base-url: "https://www.88code.org/openai/v1"`
3. ✅ Claude Code 使用 Codex 模型名称（如 `gpt-5-codex`）
4. ✅ `settings.json` 不要有注释
5. ✅ 启用 `debug: true` 方便排查问题
6. ✅ 遇到问题先重启 CLIProxyAPI

### 成功标志

**CLIProxyAPI 日志**：
```
[info] 1 Codex keys
[debug] Registered new model gpt-5-codex from provider codex
```

**Claude Code 请求日志**：
```
[debug] Use API key 88_7...482e for model gpt-5-codex
[info] [GIN] | 200 |
```

### 遇到问题？

1. 启用 `debug: true`
2. 重启 CLIProxyAPI
3. 完全重启 Claude Code
4. 查看本文档的"常见错误及解决方案"部分
5. 检查日志中的错误信息

---

**项目地址**: https://github.com/router-for-me/CLIProxyAPI
