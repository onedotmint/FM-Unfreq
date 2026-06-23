# 补核心链路E2E回归

## Goal

补一条浏览器级核心回归，覆盖用户最重要路径：进入首页、建立频率身份、写 seed、生成/发布 signal、进入结果或星图。现有质量门禁覆盖 API 单测和 Web build，但缺少跨前后端、浏览器行为层面的主链路验证。

## What I already know

* 根质量门禁 `npm run check` 已通过。
* 根依赖已有 `playwright`，但 `package.json` 还没有 e2e 脚本。
* Web 使用 Next.js App Router。
* API 使用 Fastify，核心 signal API 已有较完整 routes/service 单测。
* README 已有部署 smoke 命令，但开发态缺浏览器级核心回归命令。
* 首页入口是 `apps/web/src/app/page.tsx`，核心交互组件是 `HomeFrequencyEntry`。
* 已确认 E2E 采用 Web 层 API mock：浏览器真实跑 Web 页面，拦截后端 API 响应，不依赖真实 AI Key、数据库、Redis。

## Requirements

* 建立可在本地和 CI 复用的 E2E 测试入口。
* 覆盖核心首访链路：打开首页、处理未登录/频率身份、提交 seed、等待生成结果、确认结果页或星图可达。
* 测试应尽量使用稳定选择器或可访问名称，避免依赖脆弱 CSS。
* 测试数据不能污染生产配置；使用 Web 层 API mock 隔离外部依赖。
* E2E 命令应能纳入根质量门禁或以单独脚本运行。

## Acceptance Criteria

* [ ] 根 `package.json` 增加明确的 E2E 脚本。
* [ ] Playwright 配置和测试文件落在清晰目录中。
* [ ] 至少一条核心链路 E2E 测试能在本地运行。
* [ ] README 或 debug 文档补充运行方式和环境要求。
* [ ] 现有 `npm run check` 继续通过；如 E2E 不纳入默认 check，文档说明原因。

## Definition of Done

* Tests added/updated.
* Lint / typecheck / build green.
* E2E command verified or说明外部依赖限制。
* Docs updated for new command.

## Out of Scope

* 不做完整浏览器矩阵。
* 不覆盖所有 legacy 页面。
* 不引入新的外部测试服务。
* 不把 AI 真实 provider 调用作为 E2E 必需条件。

## Technical Notes

* Root dependency: `playwright`.
* Root scripts currently include `check`, `api:check`, `web:check`, `deploy:test` but no E2E script.
* API fallback behavior depends on provider配置；E2E 应避免真实 AI Key 依赖。
* Selected strategy: Web-layer API mock. Full-stack DB/Redis/provider E2E is out of scope for this task.
* Frontend quality spec requires UI accessibility and tests for user-facing flows.
* Likely files:
  * `package.json`
  * `playwright.config.ts`
  * `tests/e2e/*`
  * `README.md`

## Open Questions

* None.
