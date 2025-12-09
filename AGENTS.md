# AGENTS 指南

面向二次开发的快速导览。覆盖核心职责、关键模块、接口约定与常见坑。

## 项目概览
- 目标：提供 Google Business Gemini 的 OpenAI 兼容代理，支持多账号轮询、图片输入输出、文件上传，以及 Web 管理后台。
- 技术栈：Flask + requests；单页管理前端 `index.html`（原生 JS/Fetch）；配置文件 `business_gemini_session.json`。
- 运行方式：`python gemini.py` 或 `docker-compose up -d`。健康检查 `/health`。

## 核心组件（后端）
- `AccountManager`：加载/保存配置；账号轮询；错误冷却（auth 15m、rate 5m、generic 2m）；状态字段 `account_states[*]` 包含 jwt/session/cooldown。
- JWT 获取：`get_jwt_for_account` 处理新版 refreshcookies 流程；依赖 cookies `__Secure-C_SES`、`__Host-C_OSES`、`csesidx`，可附加 `extra_cookies`。
- 会话与聊天：
  - `ensure_jwt_for_account` 缓存/刷新 JWT（>240s 重新取）。
  - `ensure_session_for_account` 缓存会话，不存在则 `create_chat_session`。
  - 非流式：`stream_chat_with_images` 全量拉取并解析生成文本/图片。
  - 流式：`stream_chat_realtime` 真正 SSE，每块 regex 抽文本，可附 `<think>`。
- 文件与图片：
  - 上传：`/v1/files` → `upload_file_to_gemini`（Base64 打包后调 `widgetAddContextFile`）。
  - 映射：`FileManager` 维护 openai file_id ↔ gemini fileId（内存态）。
  - 图片缓存：`image/` 目录，缓存 1 小时，`/image/<filename>` 提供访问；输出模式 `image_output_mode` 支持 `url`（默认）或 `base64`。
- 错误处理：`raise_for_account_response` 根据状态码抛 `AccountAuthError` / `AccountRateLimitError` / `AccountRequestError`；触发后会写入冷却并在 UI 显示。
- 日志：`filtered_print` 按全局级别过滤；`/api/logging` 读写级别，配置可持久化。

## 接口一览
- OpenAI 兼容：
  - `GET /v1/models`
  - `POST /v1/chat/completions`（支持 `messages`/`prompts`，`stream`，图片 `image_url`/`files`，`include_thought`）
  - 文件：`POST /v1/files`，`GET /v1/files`，`GET/DELETE /v1/files/:id`
  - 辅助：`GET /v1/status`，`GET /health`，`GET /image/<file>`
- 管理端（需 admin token）：
  - 账号：`GET/POST /api/accounts`，`PUT/DELETE /api/accounts/:id`，`POST /toggle`，`POST /refresh-cookie`，`GET /test`
  - 模型：`GET/POST /api/models`，`PUT/DELETE /api/models/:id`
  - 配置：`GET/PUT /api/config`，`POST /api/config/import`，`GET /api/config/export`
  - 代理：`POST /api/proxy/test`，`GET /api/proxy/status`
  - 令牌：`POST /api/auth/login`，`GET/POST /api/tokens`，`DELETE /api/tokens/:token`
  - 系统：`GET /api/status`，`GET/POST /api/logging`

## 鉴权模型
- 管理端：密码首登即设置，`/api/auth/login` 返回 HMAC token（也写 cookie `admin_token`）。
- OpenAI 兼容端：需要 `X-API-Token`/`Authorization: Bearer <token>` 或 admin token。`api_tokens` 来自配置。

## 前端速览
- `index.html`：管理控制台（仪表盘、账号、模型、配置、代理测试、token 管理、导入导出）。使用本地存储保存 admin token，所有请求经 `apiFetch` 注入头。
- `chat_history.html`：聊天记录展示页（受 admin 保护）。
- 样式：CSS 变量、亮/暗主题，原生组件+模态框+toast。

## 配置要点（`business_gemini_session.json`）
- `proxy` / `proxy_enabled`：代理地址与开关，`get_proxy()` 才会生效。
- `accounts[*]`：`team_id`、`secure_c_ses`、`host_c_oses`、`csesidx`、`user_agent`、`available`，可选 `extra_cookies`。
- `models[*]`：`id`、`name`、`description`、`context_length`、`max_tokens`、`enabled`；请求模型名按 `id` 透传到 Gemini。
- 其他：`api_tokens`、`admin_secret_key`、`log_level`、`image_output_mode`。

## 运行与调试
- 直接运行：`pip install -r requirements.txt && python gemini.py`。
- Docker：`docker-compose up -d`（挂载配置/前端/后端，健康检查已配置）。
- 常见测试：
  - 模型列表：`curl http://127.0.0.1:8000/v1/models -H "Authorization: Bearer <token>"`
  - 聊天：`POST /v1/chat/completions`，`stream:true` 时为 SSE。
  - 文件上传：`POST /v1/files` multipart；返回的 `file-xxx` 可在 chat 引用。
  - 管理登录：`POST /api/auth/login {"password": "..."}`

## 已知注意事项
- `/v1/files` 列表/查询当前引用 `openai_file_id` 键，但 `FileManager` 存储字段为 `id`/`filename` 等，需对齐字段或补充别名，否则可能报错。
- 文件映射与缓存均为内存态，进程重启会丢失映射记录；生产需外部存储或持久化策略。
- 代理检测仅简单访问 Google，可能受环境影响；生产应根据网络策略调整。

## 扩展建议
- 将文件映射与会话状态放入持久化存储（Redis/SQLite）以支持多实例。
- 补充单元测试（接口校验、模型映射、图片模式切换）。
- 增强日志：按请求链路附 trace id；将关键事件输出为结构化日志。

## 贡献提示
- 遵循现有日志/错误分类，确保异常路径调用 `mark_account_cooldown`。
- 变更配置字段时同步更新前端表单与导入导出。
- 保持响应格式兼容 OpenAI（`choices[].delta/content`、`[DONE]` 标记等）。
