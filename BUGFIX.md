# BUGFIX.md

## Bug 标题

变更类路由强制要求 `Origin`，DSH Desktop 代理按设计删掉 `Origin`/`Sec-Fetch-Site`，桌面端面板的变更调用全部 403（上游 issue #8、#9、#10）

## 证据链

- 上游 open issue 三条指向同一根因：
  - #8：`POST dsh-app://app/_dsh/dsh-ios/status 403 (Forbidden)`，官方桌面版失效。
  - #9：报告者在 DSH Desktop 代码里定位到 `forwardWebRequest` 转发时主动删除请求头，插件收到的请求没有 `Origin`，而严格路由要求 Origin。
  - #10：给出成因与修复形态——DSH Desktop 0.2.x 从 `dsh-app://app` 提供页面，经已认证的 Host 转发 API 调用；`forwardWebRequest`（deepseek-harness `apps/desktop/src/web-document.ts`）删掉 `Origin` 与 `Sec-Fetch-Site` 并附上 Host 会话 Cookie。本次修复前已拉取该文件固定提交（687f282e…）核对，转发逻辑与 issue 描述逐字一致：`for (const name of ['host', 'origin', 'cookie', 'sec-fetch-site']) headers.delete(name)` 后 `headers.set('cookie', cookie)`。
- 修复前离线复现（`~/workspace/.fixwork/ios/repro.mjs`：在回环 HTTP 服务器上挂载真实 `StreamRoutes.handleDeviceAction`，simHost/access 用桩，fence 在处理函数第一行执行，故以 text/plain 请求体区分「过 fence → 415 bad_content_type」与「被 fence 拦 → 403 forbidden」）：
  - 桌面代理形态（无 Origin、无 Sec-Fetch-Site、带 Cookie）→ **HTTP 403 code=forbidden** ← bug 本体
  - 无 Origin、无 Cookie → 403 forbidden（符合预期）
  - 无 Origin + Cookie + `Sec-Fetch-Site: cross-site` → 403 forbidden（符合预期）
  - 浏览器同源 Origin → 415（过 fence，符合预期）
  - 浏览器跨站 Origin → 403 forbidden（符合预期）
  - 脚本 exit 1。
- `git stash` 回退修复后重建复跑：逐字一致（仍仅桌面代理形态一例失败，exit 1），证明缺陷为上游既有、非本次引入。

## 根因

`src/stream-routes.ts` 的 `isTrustedBrowserRequest(req, requireOrigin)`：`requireOrigin=true`（全部变更类路由）时，请求无 `Origin` 头直接返回 `false`。它默认「无 Origin = 不可信」，但 DSH Desktop 的已认证代理按设计就不发 `Origin`，于是桌面壳内所有变更调用（grant/capture/switch-device/real-start/device-action 等）在 fence 第一关被拒，面板回退「画面不可用」。

## 改动文件

- `src/stream-routes.ts`（唯一逻辑改动）：
  - 新增 `isDesktopProxyRequest()`：当且仅当请求**无 `Origin`**、**无 `Sec-Fetch-Site`**、且带**非空 `Cookie`** 头时，认定为 DSH Desktop 代理形态（跨站页面无法为该 authority 植入 Host 会话 Cookie，且真实浏览器 fetch 必然带 `Sec-Fetch-Site`，两条共同排除浏览器伪造）。
  - `isTrustedRequest()`：回环检查（peer + Host 均为回环）不变；`requireOrigin` 时桌面代理形态可替代 Origin；其余请求仍走原 `isTrustedBrowserRequest` 全部检查。
- 同步项（非逻辑）：`package.json` 版本 0.1.0-rc.10 → 0.1.0-rc.11、`repository.url` 改指 `https://github.com/lhg-dsh-plugins/lhg-dsh-ios`；README.md 顶部追加中文 fork 头（上游原文一字未删）；新增 UPSTREAM.md、BUGFIX.md；LICENSE 逐字未动。pnpm-lock.yaml 无根包版本字段，无待同步项。

## 复验结果

- 复现脚本修复后 exit 0，5/5 全过：桌面代理形态 → 415（已过 fence）；无 Cookie 的无 Origin 请求仍 403；带 cross-site 元数据的仍 403；浏览器同源/跨站行为与基线一致。
- `pnpm run typecheck`（tsconfig + tsconfig.client）退出 0；`pnpm run build` 通过。
- 平台无关烟测 `scripts/dev-plugin-env-smoke.mjs` 9/9 通过。
- ready/ 文件集与上游 git 跟踪集逐项核对一致（+ UPSTREAM.md + BUGFIX.md），无 .git/node_modules/lib。

## 验证边界

- 未在真实 DSH Desktop 0.2.x 壳内端到端验证：代理形态依据上游 issue #10 的实测记录与 deepseek-harness 固定提交源码建模，fence 语义在回环服务器上用真实路由处理函数验证。
- 上游 `npm test` 全链与 `scripts/dev-routes-smoke.mjs` 均需 macOS + iOS 模拟器/真机（simctl、xcodebuild、serve-sim），Linux 沙箱无法运行；本版只动传输 fence 的纯请求头判定，不触碰模拟器/设备逻辑。真机/模拟器路径的行为回归需在 macOS 上补跑 routes smoke 确认。
- issue #10 还提到 routes smoke 中 stream/ws 步骤在其环境有 3 例与 serve-sim 响应形态有关的失败，与本 fence 改动无关，本版未处理。
