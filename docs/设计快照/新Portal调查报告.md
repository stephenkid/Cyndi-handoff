> snapshot of .agents/tasks/client-portal-investigation.md as of 2026-11-26; 本文已按公开仓库规则做过脱敏（去掉密码、个人邮箱、利用细节）。
> the live copy lives in cyndi-global-website, check it for newer revisions.

---

# 新客户 Portal 调查报告（只读）

范围：`git log 6dd7c25..HEAD` 中 2026-11-24/25 的 portal 相关提交。作者 `Cyndi Global` / `Georgiazhu`。本地 `main` 与 `origin/main` 一致。没有改动任何代码，也没有访问线上 D1。

## 一、结论摘要

1. 这是主站（`product-showcase/public/portal/*`）上的一个客户 Portal，有登录、公开注册、忘记密码和登录后 Dashboard。它和商城、Showcase 共用同一套客户账户：`customers` 表、`customer_session` cookie、`CUSTOMER_SESSION_SECRET`。`showcase/` 仓库没有改动。
2. 角色只有两级：`viewer`（默认）和 `senior`（多一个产品资料下载）。存在两套互不相通的"项目进度"模型：
   - `portal_projects`：管理员手填三个百分比。
   - `client_projects`：Dashboard 用的 stage 1/2/3。它的客户端页面已被删除，现在只剩 Dashboard 在用。
3. 自助注册的用户进不了 Showcase、库存和 `/api/admin/*`，这部分授权是对的。主要风险在下面几点：
   - **Dashboard 往真实的 `orders` 表写数据。** 已登录的客户（包括刚注册的 viewer）可以通过 Dashboard 的"确认订单"动作往真实的 `orders` 表插入 `ORD-26-*`、状态为 `confirmed` 的订单。演示用的假订单也会写进同一张表。这些订单会出现在 Cyndi 后台 Orders 的"全部"列表里，但打不开，后台报 "Invalid order number"。
   - **公开接口（注册、重置密码、验证、登录）都没有限流。** 重置密码的验证码可以反复申请，存在被暴力接管账户的风险（包括 HRT 的 Showcase 账户），也可以被用来向任意邮箱发信。邮件发送失败时验证码记录照样生成。
   - Dashboard 页面把数据直接拼进 `innerHTML`，没有统一转义（存储型 self-XSS 风险）。
4. 迁移编号冲突：`0014`、`0015`、`0016` 已被这次提交占用。我们计划中的 `0014_client_scope.sql` / `0015_redemption.sql` 必须改号（建议 `0017` / `0018`）。另外，运行时自迁移已经改了线上 schema，设计里"0013 是最新已应用迁移"的前提不再成立。
5. Resend 封装在 `functions/_lib/mailer.js`，这正是兑换设计里计划新建的同名文件。可以复用它的发送部分。但公开注册接口没有限流，会和兑换邀请邮件抢同一个 Resend 配额，所以要先加限流再复用。

## 二、证据

### 1. Portal 是什么、角色、三阶段进度

**页面**（主站静态页）：
- `/portal/login/`：调用 `/api/login`。如果 `mustChangePassword` 为真，跳到 `/portal/setup-password/`，否则跳到 `/portal/dashboard/`。
- `/portal/register/`、`/portal/forgot-password/`、`/portal/reset-password/`、`/portal/setup-password/`、`/portal/dashboard/`。
- `product-showcase/public/_redirects` 把 `/portal` 和 `/portal/` 301 到 `/portal/dashboard/`。旧的 `/portal/index.html` 在 92e14ba 中删除。
- 首页 `content.js` 里的 `PORTAL_LOGIN_URL = '/portal/login/'`。

**角色**（`functions/_lib/portal.js`、`functions/_lib/customer-auth.js`）：
- 列 `customers.portal_role`，取值 `'viewer'` 或 `'senior'`，默认 `viewer`。另加了 `customers.must_change_password`。
- `isSenior(row)` 的判断是 `portal_role==='senior' || showcase_access===1`。`ensurePortalSchema` 每个 isolate 启动时会执行 `UPDATE customers SET portal_role='senior' WHERE showcase_access=1`。
- 角色怎么分配：
  - 管理员在 `/admin/customers.html` 设置（`functions/api/admin/customers.js` 的 `normPortalRole`，以及新增的 `requirePasswordSetup`）。
  - 公开注册固定为 `viewer`。
  - 有 Showcase 权限的账户自动成为 `senior`。
- viewer 能做的：登录、看 Dashboard、执行 Dashboard 的全部写操作（`mutateDashboard` 没有任何角色判断）。
- senior 额外能做的：`GET /api/portal/download?productId=` 下载 `showcase_products` 的资料文本（`functions/api/portal/download.js`）。这个接口不按 `client_id` 过滤，可以下载任意产品。目前没有页面调用它，但接口仍然在线。

**三阶段模型 A**（`migrations/0014_customer_portal.sql`，`portal.js` 中的 `getProjectForCustomer`、`listProjects`、`saveProject`）：
- 表 `portal_projects(customer_id PK, title, showcase_pct, mockup_pct, order_pct, *_note, status, updated_at, updated_by)`。
- 管理员通过 `/admin/portal-projects.html` 和 `functions/api/admin/portal-projects.js`（`readSession`，即 admin cookie）手工设置百分比。
- 客户端读取接口是 `GET /api/portal/project`，但显示它的页面已被删除，所以这套模型现在是孤立的。

**三阶段模型 B**（`functions/_lib/dashboard.js`）：
- 表 `client_projects(project_no TEXT PK, user_id TEXT, title, stage 1|2|3, status, shortlist_json, mockups_json, …)`。
- 同时给 `orders` 加了 `project_no` 列。两者都没有迁移文件，只靠运行时 `ensureDashboardSchema` 创建。
- 阶段对应关系：1 = shortlist，2 = mockup，3 = order。
- `mutateDashboard` 支持的动作：`removeShortlist`、`confirmShortlist`、`removeMockup`、`uploadMockup`、`confirmMockup`、`confirmOrder`。其中 `uploadMockup` 只记录文件名，不上传文件。
- 没有"新建项目"的动作。`seedIfEmpty` 种入的 `PRJ-26-001..005` 是全局主键，所以只有第一个空账户能拿到这些演示项目，之后的新用户 Dashboard 是空的。
- Dashboard 里的 `showcases` 和 `portals`（例如 "New Hire Kits (/newhire-welcome)"）是写死的演示数据。

### 2. 账户、密码、会话

- 账户就是 `customers` 表，和商城、Showcase 是同一套，没有另起一套账户系统。
- 密码：`hashPassword` 使用 PBKDF2-SHA256，迭代次数最多 100000（可通过 `CUSTOMER_PBKDF2_ITERATIONS` 调低）。存储格式为 `pbkdf2$iter$salt$hash`，兼容旧的 SHA-256 格式，旧格式登录成功后自动升级。
- 会话：`createCustomerSession` 生成无状态 HMAC token，`aud:'customer'`，有效期 30 天，cookie 为 `customer_session`（`HttpOnly; SameSite=Lax`，https 下加 `Secure`）。密钥是 `CUSTOMER_SESSION_SECRET`，没配置时返回 503。
- 重置 token 和注册 token 用同一个密钥，但 `aud` 分别是 `'password-reset'` 和 `'portal-register'`，有效期 15 分钟，并绑定到具体的记录 id，不能当会话用。
- 互通情况：
  - Portal 用户和 Showcase 用户是同一个人、同一个 cookie。能否进 Showcase 由 `hasShowcaseAccess` 决定（要求 `showcase_access=1`，且所属 client 为 active），见 `functions/api/showcase/auth.js` 和 `auth-lib.js`。
  - `/api/admin/*` 只认 `logistics_admin` cookie（`aud:'admin'`，`readSession`，auth.js:118）。客户 cookie 进不了后台，后台 cookie 也进不了客户接口。

### 3. 公开注册能拿到什么

- 流程：`/api/portal/register/request` 发码 → `verify` 换取 token → `complete` 插入账户。插入的值固定为 `portal_role='viewer', showcase_access=0, client_id=NULL`，然后直接签发会话，并调用 `getProjectForCustomer` 种入 `portal_projects`。
- 注意：原来的商城注册 `/api/register` 本来就允许无验证的自助注册，所以"任何人能建客户账户"并不是这次新引入的。
- 自助注册用户无法访问：
  - Showcase 下单：`hasShowcaseAccess` 为 false，返回 403。
  - 库存：`/api/admin/inventory.js` 需要 admin cookie。
  - 任何 `/api/admin/*`。
- 授权缺口：
  - **G1：**已登录客户可以通过 Dashboard 往真实 `orders` 表插入 `confirmed` 订单（`total=0`，`client_id` 为空），需要一个处于第 3 阶段的项目。修复：停用或加权限限制。
  - **G2：**senior 下载产品资料时没有 client 隔离（见上文）。
  - **G3：**`/api/set-initial-password` 注释说"前提是 must_change_password=1"，但代码没有校验。任何会话都能在不提供旧密码的情况下改密码。
  - **G4：**重置密码不会让已签发的 30 天会话失效，因为 token 是无状态的。

### 4. 数据

- Dashboard 通过 `loadDashboard` 读取：
  - `client_projects WHERE user_id=?`
  - `orders WHERE customer_id=? LIMIT 200`

  这就是 Showcase 和商城共用的 `orders` 表。HRT 账户的 `SHOWCASE-` 订单（含 pending/cancelled）会出现在 Dashboard 里。
- Dashboard 的 "Inventory" 是把这些订单的件数直接相加，和 `0013_inventory.sql` 的 `inventory` / `order_stock_lines` 无关，数字会有误导。
- 写入方面：
  - 不会修改已有的 Showcase 订单，也不碰 `inventory` 和 `order_stock_lines`。
  - 会新增 `ORD-26-<nnn>-NN` 订单（`orderNoForProject` 全表 LIKE 扫描），状态 `confirmed` 不在 `ORDER_STATUSES` 中。
  - `seedIfEmpty` 用 `INSERT OR IGNORE` 插入两条假订单：`ORD-26-004-01`（2026-10-18，600 件 + 320 件）和 `ORD-26-005-01`（280 件）。因为 `order_no` 是主键，只有第一个空账户会拿到。
  - 后台 `functions/api/admin/orders.js` 的 `isValidOrderNo` 只接受 `SW-` 和 `SHOWCASE-`/`SC-`，所以这些行在"全部"列表可见，但打不开、改不了。`/api/orders/mine` 会过滤掉它们。
- 新增的迁移和运行时 DDL：

| 文件 | 内容 |
|---|---|
| `0014_customer_portal.sql` | `customers` 加 `portal_role`、`must_change_password`；新表 `portal_projects` |
| `0015_portal_password_resets.sql` | 新表 `portal_password_resets` |
| `0016_portal_registrations.sql` | 新表 `portal_registrations` |
| 无文件（只在 `dashboard.js` 运行时执行） | 新表 `client_projects`；`orders` 加 `project_no` |

  以上全部由 `ensurePortalSchema` / `ensureDashboardSchema` 在线上首次请求时自动执行，所以线上很可能已经生效。
- 冲突：
  - 设计文档 `.kiro/reports/redemption-design.md` 写的是 "Migrations: 0014_client_scope.sql and 0015_redemption.sql (0013 is the latest applied)"，编号已经被占。
  - 和 `0013_inventory.sql` 没有表冲突。
  - 我们 0014 Section A 要给 `showcase_categories`、`showcase_products`、`orders` 加 `client_id`，和 `orders.project_no` 不冲突。
  - 另外工作区里还有一个未跟踪的 `migrations/0010_price_backup.sql`，与 `0010_showcase.sql` 同号（这是本地遗留，与这次提交无关）。

### 5. 邮件（Resend）

- 文件：`functions/_lib/mailer.js`
  - `sendPortalCodeEmail(env, to, code, purpose)` 和 `sendPasswordResetEmail`。
  - 直接 `fetch('https://api.resend.com/emails')`，或者走通用的 `MAIL_API_URL`。
- 发件人取 `EMAIL_FROM`，默认是 `Cyndi Global <no-reply@cyndiglobal.com>`，因此 `cyndiglobal.com` 需要在 Resend 完成域名验证。从 d72c3ab 之后换了新 key 来看，发送应该已经能用了。
- 模板：内联的纯文本和 HTML，只有"注册"和"重置"两种。`code` 是服务端生成的数字，插入 HTML 是安全的。
- 限流：没有。每次请求都会建一条记录并发一封信。
- Cloudflare Pages 需要的环境变量：

| 变量 | 说明 |
|---|---|
| `DB` | 绑定 |
| `CUSTOMER_SESSION_SECRET` | 必需 |
| `RESEND_API_KEY` | 必需，用于发信 |
| `EMAIL_FROM` | 建议设置 |
| `MAIL_API_URL` / `MAIL_API_KEY` | 可选，替代 Resend |
| `CUSTOMER_PBKDF2_ITERATIONS` | 可选 |
| `ALLOW_RESET_CODE_IN_RESPONSE` | 生产环境必须不设置。设为 `1` 时验证码会直接出现在响应里 |

- 能否发兑换链接：传输层可以复用。但现有函数写死了验证码模板，需要抽出一个通用的 `mailSend(env, {to, subject, text, html})`。设计里的 `REDEEM_MAIL_FROM` 可以回退到 `EMAIL_FROM`，两者共用同一个 `RESEND_API_KEY` 和同一个配额。

### 6. 安全速查

| 项 | 现状 | 风险 |
|---|---|---|
| 限流 | `register/request`、`password-reset/request`、两个 `verify`、`/api/login` 都没有。全仓库搜索不到 throttle 或 `CF-Connecting-IP` | 高 |
| 验证码 | 6 位数字（`randomCode`，`crypto.getRandomValues`），15 分钟过期，每条 5 次尝试，用后作废，加盐 SHA-256 存储 | 单条记录的强度够 |
| 暴力重置 | 重置验证码接口没有限流，可无限申请新验证码；发信失败时记录照样生成。存在被暴力接管任意已知邮箱账户的风险，包括有 Showcase 权限的 HRT 账户。修复：按 IP 和邮箱限流，发信失败时不保留记录 | 高 |
| 邮箱枚举 | 重置接口统一返回 `{ok:true}`，但只有账户存在时才调用 Resend，响应时间有差异。`register/complete` 在验证码通过后对已存在的邮箱返回 409（可以接受）。旧的 `/api/register` 本来就会直接 409 | 中（旧问题） |
| 邮件轰炸 | 可以对任意外部邮箱无限发信，损害发件域信誉，耗尽 Resend 配额 | 高 |
| 诊断残留 | 早先有一个诊断分支，已完全撤回，当前代码中不存在 | 已清除 |
| 硬编码密钥 | 没有发现 | 无 |
| CSRF | 没有 Origin 校验。`corsHeaders` 只回显两个官方域。依赖 `SameSite=Lax`（跨站 POST 不会带 cookie） | 低 |
| XSS | Dashboard 页面用 `innerHTML` 拼接用户提供的数据（项目名、文件名、订单商品名），`esc()` 只用在部分字段。修复：全部输出都转义 | 中（self-XSS，但数据写进了 D1） |
| 其他 | G3（`set-initial-password` 缺前提校验）、G4（重置后旧会话不失效） | 低到中 |

### 7. 与我们计划的冲突

| 方面 | 新 Portal | 我们的计划（Showcase admin + 兑换） | 处理建议 |
|---|---|---|---|
| 账户 | `customers` + `customer_session` | 同一套（A1：Fei、Megan 各自一个客户账户） | **复用**，这是好事：一个登录同时用于 Portal、Showcase 和 Showcase admin |
| 角色 | `portal_role` viewer/senior；`isSenior` 把 `showcase_access` 也算作 senior | `client_admins` 表（D10） | **分开**。不要往 `portal_role` 里加 `admin`。`requireShowcaseAdmin` 只认 `client_admins`，与 senior 无关 |
| 账户预劫持 | 公开注册（有验证）加上旧的 `/api/register`（无验证） | A1 要在 Customers 页面给 Fei、Megan 建号 | 如果对方邮箱已存在（可能是别人抢注的），不要直接给它挂 client 和权限。先重置密码，或要求邮件验证后再授权 |
| 术语和路由 | `/portal/*`、`/api/portal/*`、`portal_*` 表；Dashboard 里还有写死的 "Portals" 区块 | 兑换 "portal"：`redemption_portals`、`/r/<token>`、`/redeem`、`/api/redeem`、`/api/showcase-admin/*` | 路径不冲突，但概念重名。我们一律叫 "Redemption"（兑换），API 和表都用 `redemption`/`rdm` 前缀，不使用 `/api/portal/*` |
| 迁移 | 已占用 0014、0015、0016 | 计划 0014、0015 | 改成 `0017_client_scope.sql`、`0018_redemption.sql`，并改掉设计文档中 "0013 latest" 的表述 |
| `orders` 表 | 新增 `ORD-26-*` 订单，状态 `confirmed`，`client_id` 为空，新增 `project_no` 列；种子假订单 | 统一 `isValidOrderNo`、加 `RDM-` 前缀、加 `awaiting_approval` 状态、`orders.client_id` | 我们在统一订单号校验和状态时，要决定 `ORD-` 和 `confirmed` 怎么处理。建议请对方先停用 `confirmOrder` 和 `seedIfEmpty` |
| 库存 | Dashboard 的 "Inventory" 是订单件数之和 | Showcase admin 的 Inventory 读的是 0013 的真实库存 | 两个界面数字会对不上。建议 Dashboard 改为调用 inventory 库（按 client 过滤），或者去掉这个区块 |
| 邮件 | `functions/_lib/mailer.js`（`sendPortalCodeEmail`），`EMAIL_FROM` | 设计计划新建同名文件，用 `mail*` 前缀和 `REDEEM_MAIL_FROM` | **复用**这个文件：在里面加通用的发送函数，`REDEEM_MAIL_FROM` 回退到 `EMAIL_FROM`。设计里的 `redemption_throttle` 同时用来给 Portal 接口限流 |
| 客户 UI | 主站的 `/portal/dashboard/`，有 Orders、Inventory、Showcases、Portals 四个区块 | `showcase/` 仓库里的 Showcase admin（Orders、Inventory、Redemption） | 两个"客户后台"并存。需要和对方确定 Fei、Megan 的入口在哪里。建议客户管理功能放在 Showcase admin，Portal Dashboard 只保留只读的项目进度 |
| Cyndi 后台 | 新的 `/admin/portal-projects.html`，所有 admin 页面的导航都加了这一项 | 我们也要在这些导航里加 Redemption 页 | 只是文本层面的合并，没有实质冲突 |
| `_redirects` | 文件已存在（`/portal` 跳转） | 备用方案要加 `/r/* /redeem 200` | 追加即可 |

## 三、建议的下一步

1. 先和提交者（Georgia Zhu）确认 Portal Dashboard 的定位：是演示还是正式功能。在得到答复前，我们照常推进，不改动她的代码。
2. 请她（或经她同意后由我们）优先修三处：
   - `register/request`、`password-reset/request`、`verify`、`/api/login` 加 IP 和邮箱维度的限流，并且发信失败时不保留验证码记录。
   - Dashboard 去掉 `seedIfEmpty` 的假订单，停用 `confirmOrder`（至少要求 senior 或有 client）。
   - Dashboard 的 `innerHTML` 全部改用 `esc()`。
3. 确认线上 Pages 没有设置 `ALLOW_RESET_CODE_IN_RESPONSE`，并检查 `orders` 中是否已经有 `ORD-26-%` 假订单（只读查询，确认后再决定是否清理）。
4. 修改兑换设计：迁移改号为 `0017`/`0018`；把"线上已有 portal 相关列和表、`orders.project_no`"写进前提；在现有 `mailer.js` 上扩展邮件功能；`REDEEM_MAIL_FROM` 回退到 `EMAIL_FROM`；把限流表也用于 Portal 接口。
5. 在 A1（给 Fei、Megan 建号）中加一条规则：给已存在的账户授予 client、Showcase 或管理权限之前，先确认邮箱归属（重置密码或发送验证码）。
6. `/api/portal/download` 在 0017 之后要加 `client_id` 过滤，或者暂时下线。
