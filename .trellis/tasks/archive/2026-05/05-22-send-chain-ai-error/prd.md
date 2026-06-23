# Fix send chain AI error

## Goal

让信号发送链路在可重写的情况下不要提前抛 `AI_UNAVAILABLE`。现在的问题看起来出在扩写层自己先做了终审，挡住了外层的一次重写。

## What I already know

* 发送入口是 `POST /api/signals/generate`。
* 链路会走 `generateSignalDraft()` -> `generateReviewedSignalExpansion()`。
* 外层已经有一次 review + rewrite 逻辑。
* `apps/api/src/modules/ai/service.ts` 里 `generateSignalExpansionWithFallback()` 目前也做了一次本地 review。
* 这样一来，扩写只要“不够完美”就会先报错，外层重写根本进不去。
* 前端把 `AI_UNAVAILABLE` 映射成“信号生成暂时不可用。”。

## Assumptions

* 用户说的“发送后总是遇到 ai error”指的是生成链路，不是发布或共鸣接口。
* 需要保留 schema 校验和真正的 provider 失败处理。
* 不引入本地兜底文案。

## Open Questions

* 无。

## Requirements (evolving)

* 去掉扩写 helper 里的重复终审。
* 保留 provider JSON 校验和 blocked phrase 校验。
* 让外层 review/rewrite 接管“是否需要重写”的判断。
* 维持一次重写上限。

## Acceptance Criteria (evolving)

* 可重写的扩写结果不会在 helper 层提前变成 `AI_UNAVAILABLE`。
* 非法 JSON、schema 不合法、blocked phrase 仍然报错。
* `generateSignalDraft()` 的 rewrite 路径能覆盖第一次 review 失败的情况。
* 相关单测通过。

## Definition of Done (team quality bar)

* 测试已补。
* lint / typecheck / API tests 过。
* 行为变化已写进任务记录。

## Out of Scope (explicit)

* 不改前端文案。
* 不重写提示词。
* 不加新的本地生成兜底。

## Technical Notes

* `apps/api/src/modules/ai/service.ts`
* `apps/api/src/modules/ai/service.test.ts`
* `apps/api/src/modules/signals/service.test.ts`
* Specs: `backend/api-contracts.md`, `backend/error-handling.md`, `backend/quality-guidelines.md`
