# ARCHITECTURE：午饭投票 v1

> 状态：草案
> 取代：无

## 1 现状基线

新项目。没有代码、没有仓库、没有测试。T00 初始化后，基线定义为：`npm test`、`npm run typecheck`、`npm run lint`、`npm run build` 四条命令在空项目上全部退出码为 0。

## 2 技术选型

| 层 | 选择 | 理由 | 状态 |
|---|---|---|---|
| 语言 | TypeScript，`strict: true` | 契约前后端共用 | 新定（D-01） |
| 框架 | Next.js App Router，Node 运行时，`next start` 常驻进程 | 主流；页面和接口同仓 | 新定（D-01） |
| 数据库 | SQLite + Prisma | 20 人以内；单文件，备份简单 | 新定（A-02） |
| 校验 | zod | 入参校验和类型从同一份 schema 推导 | 新定 |
| 时区计算 | date-fns + date-fns-tz | 主流；支持 IANA 时区和夏令时 | 新定 |
| 样式 | Tailwind CSS | 页面少，不需要组件库 | 新定 |
| 单元与流程测试 | Vitest | 与 TypeScript 配合好，速度快 | 新定 |
| 端到端测试 | Playwright | 可设 375px 视口并截图 | 新定 |
| 定时 | 外部 cron 调用 tick 接口 | 见 D-06 | 新定 |

具体版本号在 T00 锁定并写进 `package.json`，不用 `latest`。

## 3 分层与目录

```
prisma/
  schema.prisma
  migrations/
src/
  contracts/   zod schema、错误码、共享类型。前后端都从这里引用
  domain/      纯函数。不读当前时间、不访问数据库和网络
    time.ts      截止时刻计算、开放判断、截止时间显示
    options.ts   选项校验、是否锁定
    tally.ts     计票：winner / tie / none
    redact.ts    匿名裁剪、结果视图和历史行组装
    message.ts   通知文案
  ports/       接口：Repo、Clock、Notifier、TokenService
  adapters/
    memory/      内存 Repo、可调时钟、记录型 Notifier（测试用）
    prisma/      SQLite Repo
    webhook/     群机器人 Notifier
    crypto/      令牌生成与 SHA-256
    system/      系统时钟
  services/    用例。权限判断只在这一层（D-08）
  server/      组装层：按环境把 adapters 注入 services；从 cookie 取身份
  app/         Next.js 页面和 route handlers。只做参数解析、调 services、把错误码映射成 HTTP 状态
scripts/
  reset-leader-link.ts   组长找回链接（A-13），通过 server 层调用 services
tests/
  unit/        对应 domain
  flow/        services + memory adapters；T07 起同一套用例再跑一遍 prisma adapter
  e2e/         Playwright；screenshots/ 存截图
```

依赖方向（T00 用 ESLint `no-restricted-imports` 按目录强制，违规时 `npm run lint` 失败）：

- `app` → `server` → `services` → `domain`、`ports`、`contracts`
- `adapters` → `ports`、`contracts`
- `domain` → 只能引用 `contracts` 里的类型
- `app` 不允许直接引用 `adapters` 和 `prisma`

环境变量（T00 写进 `.env.example`，只放占位值）：

| 变量 | 用途 |
|---|---|
| `DATABASE_URL` | SQLite 文件路径 |
| `APP_URL` | 对外地址；用于通知里的链接和 `Origin` 校验 |
| `SETUP_TOKEN` | 一次性 setup 入口的口令 |
| `CRON_SECRET` | tick 接口的口令 |
| `LV_TEST_MODE` | 只在端到端测试中设为 `1`，开启测试时钟接口 |

## 4 数据模型

主键全部为 cuid 字符串（D-02）。时间全部存 UTC。

| 实体 | 字段 | 约束与索引 |
|---|---|---|
| Team | id, name, timezone（IANA 名）, webhookUrl（可空）, createdAt | 一个部署一行（A-01） |
| Member | id, teamId→Team, displayName（1–20 字）, role（`leader` / `member`）, tokenHash, tokenIssuedAt, removedAt（可空）, createdAt | `tokenHash` 唯一；索引 `teamId`；同团队在册成员 `displayName` 不重复（services 层检查） |
| Poll | id, teamId→Team, pollDate（团队本地日期 YYYY-MM-DD）, deadlineAt, createdByMemberId→Member, createdAt | `(teamId, pollDate)` 唯一（A-09）；索引 `deadlineAt` |
| PollOption | id, pollId→Poll, label（1–30 字）, position（0–5） | `(pollId, position)` 唯一 |
| Vote | id, pollId→Poll, memberId→Member, optionId→PollOption, anonymous, createdAt, updatedAt | `(pollId, memberId)` 唯一（AC-05）；索引 `(pollId, optionId)` |
| Notification | id, pollId→Poll, kind（`poll_opened` / `poll_result`）, status（`pending` / `sent` / `failed`）, attempts, nextAttemptAt, lastError（可空，截断到 500 字）, sentAt（可空）, createdAt | `(pollId, kind)` 唯一（D-07）；索引 `(status, nextAttemptAt)` |

冗余字段：无。20 人规模下票数和参与人数在读取时计算。

派生而不落库（D-03）：

- 投票是否开放：`now < deadlineAt`
- 选项是否锁定：该投票下是否已有票
- 结果、并列项、参与人数、在册人数。在册人数 = 该团队中 `createdAt <= deadlineAt` 且（`removedAt` 为空或 `removedAt > deadlineAt`）的成员数；参与人数 = 该投票的票数
- 成员是否在册：`removedAt` 为空

团队时区修改只影响之后发起的投票；已发起投票的 `deadlineAt` 不重算。

## 5 状态机

**Poll**：只有一个由时间决定的派生状态。

| 状态 | 条件 | 允许的动作 |
|---|---|---|
| 开放 | `now < deadlineAt` | 投票、改票；无票时组长改选项 |
| 已截止 | `now >= deadlineAt` | 查看结果；不接受任何写入 |

**Notification**：落库的状态。

| 从 | 到 | 触发者 | 条件 |
|---|---|---|---|
| （无） | pending | createPoll（`poll_opened`）、tick（`poll_result`） | 同一 `(pollId, kind)` 只插入一次 |
| pending | sent | tick | webhook 返回 2xx |
| pending | pending | tick | 发送失败且 `attempts < 5`；`nextAttemptAt` 按 1、2、4、8 分钟退避 |
| pending | failed | tick | 第 5 次失败 |
| pending | failed（原因 `expired`） | tick | 种类为 `poll_opened` 且投票已截止；或种类为 `poll_result` 且距截止已超过 24 小时（A-17） |

团队未配置 `webhookUrl` 时，通知保持 pending，不计失败次数，管理页提示去设置。

**Member**：在册 →（组长移除）→ 已移出。不可恢复；需要时重新添加为新成员。

## 6 接口契约

身份：请求 cookie `lv_session` 中的令牌 → SHA-256 → 查在册成员。错误响应统一为 `{ "error": { "code": "...", "message": "..." } }`。

| 动作 | 调用者 | 入参 | 出参 | 错误码 |
|---|---|---|---|---|
| `POST /api/setup` | 持部署环境变量 `SETUP_TOKEN` 的人 | teamName, leaderName, timezone | 组长专属链接（只返回这一次） | UNAUTHORIZED, SETUP_DONE, INVALID_INPUT |
| `GET /m/[token]` | 任何人 | 路径中的令牌 | 写 cookie 后 303 到 `/`；无效时渲染提示页 | — |
| `GET /api/me` | 成员 | — | memberId, displayName, role, team.name, team.timezone | UNAUTHORIZED |
| `GET /api/members` | 组长 | — | 成员列表（含已移出标记） | UNAUTHORIZED, FORBIDDEN |
| `POST /api/members` | 组长 | displayName | 成员 + 专属链接 | NAME_TAKEN, MEMBER_LIMIT（上限 50，A-16）, INVALID_INPUT |
| `POST /api/members/[id]/reset-link` | 组长 | — | 新专属链接 | NOT_FOUND |
| `DELETE /api/members/[id]` | 组长 | — | 204 | NOT_FOUND, FORBIDDEN（不能移除自己） |
| `PUT /api/team/settings` | 组长 | timezone?（须为合法 IANA 名）, webhookUrl?（须为 https） | 团队设置 | INVALID_INPUT |
| `POST /api/polls` | 组长 | options: string[] | PollView | INVALID_OPTIONS, POLL_EXISTS_TODAY, PAST_DEADLINE |
| `PUT /api/polls/[id]/options` | 组长 | options: string[] | PollView | INVALID_OPTIONS, OPTIONS_LOCKED, POLL_CLOSED, NOT_FOUND |
| `GET /api/polls/today` | 成员 | — | PollView 或 null | UNAUTHORIZED |
| `PUT /api/polls/[id]/vote` | 成员 | optionId, anonymous | myVote | POLL_CLOSED, OPTION_NOT_FOUND, NOT_FOUND |
| `GET /api/polls/[id]/result` | 成员 | — | ResultView | RESULT_NOT_READY, NOT_FOUND |
| `GET /api/history` | 成员 | cursor?（pollDate） | items: HistoryRow[]，每页 30；nextCursor | UNAUTHORIZED |
| `POST /api/cron/tick` | 外部 cron | 请求头 `Authorization: Bearer <CRON_SECRET>` | created, sent, failed 计数 | UNAUTHORIZED |
| `POST /api/test/clock` | Playwright | now（ISO 时间） | 204 | 未设 `LV_TEST_MODE=1` 时 404 |
| `npm run leader:reset-link` | 服务器上的运维人员 | — | 在终端打印新的组长链接，旧令牌失效 | 没有团队时退出码非 0 |

错误码到 HTTP：UNAUTHORIZED 401，FORBIDDEN 403，NOT_FOUND / OPTION_NOT_FOUND 404，INVALID_* 400，其余 409。

PollView 和 ResultView 的形状（`contracts` 中定义，`domain/redact.ts` 组装）：

```ts
type PollView = {
  pollId: string;
  pollDate: string;
  deadlineAt: string;
  deadlineLabel: string;      // 例如「11:30（UTC+8）」
  isOpen: boolean;
  options: { optionId: string; label: string }[];   // 开放中不带票数（A-14）
  myVote: { optionId: string; anonymous: boolean } | null;
  votedCount: number;
  optionsLocked: boolean;
};

type ResultView = {
  pollId: string;
  pollDate: string;
  outcome: { kind: "winner"; optionId: string }
         | { kind: "tie"; optionIds: string[] }
         | { kind: "none" };
  options: { optionId: string; label: string; count: number; voterNames: string[] }[];
  participants: number;   // 投了票的在册与已移出成员
  rosterSize: number;     // 当天在册成员数
};
```

`voterNames` 只含实名票。匿名票只体现在 `count` 里，没有任何字段能还原投票人（AC-11、P-03）。

页面：`/setup`、`/m/[token]`、`/`（今日投票，截止后显示结果；未识别身份时只显示「请用组长发给你的专属链接打开一次」，AC-24）、`/polls/[id]`（结果）、`/history`、`/admin`（组长：发起、改选项、成员与链接、设置）。

## 7 关键流程

**专属链接换 cookie**：`GET /m/[token]` 校验 `lv1_` 前缀和长度 → 哈希查成员 → 写 `lv_session`（httpOnly、Secure、SameSite=Lax、Max-Age 180 天、Path=/）→ 303 到 `/`。该响应带 `Referrer-Policy: no-referrer`。改写类接口额外校验 `Origin` 与 `APP_URL` 一致。

**投票**：在一个事务里依次检查：成员在册 → `now < deadlineAt`（时钟来自 Clock 接口）→ optionId 属于该投票 → 按 `(pollId, memberId)` upsert。同一成员连点两次由唯一约束保证只有一条。

**改选项**：在一个事务里检查票数为 0 → 删除旧选项 → 插入新选项。SQLite 写入串行，所以与并发投票不会交错：先到的事务赢，后到的得到 `OPTIONS_LOCKED` 或 `OPTION_NOT_FOUND`。

**发起**：在一个事务里插入 Poll、PollOption 和一条 `poll_opened` 通知（pending，`nextAttemptAt = now`）。`deadlineAt` 由 `domain/time.ts` 按团队时区算出。

**tick**（D-06、D-07）：

1. 找出 `deadlineAt <= now` 且还没有 `poll_result` 通知的投票，各插入一条 pending；唯一约束冲突直接忽略。
2. 取 `status = pending` 且 `nextAttemptAt <= now` 的通知，最多 20 条。
3. 逐条认领：条件更新 `nextAttemptAt = now + 5 分钟`，条件是仍为 pending 且 `nextAttemptAt <= now`；影响行数为 0 说明被另一个 tick 认领，跳过。
4. 用 `domain/message.ts` 生成文案，调用 Notifier，按第 5 节更新状态。

发送成功但标记前进程崩溃时会重发一次。这是「至少一次」语义，对群通知可以接受。

「已开始」文案不列选项，因为选项在第一票之前还可能改。

**通知文案**（`domain/message.ts`，T03 的快照以此为准）：

| 种类 | 文案 |
|---|---|
| 已开始 | 今日午饭投票已开始，{截止时间标签} 截止。入口：{APP_URL} |
| 结果 winner | 今日午饭：{选项}（{票数} 票，{参与人数} 人参与）。详情：{APP_URL}/polls/{pollId} |
| 结果 tie | 今日午饭平票：{选项甲}、{选项乙}（各 {票数} 票），请组长在群里定。详情：{APP_URL}/polls/{pollId} |
| 结果 none | 今天无人投票。详情：{APP_URL}/polls/{pollId} |

## 8 验证阶梯

| 级别 | 手段 | 命令或方法 | 覆盖什么 |
|---|---|---|---|
| 一 | Vitest 单元测试 | `npm test -- tests/unit` | domain 全部函数：AC-01–04、AC-10、AC-11 |
| 二 | Vitest 流程测试，内存实现；T07 起同一套用例再跑 SQLite；route handler 测试 | `npm test -- tests/flow`、`npm test -- tests/api` | services 与路由：AC-05–09、AC-11–19、AC-25、AC-26、AC-28 |
| 三 | 类型检查、lint（含分层导入规则）、构建 | `npm run typecheck && npm run lint && npm run build` | AC-20 |
| 四 | Playwright，测试时钟加本地模拟 webhook，375px 视口截图 | `npm run e2e` | AC-14、AC-21、AC-23、AC-24、AC-27 |
| 五 | 真实部署后手工清单 | TASKS「真机手工清单」 | AC-22 |

## 9 风险与外部前置

| 项 | 类型 | 影响 | 对策 | 状态 |
|---|---|---|---|---|
| 不同聊天工具的群机器人 webhook 请求格式不同 | 平台 | 通知发不出去 | Notifier 只负责 POST；请求体由一个格式函数生成，先实现 `{"text": "..."}` 一种 | 需核实：E-01 时确认团队所用工具的请求格式 |
| Next.js 当前版本里 `cookies()`、route handler 参数的写法 | 技术 | 模型按旧版写法写错 | T00 锁定版本，T08 先读该版本官方文档 | 需核实 |
| date-fns-tz 与 date-fns 新版本的配套关系 | 技术 | 时区计算用错包 | T00 锁定版本时按官方文档确认推荐的包 | 需核实 |
| 专属链接被转发后他人冒用 | 技术 | 一人可投多票 | 组长可随时重置链接；20 人小组内接受此风险 | 已核实 |
| 反向代理访问日志记录 `/m/<令牌>` | 技术 | 令牌泄露 | 部署时对 `/m/` 路径关闭访问日志或脱敏 | 需核实：取决于 E-02 选用的代理 |
| SQLite 单实例 | 技术 | 不能多实例；文件损坏丢数据 | 每日备份数据库文件；当前量级不需要扩容 | 已核实 |
| 外部 cron 没在跑 | 运行 | 没有通知；结果页不受影响（结果是派生的） | 真机清单检查 cron；tick 每次运行写一行日志 | 已核实 |
| 测试时钟接口被带到生产 | 技术 | 任何人可改系统时间 | 只在 `LV_TEST_MODE=1` 时注册；生产启动检测到该变量直接退出 | 已核实 |
| E-01 群机器人 webhook 地址 | 外部前置 | T15 无法验证真实通知 | 组长在团队聊天工具里创建群机器人 | 待办 |
| E-02 部署主机、HTTPS 域名、cron | 外部前置 | T15 无法上线 | 需求方准备一台能长期运行 Node 的主机 | 待办 |

## 10 不引入的东西

- Auth.js 等账号体系：本期没有账号（SPEC 2 不做）。
- Redis、消息队列：outbox 表加每分钟 tick 足够。
- WebSocket、SSE、轮询：本期不做实时刷新。
- Serverless 部署：SQLite 文件和每分钟 cron 不适合。要上 serverless 先按 A-02 换 Postgres。
- Moment.js：已进入维护模式。
- UI 组件库：页面少，Tailwind 足够。
- tRPC、GraphQL：动作少，REST route handlers 足够。
- 前端状态管理库：服务端组件加表单提交足够。
