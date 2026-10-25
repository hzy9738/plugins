# TRAE 私有化供应商（TraeCode CN Enterprise 协议原生）

按 TRAE 企业版 IDE/CLI 的调用协议直连私有化部署的 Magpie 供应商插件。不依赖 traecli 进程，不自动采集网页 cookie；可选的本地会话文件仅用于增强用量明细。

## 协议面（逆向自 TRAE CN Enterprise 1.107.1 / traecli 0.201.1-tob-alpha.1，2026-10 在私有化部署实测）

| 功能 | 端点（`{base}` = 企业 HTTPS 地址） | 鉴权头 | 说明 |
| --- | --- | --- | --- |
| 登录引导 | `POST /cloudide/api/v3/trae/GetLoginGuidance` | 无 | 返回企业 `LoginHost` |
| SSO 授权页 | `https://<LoginHost>/authorization?...&auth_callback_url=http://127.0.0.1:<port>/authorize&client_id=ono9krqynydwx5&...` | — | 登录域来自部署引导，浏览器由 Magpie 宿主打开 |
| 兑换/续期 | `POST /cloudide/api/v3/trae/oauth/ExchangeToken` | 无（body 自证） | 续期 body `{"RefreshToken","ClientID":"app_EMoamEEZ73f0CkXaXp7hrann"}`（抓包实证）；返回**轮换后**的 `RefreshToken` + `Token` + 过期时间 |
| 登录校验 | `POST /cloudide/api/v3/trae/CheckLogin` | `X-Cloudide-Token` | `{IsLogin, Region, UserID}` |
| 用户/用量 | `POST /cloudide/api/v3/trae/GetUserInfo` | `x-cloudide-token` | 席位、租户权益、个人额度、账号状态；无需 cookie |
| 模型目录 | `POST /api/ide/v1/cli/get_config_list` `{"function":"chat"}` | `Authorization: Cloud-IDE-JWT` + `X-App-Id` + `X-Ide-Version-Code` + `Originator` | 实时模型目录（config_name / model_name / prompt_max_tokens / 能力） |
| 会话编程 | `POST /api/ide/v2/llm_raw_chat` | `Cloud-IDE-JWT`（Authorization 形式）+ `x-app-id` 等 | SSE 事件流：output/done/token_usage/error/content_security/queue_* |

关键结论（相对旧版的修正）：

- **token 续期有无副作用的协议入口**：`ExchangeToken`。旧版让 CLI 跑 `exec 'reply with: ok'` 续期（联网、计费）；新版直接调 ExchangeToken，零计费。
- **refresh token 每次续期轮换**，新值必须立即持久化（magpie 的 `refresh` 钩子返回值会合并进账号存储）。
- **用量两层**：`GetUserInfo`（JWT，凭据有效且接口可达时可用）给出席位/租户权益/个人额度；基础用量/超额用量/企业内置模型明细只有控制台会话接口（`X-Cloudide-Tob-Session` cookie）能读（JWT 调不通，上游 2001）。插件以 JWT 为主，检测到有效网页会话时自动追加明细（旧版会话文件自动迁移，无需浏览器自动化；会话失效时回落并提示补录——从浏览器 DevTools 把该 cookie 的值写入 sidecar 文件即可）。
- **模型目录无需 models_cache.json**：`get_config_list` 实时返回（私有化部署实测 29 个配置 → 21 个可用模型），插件以完整部署地址和完整凭据区分目录，成功后复用 60 秒；网络故障、解析失败或畸形响应时，不限时保留该身份最后一份有效目录。有效空目录会撤回模型，鉴权失效会报错并清除该身份缓存。缓存仅在进程内，宿主重启后需要一次成功获取，不把凭据或上游目录落盘。
- **企业内置模型（管理员配置的 `custom_openai_compatible`，如私有化 glm-v5）不走 llm_raw_chat**：服务端路由直接拒绝（app not found），IDE 是用本地 aiserver 解密目录里的 `ak` 密文后直连企业网关（`custom_config.base_url`）实现的，HTTP 协议层拿不到明文密钥。插件把它们的 token 配额与用量呈现在用量卡里；要聊天请向部署管理员要企业网关（OpenAI 兼容）的直接 API key，另配一个普通 openai-compatible 供应商即可。

## 登录方式

1. **登录企业 TRAE（浏览器 SSO）**：填企业 HTTPS 地址 → 读取该部署的 `GetLoginGuidance.Result.LoginHost` → 插件起本机回调服务，将授权 URL 返回 Magpie → Magpie 在用户所在机器打开浏览器 → 完成企业认证 → 回调后兑换并保存 `access + refresh`。插件不调用 macOS `open`、Windows `cmd` 或 Linux `xdg-open`。
2. **从本机 traecli 导入登录**：读取 `~/.trae/cli/auth.json`（可指定目录），校验部署一致性与凭据有效性后导入。导入后插件独立续期；由于续期会轮换 refresh token，traecli 那份在下一次续期时可能失效，需要时在 CLI 里重新登录一次即可。

旧版（traecli 目录绑定）账号在新版下会提示重新登录，用上面任一入口一键迁移。

## 远程 Linux / Docker：服务器运行 Magpie，本地浏览器认证

回调服务始终只监听服务器的 `127.0.0.1`，不需要公网端口。本地浏览器认证后若最终停在无法打开的 `http://127.0.0.1:<port>/authorize?...`，把**完整地址**粘贴回 Magpie 的回调输入框或 CLI 提示。支持此功能的宿主会核对本轮授权 URL 中的 loopback 端口，再在服务器本机请求该地址；插件仍经原回调完成兑换。也可由使用者自行配置仅本机可达的 SSH 端口转发，但默认不要求。

完整回调地址包含一次性认证材料，不要公开、写入报告或发到聊天。不同主机独立登录，不把 Mac 的 token 或浏览器 cookie 复制到服务器。旧宿主若没有回调地址输入提示，需要先核对其版本能力。

Windows、macOS、Linux 共用 Node/Bun HTTP 回调代码，打开浏览器由宿主负责。代码共用不代表三个平台的真实登录均已验收，具体结果以交付记录为准。

## GUI 默认与高级覆盖

普通 GUI 安装不需要设置环境变量：应用 ID 等协议参数内置，请求体预算默认 9 MiB；上下文优先使用目录声明，未知时为 128000。扩展窗口在供应商选项中开启。下表只供已确认部署差异的高级覆盖；无效协议或数值覆盖会在登录阶段报出具体变量名。

| 变量 | 默认/用途 |
| --- | --- |
| `TRAE_PRIVATE_APP_ID` | 默认 `7b3f9dc2-8a4e-5c6d-2f1b-9e4a3c5b7df0`（raw-chat 应用 ID） |
| `TRAE_PRIVATE_AUTH_HOST` | 可选登录域覆盖；默认读取部署 `GetLoginGuidance.Result.LoginHost`，仅接受 HTTPS |
| `TRAE_PRIVATE_OUTPUT` | 输出上限，默认 32000 |
| `TRAE_PRIVATE_CONTEXT` | 仅扩展开关开启时的目标窗口，默认 1000000；不开启时不影响未知窗口的 128000 兜底 |
| `TRAE_PRIVATE_BODY_BUDGET` | 请求体 UTF-8 字节预算，默认 9 MiB（上游 nginx 约 10MB 处 413） |
| `TRAE_PRIVATE_TOOL_RESULT_KEEP` | 历史工具结果裁剪长度，默认 20000 |
| `TRAE_PRIVATE_USER_AGENT` / `_PROTOCOL_VERSION` / `_IDE_VERSION_CODE` / `_IDE_VERSION` / `_ORIGINATOR` | 协议头覆盖（非空可打印 ASCII，≤512） |

供应商选项「按上游完整上下文窗口上报」：默认尊重目录声明的 `prompt_max_tokens`，未声明或无效时按 128000；开启后至少上报 `TRAE_PRIVATE_CONTEXT`（默认 1000000），不降低更大值。开关不验证上游容量，仅在部署确认支持时开启。

## 行为保证

- 对话流式/JSON、工具调用分片与配对、图片输入、reasoning、`token_usage` 事件、内容安全事件转错误、排队事件忽略——继承旧版久经验证的管线。
- 请求体超预算时先丢历史图片（留占位）再截断历史工具结果，仍超限返回 413 说明。
- 部署地址只接受 HTTPS、无 userinfo/query/fragment，允许路径前缀；所有请求 `redirect: "error"`。
- 401/30011/1001 映射为账号失效（`X-Magpie-Sign-In`）；`IsAllowLogin=false` 同样标记失效。
- 到期前 2 小时由 magpie `refreshLead` 触发 ExchangeToken 续期；loader 内**绝不**做轮换式续期（新 refresh token 无法在 loader 里持久化，轮换即锁死账号）。
- `ErrAllModelUserQuotaExceeded`（entitlement check failed）是上游业务错误：基础/超额额度用尽（用量卡上可见真实占比，允许超支显示），不是登录或插件故障；等额度重置或联系管理员调额。

## 已验证 / 未验证

已验证（本机，2026-10-25）：

- 历史记录：GetUserInfo / CheckLogin / get_config_list / ExchangeToken（三次真实轮换抓包）/ GetLoginGuidance / llm_raw_chat 的鉴权与错误事件流。该记录不等于成功对话或 SSO 已验收。
- v0.2.2 本轮：45 项离线回归通过，覆盖流式、非流式（含 stream 缺省）、done 内嵌用量、续期钩子返回值、目录故障与身份隔离、保守窗口及登录配置预检，以及部署登录域选择、提前到达的回调、取消/超时结束等待、完整模拟 SSO 兑换链路。Node/Bun 包装测试均在独立 Node 子进程运行，使用临时 HOME、假凭据；文件访问受限，外网与子进程调用被屏蔽，仅允许本机回调测试。

合并前仍需真实部署记录：

- v0.2.2 现场已确认旧版固定公共授权页不适用于本次私有化部署，已改为读取实际 LoginHost；真实账号登录仍需用户在正确企业入口完成，不能以离线用例代替。当前账号额度未重新核验。
- 授权页回调携带的查询参数名（当前按 `code` 处理，另兼容回调直接带 `refresh_token`）与授权码兑换的准确 body 字段。静态分析无法唯一确定（服务端只回笼统错误码），插件按可能性顺序尝试四种候选并在 `via` 字段记录命中的形状；仍需一次成功的浏览器 SSO 确认。

## 开发

```sh
node index.test.mjs   # 隔离子进程离线回归，不读取本机登录
bun test ./index.test.mjs   # 同一隔离回归的 Bun 包装入口
```

原始实现参考 [Magpie 插件仓库 PR 235](https://github.com/magpie-community/plugins/pull/235)，归属见 [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md)。
