# 午饭投票

小组每天中午的午饭投票 Web 应用：组长发起，成员用专属链接投票，11:30（团队时区）截止后自动在群里公布结果。

## 先读
- 当前版本的规格与任务：`docs/v1/`
- 文件权威顺序：DECISIONS 已定项 高于 SPEC 高于 ARCHITECTURE 高于 `docs/v1/source/PRD.md`

## 命令
```
npm test                 # Vitest，单元 + 流程
REPO=prisma npm test -- tests/flow   # 流程测试跑在 SQLite 上
npm run typecheck
npm run lint             # 含分层导入规则
npm run build
npm run e2e              # Playwright，需先 build
npm run dev
```

## 约定
- `src/domain/` 只放纯函数：不调用无参的 `new Date()` / `Date.now()`，当前时刻由参数传入；services 从 Clock 接口取时间。
- 权限判断只在 `src/services/`。路由只取身份、调用、映射错误码。
- 投票开放状态和结果不落库，每次计算。
- 时间一律存 UTC；截止时刻按团队时区算（`domain/time.ts`）。
- 错误码只从 `src/contracts/` 引用，不手写字符串。
- 用例名以对应的 AC 编号开头，例如 `AC-05 同一成员投两次只留一票`。

## 坑
暂无。踩到后追加一行：现象、原因、绕法。

## 工作方式
- 一个任务一个会话，按 `docs/v1/TASKS.md` 的条目做
- 先写测试并看到失败，再实现
- 完成时贴命令输出作为证据；任务通过后提交一次
- 新决定写进 DECISIONS；规格有误先改 SPEC 再改代码

## 边界
本节是边界的唯一出处，SPEC 6 指向这里。
- 总是做：改完跑 `npm test` 和 `npm run typecheck`
- 先问我：改已有迁移或已建表的字段；新增运行时依赖；改 SPEC 第 5 节验收
- 绝不做：在响应、日志、错误信息里输出匿名票的投票人；令牌明文入库或写日志；把截止判断放到前端；在部署配置里设 `LV_TEST_MODE`
