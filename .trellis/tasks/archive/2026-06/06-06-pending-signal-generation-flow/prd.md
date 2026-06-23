# Build pending signal generation status flow

## Goal

把单轮 seed 发射从“首页同步等待完整结果”改成“先创建 signal，再进入独立生成页查看真实进度”。用户刷新页面或重新打开链接时，仍能看到当前生成状态；完成后自动进入结果页。

## What I already know

* 用户确认继续保持单轮聊天/单轮发射，不做多轮对话。
* 用户选择独立生成页，不在首页原地显示，也不复用结果页作为生成页。
* URL 使用 `/signal/:id/transmitting`。
* 进度获取使用轮询 `GET /api/signals/:id`，不做 SSE 或 WebSocket。
* 生成状态落库到 `signals` 表，保证刷新和重进页面后可恢复。
* 生成失败后，生成页显示失败并允许重试。
* 匹配顺序保持现状：先扩写，再找同频。

## Requirements

* 新增或调整 signal 创建流程，使首页提交后能先得到一个可访问的 signal id。
* 新增独立生成页 `/signal/:id/transmitting`。
* 生成页按轮询结果展示中粒度阶段：
  * 已接收
  * 正在扩写
  * 正在匹配
  * 正在写入
  * 完成
* 生成状态必须持久化到数据库。
* 完成后自动跳转 `/result/:id`。
* 失败时停留在生成页，展示失败态、原 seed、重新发射入口和返回首页入口。
* 不能把后端工程细节直接暴露给用户。

## Acceptance Criteria

* [ ] 首页提交后不再一直停在首页等完整生成完成。
* [ ] 成功创建 pending signal 后进入 `/signal/:id/transmitting`。
* [ ] 生成页可通过轮询真实后端状态更新阶段。
* [ ] 刷新生成页后仍能恢复当前状态。
* [ ] 生成成功后自动跳转结果页。
* [ ] 生成失败时显示失败页，可重试。
* [ ] 后端 API 合同可供未来 Android 客户端使用，不只服务 Web。

## Definition of Done

* Tests added or updated for status transitions and API visibility.
* Lint / typecheck / relevant tests pass.
* Database migration included if schema changes.
* Debug docs or specs updated if signal creation chain changes.

## Out of Scope

* 多轮聊天。
* SSE / WebSocket。
* REVIEWING 结果页展示。
* AI 扩写质检统一。
* 匹配顺序调整。

## Technical Notes

* Current frontend entry: `apps/web/src/components/signals/HomeFrequencyEntry.tsx`
* Current API client: `apps/web/src/lib/signal-data.ts`
* Current API route: `apps/api/src/modules/signals/routes.ts`
* Current creation service: `apps/api/src/modules/signals/service-creation.ts`
* Current signal schema: `apps/api/src/db/schema.ts`
* Existing debug map: `docs/debug/signal-send-chain.md`
