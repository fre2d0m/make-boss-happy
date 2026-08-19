# happyboss 看板 — 详细设计

> 讽刺设计，诚实执行。看板不是监控系统，是老板的"脱水阅读器"。
> 不搞打分，不搞雷达图，不搞红绿灯 KPI。只有感觉量化和证据链。

## 0. 核心设计决策（先吵架，再动工）

| 决策点 | 结论 | 理由 |
| --- | --- | --- |
| 员工是否需要账号 | **不需要**。员工侧只有 Publish Token | 员工的真实工作发生在自己的 Agent 里，让员注册账号是这个产品死掉最快的方式 |
| 老板侧账号体系 | 邮箱 + 魔法链接（无密码） | 老板不需要记密码，老板需要少一个烦恼 |
| 提交是否可改 | **追加式，不可改**。改 = 新增一版，旧版保留 | 防表演性修饰。快乐卡一旦发布即存档，历史版本可查 |
| 评级形式 | 三档感觉枚举，禁止数字 | 数字会被 AI 优化，感觉不会（大概） |
| 愿景的归属 | 老板亲自录入，员工/skill 只读引用 | 期望是老板的，实存是员工的，混在一起公式就失效了 |

## 1. 技术栈

- **框架**：Next.js 15（App Router + React Server Components）
- **UI**：shadcn/ui + Tailwind CSS。看板是工具型 SaaS：安静、密实、可扫读，
  不做营销页，不做 hero，登录后第一屏就是看板
- **数据库**：PostgreSQL（本地开发用 Docker，生产建议 Neon / Supabase）
- **ORM**：Drizzle ORM（轻、SQL 优先、迁移可控，和 Next.js RSC 搭配最顺）
- **认证**：Auth.js v5（magic link），仅老板/管理员角色需要登录
- **校验**：zod（API 入参 + 快乐卡 schema，skill 侧和 web 侧共用一份）
- **部署**：Vercel + Neon（也可自托管，Postgres 是唯一硬依赖）

## 2. 仓库结构（monorepo，保持 `npx skills add` 可用）

```
make-boss-happy/
├── make-boss-happy/          # skill 本体（SKILL.md，保持现状，勿动）
├── docs/
│   └── design.md             # 本文档
├── packages/
│   └── shared/               # zod schema：快乐卡结构，web 与 skill 共用
└── web/                      # Next.js 应用
    ├── app/
    ├── components/
    ├── db/                   # drizzle schema + migrations
    └── lib/
```

`skills` CLI 只扫 `SKILL.md`，加 `web/` 不影响安装。

## 3. 数据模型（PostgreSQL）

### 3.1 组织与账号

```sql
organizations (
  id            uuid pk,
  name          text not null,
  slug          text unique not null,
  created_at    timestamptz default now()
)

users (                          -- 只有老板/管理员
  id            uuid pk,
  email         text unique not null,
  name          text,
  created_at    timestamptz default now()
)

memberships (
  org_id        uuid references organizations,
  user_id       uuid references users,
  role          text not null check (role in ('owner', 'viewer')),
  primary key (org_id, user_id)
)
-- owner: 能写愿景、发 token；viewer: 只能看（比如合伙人、投资人想看热闹）
```

### 3.2 愿景（期望侧）

```sql
vision_items (
  id            uuid pk,
  org_id        uuid references organizations,
  title         text not null,            -- 一句话愿景条目，如"Q3 让客户自助看数"
  detail        text,                     -- 老板补充的期望细节
  status        text not null default 'active'
                check (status in ('active', 'achieved', 'stale', 'abandoned')),
  created_at    timestamptz default now(),
  updated_at    timestamptz default now()
)
-- 没有分数。新鲜度由 updated_at 推导：
-- 超过 N 天没碰过的 active 愿景，看板顶部提示"地图可能过期了"。N 默认 30，可配。
```

### 3.3 员工与发布凭证

```sql
members (                        -- 员工不是账号，是"名字 + token"
  id            uuid pk,
  org_id        uuid references organizations,
  display_name  text not null,    -- 老板认得的名字
  note          text,             -- "前端""运营"等老板视角的备注
  archived      boolean default false,
  created_at    timestamptz default now()
)

publish_tokens (
  id            uuid pk,
  member_id     uuid references members,
  token_hash    text unique not null,  -- 只存 hash
  label         text,                  -- "张三的 Claude Code"
  revoked_at    timestamptz,
  created_at    timestamptz default now()
)
```

### 3.4 快乐卡（实存侧，核心表）

```sql
submissions (
  id                uuid pk,
  org_id            uuid references organizations,
  member_id         uuid references members,
  client_request_id text,                  -- skill 生成的幂等键，防重复提交
  supersedes_id     uuid references submissions,  -- 修订时指向上一版
  summary           text not null,         -- 干了什么（三行以内，泡沫已脱除）
  removed_fluff     text,                  -- 点名被脱除的泡沫
  vision_item_id    uuid references vision_items,  -- 可为空：对不上就是对不上
  alignment         text check (alignment in ('far', 'ok', 'nailed')),
  alignment_reason  text,                  -- 一句话理由
  water             text check (water in ('fluff', 'squeeze', 'solid')),
  water_reason      text,
  boss_feeling      text check (boss_feeling in ('talk', 'doubt', 'pleased')),
  honest_note       text,                  -- 诚实备注，逆耳的话
  ai_disclosure     text,                  -- 用了哪些 AI 能力（透明化工具链）
  raw_input         text,                  -- 员工原始输入，存档备查
  created_at        timestamptz default now(),
  unique (org_id, member_id, client_request_id)
)

evidence_links (                 -- 证据链，比评级重要
  id            uuid pk,
  submission_id uuid references submissions,
  kind          text not null check (kind in ('url', 'deploy', 'usage', 'file', 'other')),
  label         text not null,
  ref           text not null,           -- 链接 / 部署地址 / 使用记录描述
  created_at    timestamptz default now()
)
```

感觉枚举的中英文映射只存在于 UI 层：
`far/ok/nailed = 差得远/还行/高度达成`，`fluff/squeeze/solid = 全是泡沫/挤挤还有/干货`，
`talk/doubt/pleased = 想找人聊聊/将信将疑/欣慰`。

## 4. API 设计

### 4.1 员工侧（skill 调用，Bearer Token 认证）

```
POST /api/v1/submissions
Authorization: Bearer mbh_xxx
```

请求体（zod schema 见 `packages/shared`，skill 与 web 共用）：

```json
{
  "client_request_id": "uuid，由 skill 生成，重试必须相同",
  "summary": "报表生成从 4 小时压到 10 分钟，管线已上线。",
  "removed_fluff": "三页 PPT 美化",
  "vision_item_id": "可空；skill 应先拉愿景列表再对齐",
  "alignment": "ok",
  "alignment_reason": "管线支撑自助看数，但客户端还没接入",
  "water": "squeeze",
  "water_reason": "管线是干货，PPT 是泡沫",
  "boss_feeling": "doubt",
  "honest_note": "PPT 与愿景无关，建议下次别做",
  "ai_disclosure": "Claude Code 完成重写，人工 review 后上线",
  "evidence": [
    { "kind": "url", "label": "管线地址", "ref": "https://..." },
    { "kind": "usage", "label": "运营本周用了 6 次", "ref": "内部统计" }
  ]
}
```

```
GET /api/v1/vision-items        -- skill 对齐前先读愿景
GET /api/v1/me                  -- 返回 member 名与组织名，skill 用于确认身份
```

错误约定：token 无效 401；schema 不过 400 并返回哪条过不了；
重复 `client_request_id` 返回 200 + 已存在的卡片（幂等）。

### 4.2 老板侧

不开放 REST。页面全部走 RSC 直接查库，表单走 Server Actions。
少一层 API，少一层老板不需要的复杂度。

## 5. 看板页面结构

路由（`/login` 以外全部要求 owner/viewer）：

```
/login                      魔法链接登录
/                           老板快乐看板（第一屏就是它，没有落地页）
/vision                     愿景管理
/submissions                快乐卡列表（筛选：成员/感觉档位/时间）
/submissions/[id]           卡片详情 + 证据链 + 历史版本
/members                    员工与 Publish Token 管理
/settings                   组织信息、愿景新鲜度阈值
```

### 5.1 看板首页 `/` 的信息层级（从上到下）

1. **愿景新鲜度条**：每个 active 愿景一个胶囊，显示"X 天没动过"。
   超期的变色并附一句："可能不是员工偏航，是地图过期了。"
2. **本周快乐卡流**：按成员分组的卡片列。每张卡片只显示：
   脱水摘要（三行）、三个感觉徽章、证据链数量（0 时标红"口说无凭"）。
3. **诚实备注区**：所有卡片的 honest_note 单独汇成一栏。
   这是全站最重要的区域，视觉权重最高。
4. **底部一行小字**：本期共脱除 N 条泡沫。不解释，老板懂。

### 5.2 UI 约束

- 密实、安静、可扫读；卡片圆角 ≤ 8px；不用渐变装饰
- 感觉徽章用克制的三色系（差得远 = 警示色、还行 = 中性、高度达成 = 肯定色），
  但页面主色不能被任何一个档位带跑
- 不用数字汇总（没有"平均分 3.7"这种东西），聚合只给计数和分布条
- shadcn 组件映射：徽章 `Badge`、筛选 `Select`/`Tabs`、证据链 `Table`、
  愿景录入 `Dialog` + `Form`、token 发放 `AlertDialog`（只显示一次）

## 6. skill 侧契约（下一步改 SKILL.md 时遵循）

`/make-boss-happy <工作内容>` 的执行流程变为：

1. 读本地配置（`~/.config/make-boss-happy/config.json`：`endpoint` + `token`）
2. `GET /api/v1/vision-items` 拉老板愿景
3. 按现有 SKILL.md 流程脱水、对齐、评级
4. `POST /api/v1/submissions` 发布（带幂等键）
5. 终端打印快乐卡 + 发布结果链接

没有配置 token 时，退化为纯本地输出（现状行为），不报错不阻塞。

## 7. MVP 范围（第一刀切哪里）

做：登录、愿景 CRUD、成员 + token 发放、submissions 发布接口、看板首页、卡片详情。
不做：viewer 邀请流、修订版本对比视图、邮件通知、客户侧数据采集。

验收标准只有一个：老板打开看板 30 秒内能回答三个问题——
这周谁干了什么、哪些和愿景有关、哪句话是员工不敢当面说的。
