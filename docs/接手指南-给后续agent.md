# 接手指南：给后续 agent 和新工程师

> **本文档的用途**：读完这一份，你就知道平台是什么、代码在哪、负责人定了哪些硬规矩、怎么在本地跑和验证、下一步读什么。这是整套交接文档的入口。
>
> 整理日期：2026-11-26。代码基线：主仓 `cyndi-global-website` `main@fc37598`，`showcase/` 仓 `main@4e5ed57`。
> 本文不含任何密码、密钥、账号值。需要本地测试账号时，向负责人要。

---

## 1. 平台是什么

Cyndi Global 给企业客户（第一个客户是 Hudson River Trading，简称 HRT）做定制周边（swag）。平台分三层：Cyndi 员工在后台管商品、订单、库存和物流；客户员工在客户专属的选品站（Showcase）和 Portal 里下单、看项目进度；以后客户的管理员（HRT 的 Fei 和 Megan）会有自己的管理后台，并能给外部人员发"兑换链接"。线上域名是 `www.cyndiglobal.com`。已经做完的功能、没做的功能见 `待办与决策清单.md`。

## 2. 三个 git 仓库，技术栈，代码在哪

| 仓库 | 内容 | 本地位置 | 备注 |
|---|---|---|---|
| `cyndi-global-website`（GitHub `Georgiazhu/cyndi-global-website`） | 官网、商城、后台、Portal、全部后端接口、数据库迁移 | 工作区根目录 | 推到 `main` 后 Cloudflare Pages 自动部署 |
| `showcase`（GitHub `Georgiazhu/showcase`） | Showcase 前端（React + Vite） | 主仓里的嵌套目录 `showcase/` | **独立仓库**。主仓构建时会把它克隆进 `/showcase/` 路径 |
| `swag-platform-spec`（GitHub `Georgiazhu/swag-platform-spec`） | 本仓库：规格、差距分析、设计快照、交接文档 | 主仓里的嵌套目录 `swag-platform-spec/` | 只放文档，不放业务代码 |

后两个目录在主仓里是未跟踪的嵌套仓库，**不要提交进主仓**。

**技术栈**：Cloudflare Pages + Pages Functions（后端）+ D1（SQLite，库名 `cyndi-logistics`，绑定名 `DB`）+ R2（图片，绑定名 `MEDIA`）。前端是 React 19 + Vite + Tailwind。后台页面是静态 HTML，不是 React。D1 是 SQLite，不是 PostgreSQL。

**代码在哪**（主仓内）：

| 路径 | 内容 |
|---|---|
| `functions/api/*` | 后端接口。`admin/`（Cyndi 后台，要管理员 cookie）、`showcase/`（Showcase 下单、订单、登录）、`portal/`（Portal）、`orders/`、`password-reset/`、`products/`，以及 `login.js`、`register.js`、`me.js`、`tracking.js` 等 |
| `functions/_lib/*` | 共享业务逻辑：`auth.js`（后台会话）、`customer-auth.js`（客户会话）、`inventory.js`、`orders.js`、`order-status.js`、`shipments.js`、`yanwen.js`、`mailer.js`、`portal.js`、`dashboard.js`、`client-showcase.js` 等 |
| `migrations/` | 数据库迁移，目前到 `0016` |
| `tests/*.test.mjs` | 单元测试，现有 `inventory.test.mjs`、`showcase-pool.test.mjs` |
| `product-showcase/` | 主站前端（Vite 多入口）。`src/` 是 React 页面；`public/admin/` 是静态后台页面；`public/portal/` 是静态 Portal 页面 |
| `showcase/`（嵌套仓库） | Showcase 的 React 应用，用 `App.jsx` 里的 `view` 状态切页面 |
| `scripts/` | 构建和数据脚本。`build_showcase.mjs` 在主站 `postbuild` 时克隆 Showcase 仓库 |

## 3. 负责人定的硬规矩

这些是负责人（owner）明确要求的。**当作硬规则，不要自己判断"这次可以例外"。**

1. **先在本地实现并验证，负责人确认之前不要 commit。**
2. **验证只用 lint、build、curl 和 node 测试。永远不要用 Playwright。**
3. **永远不要运行 `wrangler d1 ... --remote`**（包括 `execute` 和 `migrations apply`）。线上数据库的改动由负责人把迁移 SQL 手动粘贴到 Cloudflare D1 Studio Console 里执行。因此：
   - 每个迁移必须可重复执行（`CREATE TABLE IF NOT EXISTS` 等）。
   - 不能重复执行的语句（例如 `ALTER TABLE ... ADD COLUMN`）要单独成段，注明"一条一条粘，报 duplicate column name 就跳过"。
   - 不要用 `migrations apply`，它会把本地遗留的 `0010_price_backup.sql` 一起带上线。
   - `CLOUDFLARE_DEPLOY.md` 里还留着一条旧的 `--remote` 命令，**不要照做**。
   - 本地库可以用 `--local`。
4. **推送顺序**：先推 `showcase` 仓，再推主仓（主仓构建时会克隆 showcase）。推主仓前先执行 `git pull --no-rebase`，合并机器人提交的 `ci: trigger Cloudflare Pages rebuild`。
5. **没有负责人明确说"推"，不要 push。**
6. **提交时按路径逐个 stage 文件，不要 `git add .`**。提交信息风格：`feat(area): ...`（也用 `fix(area):`、`docs:` 等）。
7. **永远不要提交这些文件**：`product-showcase/package-lock.json`、`migrations/0010_price_backup.sql`、`scripts/reprice_*.sql`、`scripts/crawl_*.py`、`scripts/output/`、`.agents/`、`.kiro/`、`.dev.vars`、`.crawl-venv/`，主仓里的 `showcase/` 路径，以及 `swag-platform-spec/`。另外 `scripts/make_admin_user.mjs` 目前也是未跟踪的，已作废，不要提交。
8. **沟通**：负责人用中文，要短的、白话的回答。
9. **用词**：代码、注释、文档里用包容性用语（用 primary/replica、allowlist/denylist 等，不用 master/slave、whitelist/blacklist）。
10. **本仓库不写凭证**：不写密码、密钥、临时密码、哈希、个人邮箱。这个仓库可能公开。需要提到时只写"默认账号要轮换"这类说法。

## 4. 怎么在本地跑

在主仓根目录：

```bash
# 主站前端（开发服务器，端口 5174）
cd product-showcase && npx vite --port 5174

# Showcase 前端（开发服务器，端口 5175，把 /api 代理到 8788）
cd showcase && npx vite --port 5175

# 后端：先构建出 dist，再在仓库根目录启动 wrangler（本地 D1 `cyndi-logistics`）
cd product-showcase && npx vite build && cd ..
npx wrangler pages dev product-showcase/dist --port 8788 --ip 127.0.0.1
```

要点：
- 后端读根目录 `.dev.vars`（已被 `.gitignore` 忽略）。至少要有变量 `CUSTOMER_SESSION_SECRET`（随机串），没有它所有客户登录会返回 503。其他变量名见 `设计快照/兑换与Showcase-admin设计-rev4.md` 第 7.2 节和 `CLOUDFLARE_DEPLOY.md`（`RESEND_API_KEY`、`EMAIL_FROM`、`YANWEN_AUTH_CODE`、`ALLOW_RESET_CODE_IN_RESPONSE` 等，只在需要时设置）。
- 本地 D1 是 wrangler 自带的本地库，和线上不通。给本地库跑迁移：`npx wrangler d1 execute cyndi-logistics --local --file=migrations/<文件>.sql`。
- 本地有测试用的管理员账号和客户账号，**只存在于负责人的本机**。需要登录时向负责人要，不要自己造，也不要写进任何文件。
- 前端改完后，要让 wrangler 看到变化，需要重新 `npx vite build`。

## 5. 怎么验证

| 改了什么 | 命令（在对应目录） |
|---|---|
| 主站前端、后台静态页 | `cd product-showcase && npx vite build` |
| 主站 lint | `cd product-showcase && npm run lint` |
| Showcase 前端 | `cd showcase && npm run lint && npm run build` |
| 后端共享逻辑 | 在主仓根目录 `node --test tests/*.test.mjs`（根目录没有 `package.json`，直接用 Node 内置测试） |
| 接口行为 | 启动本地 wrangler 后用 `curl` 调接口 |

测试的已知情况：`tests/showcase-pool.test.mjs` 用了 `node:sqlite`，需要 **Node 22.5 以上**。在 Node 20 上这个文件会失败（2026-11-26 实测：34 个用例，33 通过，1 个文件因缺 `node:sqlite` 失败），失败原因是环境，不是代码。换 Node 22 再跑。

提交前：先本地 build 和 test 通过，再等负责人确认。

## 6. 两套账户系统，必须保持分开

| | Cyndi 员工 | 客户 |
|---|---|---|
| 表 | `admin_users` | `customers` |
| Cookie | `logistics_admin`（token 的 aud 是 `admin`） | `customer_session`（aud 是 `customer`，有效期 30 天） |
| 能进的接口 | 只有 `/api/admin/*` | `/api/showcase/*`、`/api/portal/*`、将来的 `/api/showcase-admin/*` |

两种 cookie 互相进不了对方的接口。**不要为了省事把客户加进 `admin_users`，也不要让后台 cookie 通过客户接口。** HRT 的客户管理员（Fei 和 Megan）将来用各自的客户账户，再在新表 `client_admins` 里标记为管理员，不复用 `portal_role`，也不进 `admin_users`。共享的 HRT 店面账户不能当管理员。

## 7. 要读的文档和顺序

按这个顺序读，前三份读完就能开工：

| 顺序 | 文档 | 读它是为了 |
|---|---|---|
| 1 | 本文 | 规则和环境 |
| 2 | [`当前设计总览.md`](当前设计总览.md) | 平台现状、架构图、角色、风险 |
| 3 | [`待办与决策清单.md`](待办与决策清单.md) | 下一步做什么、等谁拍板、什么已经做完 |
| 4 | [`设计快照/设计附录-用户决定.md`](设计快照/设计附录-用户决定.md) | 负责人的架构决定（三层、多客户），优先级高于其他设计文档里冲突的部分 |
| 5 | [`设计快照/兑换与Showcase-admin设计-rev4.md`](设计快照/兑换与Showcase-admin设计-rev4.md) | 兑换门户、Showcase admin、多客户隔离的完整设计（很长，按章节查） |
| 6 | [`设计快照/设计评审-最新.md`](设计快照/设计评审-最新.md) | 第 4 版设计还有哪些要改 |
| 7 | [`设计快照/新Portal调查报告.md`](设计快照/新Portal调查报告.md) | 新 Portal 和我们计划的冲突、安全问题 |
| 8 | [`设计快照/库存设计-第一阶段.md`](设计快照/库存设计-第一阶段.md)、[`设计快照/库存使用指南.md`](设计快照/库存使用指南.md) | 已上线的库存怎么工作 |
| 9 | [`落地差距分析.md`](落地差距分析.md)、`API契约与数据模型速查.md`、Kotis 规格书 | 背景：对标 Kotis 的路线。注意差距分析是 2026-10-29 的快照，里面"没有库存"已经过时 |

`设计快照/` 里的文件是 2026-11-26 的副本，每个文件开头两行写明了来源路径。**活的版本在主仓 `cyndi-global-website` 里，可能有更新的修订，改设计前先去看。** 注意主仓里的 `.kiro/` 和 `.agents/` 不提交，只在负责人本机上，拿不到时以快照为准。

主仓里还有这些部署文档（不在快照里）：`TRACKING_DEPLOY.md`（燕文物流查询部署）、`CLOUDFLARE_DEPLOY.md`（部署、环境变量、统一登录）。

## 8. 常见陷阱

- **迁移编号**：`migrations/` 里最大是 `0016`。我们的多客户和兑换迁移必须用 `0017_client_scope.sql`、`0018_redemption.sql`。设计文档 rev4 里还写着 `0014`/`0015`，那是过时的。
- **运行时建表**：Portal 的一部分表和列（`client_projects`、`client_showcase_pools`、`orders.project_no` 等）只在代码里首次请求时自动创建，没有迁移文件。
- **共享的 `orders` 表**：商城 `SW-`、Showcase `SHOWCASE-`（旧 `SC-`）、Portal `ORD-`、将来的兑换 `RDM-` 都写这张表。订单号校验函数在几个文件里各有一份，改订单号规则时要全部检查。
- **Portal 是另一位贡献者（Georgia Zhu）的代码**，其中一些行为和我们的设计冲突（见总览第 4、8 节）。改她的代码行为之前，先问负责人。
- **三个未审过的库存提交**（c3c3943、28a7a6f、44f6370）在等作者答复，不要还原也不要扩展，见待办清单第 11 项。
