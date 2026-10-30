# TASKS：午饭投票 v1

规则：一个任务一个新会话。开头只读 CLAUDE.md、本任务条目和它列出的文件。完成时贴出完成标准的命令输出。

每个任务的完成标准都隐含 AC-20：`npm run typecheck && npm run lint && npm run build` 退出码为 0，下文不再逐条重复。

## 总览
| 编号 | 任务 | 阶段 | 依赖 | 状态 |
|---|---|---|---|---|
| T00 | 初始化仓库与工具链 | 准备 | 无 | 未开始 |
| T01 | 契约：schema 与错误码 | 准备 | T00 | 未开始 |
| T02 | 时间规则 | 规则 | T01 | 未开始 |
| T03 | 投票规则：选项、计票、匿名裁剪、文案 | 规则 | T01、T02 | 未开始 |
| T04 | 端口与内存实现 | 流程 | T01 | 未开始 |
| T05 | 用例服务层 | 流程 | T02、T03、T04 | 未开始 |
| T06 | 通知 outbox 与 tick | 流程 | T03、T05 | 未开始 |
| T07 | SQLite 数据层 | 数据（不可逆） | T04、T05、T06 | 未开始 |
| T08 | 群机器人 Notifier | 接入 | T04 | 未开始 |
| T09 | 身份与鉴权 | 接入 | T07、T08 | 未开始 |
| T10 | API 路由 | 接入 | T05、T06、T08、T09 | 未开始 |
| T11 | 端到端测试夹具 | 界面 | T10 | 未开始 |
| T12 | 成员页面：今日投票与结果 | 界面 | T11 | 未开始 |
| T13 | 组长页面：setup 与管理 | 界面 | T11 | 未开始 |
| T14 | 历史页 | 界面 | T11 | 未开始 |
| T15 | 端到端主流程 | 验收 | T12、T13、T14 | 未开始 |
| T16 | 部署与真机验收 | 上线 | T15、E-01、E-02 | 未开始 |

P-01、P-02、P-03 未拍板时按推荐项执行（A-12）。需求方若选了非推荐项，受影响的任务见 DECISIONS 待拍板表。

---

### T00 初始化仓库与工具链
- **目标**：建立 Git 仓库和四条基线命令，并用 lint 规则强制分层依赖方向
- **先读**：CLAUDE.md；ARCHITECTURE 1、2、3、8
- **要改**：仓库根目录（`package.json`、`tsconfig.json`、ESLint 配置、`vitest.config.ts`、`playwright.config.ts`、`.gitignore`、`.env.example`）；`src/` 下各层的空目录；`prisma/schema.prisma` 空壳
- **完成标准**：
  - `git log --oneline` 至少有一条提交
  - `npm test`、`npm run typecheck`、`npm run lint`、`npm run build` 退出码均为 0
  - 在 `src/domain/` 临时加一个文件引用 `src/adapters/`，`npm run lint` 失败；验证后删除该文件
  - `.env.example` 列出 ARCHITECTURE 3 环境变量表中的全部变量，值为占位符
  - `package.json` 中没有 `latest`；ARCHITECTURE 9 中两条「需核实」的版本问题已查官方文档并把结论写进 DECISIONS
- **依赖**：无
- **不要做**：不写业务代码；不写 Prisma 模型

### T01 契约：schema 与错误码
- **目标**：在 `src/contracts/` 定义全部接口的入参出参 schema、共享类型和错误码
- **先读**：ARCHITECTURE 4、6
- **要改**：`src/contracts/`；`tests/unit/contracts/`
- **完成标准**：`npm test -- tests/unit/contracts` 通过，ARCHITECTURE 6 中每个动作的入参 schema 至少有一条合法样例和一条非法样例；`npm run typecheck` 通过
- **依赖**：T00
- **不要做**：业务规则（如选项不重复）不写在 schema 里，留给 T03

### T02 时间规则
- **目标**：实现截止时刻计算、开放判断、团队本地日期、截止时间显示四个纯函数
- **先读**：SPEC 4 的 C-02、SPEC 5 的 AC-02 至 AC-04；DECISIONS D-04、A-03、A-04；ARCHITECTURE 3（`domain/time.ts`）
- **要改**：`src/domain/time.ts`；`tests/unit/time.test.ts`
- **完成标准**：`npm test -- tests/unit/time` 通过。用例名以 AC 编号开头，覆盖 AC-02 的全部日期、AC-03 的两个边界、AC-04 的格式；另有一条跨日用例：UTC 2026-11-01T16:30Z 在 `Asia/Shanghai` 下的本地日期是 2026-11-02
- **依赖**：T01
- **不要做**：函数内部不调用无参的 `new Date()` 或 `Date.now()`，当前时刻一律由参数传入

### T03 投票规则：选项、计票、匿名裁剪、文案
- **目标**：实现选项校验与锁定判断、计票、结果视图组装（含匿名裁剪）、通知文案
- **先读**：SPEC 4 的 C-03、C-04、C-05、C-09、C-12；SPEC 5 的 AC-01、AC-10、AC-11、AC-15、AC-16；DECISIONS A-05、A-07、A-10、A-14、P-02、P-03；ARCHITECTURE 6（PollView、ResultView）、7（通知文案表）
- **要改**：`src/domain/options.ts`、`tally.ts`、`redact.ts`、`message.ts`；`tests/unit/`
- **完成标准**：`npm test -- tests/unit` 通过，包括：
  - AC-01 每种违规各一条用例
  - AC-10 的 winner、tie、none 三种
  - AC-11：构造含匿名票的输入，断言 `JSON.stringify(view)` 中不出现匿名投票人的 memberId 和名字
  - 开放中的 PollView 不含任何票数和他人选择（A-14）
  - 文案：「已开始」、winner、tie、none 四种各一个快照，与 ARCHITECTURE 7 通知文案表逐字一致
- **依赖**：T01、T02（文案用到截止时间格式）
- **不要做**：不实现平票裁决（P-02 选 b 时另加任务）

### T04 端口与内存实现
- **目标**：定义 Repo（含事务）、Clock、Notifier、TokenService 接口，给出内存实现和令牌实现
- **先读**：ARCHITECTURE 3、4、6；DECISIONS D-02、D-05
- **要改**：`src/ports/`；`src/adapters/memory/`；`src/adapters/crypto/`；`tests/unit/adapters/`
- **完成标准**：`npm test -- tests/unit/adapters` 通过：内存 Repo 对 ARCHITECTURE 4 中每个唯一约束都会报冲突；事务中途抛错时数据回滚；令牌匹配 `^lv1_[A-Za-z0-9_-]{43}$`，同一令牌的哈希稳定
- **依赖**：T01
- **不要做**：内存 Repo 不加任何业务判断，只模拟数据库行为

### T05 用例服务层
- **目标**：实现 setup、成员管理、发起与改选项、投票、今日视图、结果、历史的用例，权限判断全部在这一层
- **先读**：SPEC 5 的 AC-05 至 AC-13、AC-18、AC-19、AC-25；ARCHITECTURE 4（含在册人数的算法）、5、6、7（投票、改选项、发起）；DECISIONS D-03、D-08、A-06、A-09、A-11、A-13、A-14、A-16
- **要改**：`src/services/`；`tests/flow/`
- **完成标准**：`npm test -- tests/flow` 通过，上述每条 AC 至少一个用例，用例名以 AC 编号开头；全部运行在内存实现和可调时钟上
- **依赖**：T02、T03、T04
- **不要做**：不引用 Prisma 和 Next.js

### T06 通知 outbox 与 tick
- **目标**：实现 tick 用例：补插结果通知、认领、发送、按状态机更新
- **先读**：SPEC 5 的 AC-15 至 AC-17、AC-26；ARCHITECTURE 5（Notification）、7（tick）；DECISIONS D-06、D-07、A-08、A-17
- **要改**：`src/services/tick.ts`；`tests/flow/tick.test.ts`
- **完成标准**：`npm test -- tests/flow/tick` 通过，覆盖 AC-15、AC-16、AC-17、AC-26，另加两条：两个 tick 并发调用只发送一次；团队未配置 webhook 时通知保持 pending 且不计失败
- **依赖**：T03、T05
- **不要做**：不写真实的 HTTP 发送（那是 T08）

### T07 SQLite 数据层
- **目标**：写 Prisma schema 和首个迁移，实现 Prisma 版 Repo，让全部流程测试在 SQLite 上也通过
- **先读**：ARCHITECTURE 4、7；DECISIONS D-02、D-04、D-05、D-07、A-02、P-01
- **要改**：`prisma/schema.prisma`；`prisma/migrations/`；`src/adapters/prisma/`；`tests/flow/` 中切换实现的入口
- **完成标准**：`npx prisma migrate dev --name init` 生成迁移；`REPO=prisma npm test -- tests/flow` 全部通过，与内存实现跑同一套用例；`npm run typecheck` 通过
- **依赖**：T04、T05、T06
- **不要做**：P-01 若被拍板为 B 或 C，先停下更新 ARCHITECTURE 4 再继续；迁移生成后不改已有迁移文件，只新增

### T08 群机器人 Notifier
- **目标**：实现向 webhook 地址 POST 文案的 Notifier
- **先读**：ARCHITECTURE 5（Notification）、9（webhook 格式一行）；DECISIONS A-08
- **要改**：`src/adapters/webhook/`；`tests/unit/adapters/webhook.test.ts`
- **完成标准**：`npm test -- tests/unit/adapters/webhook` 通过：用本地 HTTP 服务断言请求体为 `{"text": "..."}`；2xx 视为成功；非 2xx 和 5 秒超时视为失败并返回错误信息
- **依赖**：T04
- **不要做**：不在这里做重试，重试由 tick 负责

### T09 身份与鉴权
- **目标**：实现组装层、从 cookie 取身份、专属链接换 cookie、setup 接口、测试时钟接口
- **先读**：ARCHITECTURE 3（环境变量、scripts）、6（身份、`/api/setup`、`/m/[token]`、`/api/test/clock`、`leader:reset-link`）、7（专属链接换 cookie）、9（Next.js 一行和测试时钟一行）；SPEC 5 的 AC-13、AC-14、AC-28；DECISIONS D-05、A-13、P-01；T00 锁定版本的 Next.js 官方文档中 cookies 和 route handlers 两节
- **要改**：`src/server/`；`src/app/m/[token]/`；`src/app/api/setup/`；`src/app/api/test/clock/`；`scripts/reset-leader-link.ts` 及 `package.json` 中的 `leader:reset-link`；`tests/api/auth.test.ts`
- **完成标准**：`npm test -- tests/api/auth` 通过：有效令牌返回 303，`Set-Cookie` 含 `HttpOnly`、`Secure`、`SameSite=Lax`；无效令牌渲染提示页；重置后旧令牌返回 401；未设 `LV_TEST_MODE` 时 `/api/test/clock` 返回 404；`NODE_ENV=production` 且 `LV_TEST_MODE=1` 时组装层启动抛错；AC-28：脚本输出新组长链接后旧令牌返回 401
- **依赖**：T07、T08（组装层要注入 webhook Notifier）
- **不要做**：不把令牌写进任何日志；不加账号登录

### T10 API 路由
- **目标**：按 ARCHITECTURE 6 实现其余全部 route handlers，包括 cron tick
- **先读**：ARCHITECTURE 6 全节；SPEC 5 的 AC-11、AC-13、AC-25
- **要改**：`src/app/api/`（T09 已建的除外）；`tests/api/`
- **完成标准**：`npm test -- tests/api` 通过：每个动作至少一条成功用例和一条错误码用例；错误码到 HTTP 状态的映射逐个测试；以组长身份请求结果和历史，响应中不含匿名投票人（AC-11）；开放中的 `/api/polls/today` 响应不含各选项票数（AC-25）；改写类接口在 `Origin` 不符时拒绝
- **依赖**：T05、T06、T08、T09
- **不要做**：路由里不写业务判断，判断都在 services

### T11 端到端测试夹具
- **目标**：让 Playwright 能在独立 SQLite 文件、测试时钟和本地模拟 webhook 上跑应用
- **先读**：ARCHITECTURE 6（`/api/test/clock`）、8 第四级
- **要改**：`playwright.config.ts`；`tests/e2e/fixtures/`
- **完成标准**：`npm run e2e -- tests/e2e/smoke.spec.ts` 通过：每次运行前重建数据库；能设置测试时钟；模拟 webhook 能读出收到的请求；有一个辅助函数通过 setup 接口建好团队并返回组长和成员的链接
- **依赖**：T10
- **不要做**：不写页面

### T12 成员页面：今日投票与结果
- **目标**：实现 `/`（今日投票，截止后显示结果）和 `/polls/[id]`
- **先读**：SPEC 2、SPEC 5 的 AC-14、AC-21、AC-24；ARCHITECTURE 6（页面清单、PollView、ResultView）；DECISIONS A-04、A-06、A-10、A-14
- **要改**：`src/app/page.tsx`；`src/app/polls/[id]/`；`tests/e2e/vote.spec.ts`
- **完成标准**：`npm run e2e -- tests/e2e/vote.spec.ts` 通过：打开 `/m/<令牌>` 后 `page.url()` 等于 `APP_URL/`（AC-14）；无 cookie 访问 `/` 只显示找链接提示（AC-24）；当天没有投票时显示「今天还没有发起投票」；能投票、改票、切换匿名；开放中只显示「已有 N 人投票」；截止后显示结果；在 375px 视口生成 `tests/e2e/screenshots/vote.png` 和 `result.png`，并断言 `document.documentElement.scrollWidth <= 375`
- **依赖**：T11
- **不要做**：不做实时刷新

### T13 组长页面：setup 与管理
- **目标**：实现 `/setup` 和 `/admin`（发起、改选项、成员与链接、团队设置）
- **先读**：SPEC 2、SPEC 5 的 AC-08、AC-21；ARCHITECTURE 5（未配置 webhook 时的提示）、6；DECISIONS A-03、A-05、A-16、P-01
- **要改**：`src/app/setup/`；`src/app/admin/`；`tests/e2e/admin.spec.ts`
- **完成标准**：`npm run e2e -- tests/e2e/admin.spec.ts` 通过：setup 后组长链接只显示一次；能添加、重置、移除成员并复制链接；有票后改选项按钮不可用且接口返回 `OPTIONS_LOCKED`；生成 375px 截图 `admin.png` 并断言无横向滚动
- **依赖**：T11
- **不要做**：普通成员访问 `/admin` 时只显示无权限提示，不在前端做权限之外的判断

### T14 历史页
- **目标**：实现 `/history`
- **先读**：SPEC 5 的 AC-18、AC-19、AC-21；ARCHITECTURE 6（`/api/history`、ResultView）；DECISIONS A-07、A-11
- **要改**：`src/app/history/`；`tests/e2e/history.spec.ts`
- **完成标准**：`npm run e2e -- tests/e2e/history.spec.ts` 通过：按日期倒序；能翻到第二页；已移出成员名字带「已移出」；每天显示参与人数 / 在册人数；生成 375px 截图 `history.png` 并断言无横向滚动
- **依赖**：T11
- **不要做**：不做口味统计图表

### T15 端到端主流程
- **目标**：按 SPEC 5 AC-23 的九个步骤写一条完整用例
- **先读**：SPEC 5 的 AC-23、AC-27；DECISIONS A-15
- **要改**：`tests/e2e/main-flow.spec.ts`
- **完成标准**：`npm run e2e` 全部通过（含此前各页面用例）；主流程用例里统计首次打开 `/` 时 JS 响应的传输字节数，断言不超过 200 KB（AC-27）；`npm run typecheck && npm run lint && npm run build` 通过
- **依赖**：T12、T13、T14
- **不要做**：为让用例通过而修改 AC-23 的步骤或预期；发现 SPEC 有误时先改 SPEC 并在 DECISIONS 登记

### T16 部署与真机验收
- **目标**：给出可复现的部署方式，并在真实环境完成真机手工清单
- **先读**：ARCHITECTURE 9；本文件「真机手工清单」；`.env.example`
- **要改**：`deploy/`（Dockerfile、cron 示例、数据库备份脚本、反向代理配置示例）
- **完成标准**：`docker build -f deploy/Dockerfile .` 成功；本地 `docker run` 后对 `/api/cron/tick` 带正确密钥请求返回 200、错误密钥返回 401；真机手工清单全部打勾
- **依赖**：T15、E-01、E-02
- **不要做**：不在部署配置里设置 `LV_TEST_MODE`

## 外部前置（人来办）
| 编号 | 事项 | 阻塞哪些任务 | 状态 |
|---|---|---|---|
| E-01 | 组长在团队聊天工具里创建群机器人，拿到 webhook 地址，并确认该工具要求的请求格式 | T16；格式与 `{"text"}` 不同时需回到 T08 | 待办 |
| E-02 | 准备一台能长期运行 Node 的主机、一个 HTTPS 域名，并能配置每分钟一次的 cron | T16 | 待办 |
| E-03 | 需求方对 P-01、P-02、P-03 拍板 | 不阻塞，按推荐项先做；选非推荐项时见 DECISIONS | 待办 |

## 真机手工清单

在工作日 11:30 之前开始，团队时区按实际设置。

- [ ] 用 setup 令牌在生产地址创建团队；组长链接只显示一次，刷新后不再显示
- [ ] 在管理页填入 E-01 的 webhook 地址，把时区设为团队实际时区
- [ ] 用手机打开自己的专属链接，跳转后地址栏只剩根地址
- [ ] 在群里点通用入口地址，直接进入今日页面
- [ ] 发起一场投票，1 分钟内群里收到「已开始」通知（AC-22）
- [ ] 用两部手机各投一票，其中一票匿名
- [ ] 11:30 后 1 分钟内群里收到结果通知，内容与结果页一致（AC-22）
- [ ] 以组长身份打开结果页和历史页，看不到匿名票的投票人
- [ ] 对生产地址 `POST /api/test/clock` 返回 404
- [ ] 在反向代理日志里搜索 `/m/lv1_`，没有结果
- [ ] 运行备份脚本，生成当天的数据库文件副本
