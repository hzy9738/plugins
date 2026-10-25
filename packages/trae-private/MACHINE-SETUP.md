# 换机安装与手工验收

协议原生版不再要求安装 traecli：浏览器 SSO 登录即可。traecli 仅在选用「从本机 traecli 导入登录」时才需要，且导入后插件独立续期（续期会轮换 refresh token，CLI 那份可能在它下次续期时失效）。每台机器独立登录，不复制真实 auth.json、Magpie 登录记录或浏览器 profile。GUI 默认无需环境变量：目录未声明上下文时按 128000，扩展窗口可在供应商选项中开启。

| 内容 | 随仓库传递 | 说明 |
| --- | --- | --- |
| index.mjs、index.test.mjs、test/、package.json、README、许可证 | 是 | 无部署主机、无真实凭据 |
| CLI auth.json（仅导入路径用） | 否 | 每机独立产生 |
| Magpie plugin-auth.json | 否 | 本机登录状态（access + 轮换后的 refresh） |

## 安装

```sh
magpie plugin add <本仓库路径>
```

离线回归（不需要网络与登录）：

```sh
node index.test.mjs
bun test ./index.test.mjs
```

两个入口都使用独立 Node 子进程和临时目录；网络、子进程及沙盒外文件访问由 `test/isolate.mjs` 限制。需要 Node，Bun 只用于仓库测试入口。

## 登录

1. `magpie plugin login trae-private`（或在 Magpie 界面）选「登录企业 TRAE（浏览器 SSO）」，填企业 HTTPS 地址。插件读取部署 LoginHost 并返回授权地址，由 Magpie 打开浏览器；不依赖服务器安装浏览器。
2. Magpie 在远程服务器时，本地浏览器最后的 loopback 页面可能打不开。把完整回调地址粘贴回 Magpie 即可由宿主在服务器本机完成回调；不要公开该地址。新登录各机独立，不复制 Mac token。
3. 已有本机 traecli 登录的，选「从本机 traecli 导入登录」一键迁移。

## 手工验收清单

1. 浏览器 SSO：IdP 跳转、回调收码、ExchangeToken 兑换成功（账号里 `via` 字段记录命中的兑换形状）；取消与超时路径不残留本地服务。
2. 用量：`magpie accounts` 出现席位/额度窗口（GetUserInfo，JWT 直读，无 cookie）。
3. 模型：`magpie provider models trae-private` 与部署模型菜单一致。上游暂时故障时同一身份沿用 last-good，合法空目录会撤回模型，鉴权失效明确要求登录。宿主重启后缓存为空，需要成功读一次目录。
4. 对话（需账号有可用额度）：流式回答、工具调用、`token_usage`；`ErrAllModelUserQuotaExceeded / entitlement check failed` 是额度业务错误，不是登录故障。
5. 续期：到期前 2 小时 magpie refreshLead 自动调 ExchangeToken；确认新 refresh token 已持久化（连续两次续期不锁账号）。

更新采用正常 git 拉取并重启/重新启用插件。发布、部署和上游 PR 由所有者另行决定。许可证与来源见 [THIRD-PARTY-NOTICES](THIRD-PARTY-NOTICES.md)。
