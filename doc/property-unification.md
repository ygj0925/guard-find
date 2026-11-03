# 房源服务统一迭代

## 服务边界与契约

- 管理后台和微信小程序共同使用 `guard-find-backend` 的数据库及房源服务。
- 小程序接口 `/miniapp/auth`、`/miniapp/property`、`/miniapp/im`、`/miniapp/notices`；管理接口 `/system/property`、`/system/im`。
- App 继续使用 `guard-find-server` 的 `/app/**`，不修改 App 的服务地址。
- 小程序身份与后台管理员身份分开，Sa-Token 登录域为 `miniapp`；房东 ID 是小程序用户 ID，不能填写 sys_user ID。
- 房源业务为平台级数据，仅默认租户后台可管理。独立业务表不经过租户行拦截，不开放给其他租户管理。
- 列表 page 从 1 开始，size 上限 100；消息游标按 ID 升序，首次/增量最多 100 条，客户端持续拉取直到不足一页。
- ID 在客户端作为字符串保存，避免 Java Long 转 JavaScript Number 丢失精度。
- 房源状态 1 草稿 / 2 上架 / 3 下架；仅上架内容公开，房东可查看自己的非公开房源。
- 预约状态 1 待确认 / 2 已确认 / 3 已拒绝 / 4 已取消；房东确认或拒绝，预约人取消。后台可按相同状态规则处理。

## 部署与数据

新增 Liquibase changeset 只作用于 backend 数据库。server 历史数据不会自动跨数据库共享；上线前按实际数据归属安排用户、房源、收藏、历史、预约及 IM 数据的一致性迁移，并保留原 ID。不要把 server 的整个业务 SQL 导入 backend。

backend 微信登录需要 `MINIAPP_WECHAT_APP_ID` 和 `MINIAPP_WECHAT_APP_SECRET`。开发默认 backend 端口 8000；生产小程序通过 VITE_SERVER_BASEURL 及微信各渠道变量配置已备案的 HTTPS 域名，并配置微信合法请求域名。微信登录身份与旧 server token 不通用，切换后需要重新登录。

## 已发现的问题与修复范围

房源地址校验错位、租赁方式筛选缺失、可选信息无法清空、图片空数组不生效、草稿详情泄露；收藏/历史列表未过滤非公开数据；预约并发覆盖；聊天已读接口缺少成员检查、会话删除影响对方摘要、消息无限加载、客服会话扫描截断；前端轮询串话与 ID 精度风险。迭代覆盖这些路径和相关前端入口，验证结果见后续记录。

## 功能与接口对应

下面路径均由 backend 提供，响应统一沿用项目 `R` 包装。分页返回 `data.list`、`data.total`。

| 功能 | 小程序 | 管理后台 |
| --- | --- | --- |
| 新增房源 | POST `/miniapp/property` | POST `/system/property`，额外传 `ownerId` |
| 编辑基本信息 | PUT `/miniapp/property/{id}`，仅本人 | PUT `/system/property/{id}` |
| 图片上传 | POST `/miniapp/property/images` | POST `/system/property/images` |
| 房源列表 | GET `/miniapp/property/list`，仅上架 | GET `/system/property/page` |
| 详情 | GET `/miniapp/property/{id}` | GET `/system/property/{id}` |
| 我的房源 | GET `/miniapp/property/mine` | 列表可查看所有房源 |
| 上下架、草稿 | POST `/miniapp/property/{id}/status` | POST `/system/property/{id}/status` |
| 删除 | DELETE `/miniapp/property/{id}` | DELETE `/system/property/{id}` |
| 收藏切换 | POST `/miniapp/property/{id}/favorite` | — |
| 收藏、浏览历史 | GET `/miniapp/property/favorites`、`/history` | — |
| 创建预约 | POST `/miniapp/property/{id}/booking` | — |
| 预约列表 | GET `/miniapp/property/bookings?role=guest或owner` | GET `/system/property/bookings` |
| 处理预约 | POST `/miniapp/property/booking/{id}/status` | POST `/system/property/booking/{id}/status` |
| 房源管家 | 详情展示管家身份 | GET `/system/property/manager`、POST `/system/property/manager/{userId}` |
| 用户私信、客服 | `/miniapp/im/**` | `/system/im/**`，客服会话分页 |
| 业务通知 | `/miniapp/notices/**` | — |

图片使用 multipart 字段 `file`，仅 JPG/JPEG/PNG，每张不超过 5MB，校验真实图片格式和尺寸；每套房源最多 9 张。两个入口复用后台默认存储，上传目录由服务端按端别、用户和日期生成，不能由客户端指定。需要在“存储管理”中配置可公开访问的 HTTP(S) 地址，并在微信后台配置 uploadFile/downloadFile 合法域名。

管理房源新增、编辑、状态、删除、管家与预约处理分别使用独立权限码。编辑时修改状态仍需状态权限，不能通过基本信息接口绕过。仅默认租户可使用这组业务接口，小程序身份和管理身份不能互用。

## 并发与性能调整

- 房源编辑、状态、收藏、预约创建使用行锁和事务；预约状态变更使用条件更新，重复确认/取消不会覆盖其他操作。删除存在进行中预约的房源会明确拒绝。
- 收藏和历史先在 SQL 中过滤已删除/非上架房源再分页，修复列表数量与 total 不一致；房东和预约用户信息批量查询。
- 客服会话在数据库分组、计数和分页，避免全量读取后按 5000 条截断。
- IM 每批最多 100 条，仅将实际返回的对方消息标记已读；删除会话仅隐藏自己的入口，新消息会恢复入口。
- 管理端房源弹窗按需加载，弹窗状态隔离，减少打开弹窗时整张表格重渲染。
- 聊天采用串行轮询，退出页面停止，切换会话丢弃过期响应；接收游标独立于发送回包，防止漏收并发消息。

## 上线步骤与未验证边界

1. 备份 backend/server 数据库；分别审核并运行各自的 Liquibase 变更。backend 新增小程序、房源、互动、IM 表及菜单；server 仅扩展图片字段容量。没有自动执行真实数据库迁移。
2. backend 配置微信 App ID/Secret、默认文件存储与公网地址；管理角色分配新增房源运营权限。
3. 管理端生产环境默认请求 `/api`，需将 `/api` 反向代理到 backend，并去除 `/api` 前缀；也可在构建时设置 `GUARD_FIND_API_BASE_URL`。小程序通过 `.env.production.local` 覆盖 `VITE_SERVER_BASEURL` 及对应微信渠道变量，不能直接发布示例域名。
4. 历史数据若需迁移，先按同一小程序 App ID 下的微信 openid 对齐用户、保留业务 ID，再迁移关联数据。不同微信应用的 openid 不能直接用于合并用户。App 继续连接 server，不会自动与 backend 共享新数据；如需要 App 与小程序也实时共享数据，需要另外确定共享服务或同步方案。
5. 使用真实微信开发者工具完成登录、发布图片、管理端编辑、预约处理、通知和双用户聊天验收。本次本地验证没有调用真实微信服务、对象存储或真实数据库。

## 其他存量问题

旧契约脚本合并 backend 和 server 扫描，会把仅 server 提供的接口误判为管理端可用。本次改为默认只检查 backend，`API_CONTRACT_BACKEND` 可显式指定其他单个仓库。最终 169 个有效前端请求全部匹配，HTTP 方法不匹配、缺失路径、未知权限码、可见菜单缺失页面均为 0。另有 4 个已登记下线的旧聊天接口和 10 个隐藏深链旧页面，沿用现有登记。

此前跨仓库检查发现的 5 个缺失页面（内容审核、举报管理、采集源、采集候选、采集日志）来自 server 的种子菜单，backend 当前菜单不包含它们。如将来需要迁入这些独立业务，应同时迁入后端能力和管理页面，不能仅复制菜单。

## 本地验证记录（2026-11-03）

- backend：49 项测试通过，包含房源校验/权限/预约、图片格式与大小、IM 成员边界、登录域、Bearer 解析和租户隔离；API reactor 编译通过。
- server：真实执行 13 项房源/IM 测试通过（显式 `-Dtests.skip=false`），App `/app/**` 路径保留，App 仓库无改动。
- 小程序：8 个测试文件、35 项测试通过；`pnpm type-check`、本次修改文件的 ESLint 检查、微信小程序 development 构建通过。
- 管理端：TypeScript 检查通过；Biome 无错误，有两处既有 optional-chain 建议；仅 backend 的接口契约检查通过。
- 管理端完整 Jest 回归：14 个测试文件、88 项全部通过。本机首次运行有 2 项图标 UI 测试超过 30 秒；最终使用 `--runInBand --testTimeout=120000` 完整重跑通过。Jest 直接映射与 tsconfig 一致的路径别名，避免 `configUmiAlias` 为单元测试先执行整站准备编译。
- 管理端 `npm run build:zy` 生产构建通过，保留生产 Mako/Tailwind 配置。
- 所有仓库 `git diff --check` 通过。

检查覆盖本地编译、静态契约与自动化回归，不代表完成真实微信、对象存储或数据库的端到端联调，也未进行真实浏览器性能基准测量。

## 房源运营页面拆分与表单更新

- 看房预约独立入口为 `/guardfind/bookings`，支持房源 ID、预约状态筛选和分页，确认、拒绝、取消沿用原有预约操作权限。
- 管家列表独立入口为 `/guardfind/managers`，支持分页查看、设置和取消管家身份；房源管理中不再挂载这两个查询弹窗。
- 新增与编辑房源共用 Ant Design DatePicker，可入住日期仍按 `YYYY-MM-DD` 提交，清空时提交 null；图片使用 Upload 图片卡片和 Image 预览，支持删除与编辑回显，不再展示图片地址输入框。
- 用户沟通页提供服务端全量会话搜索、未读筛选、滚动加载、草稿保留、发送失败提示和新消息定位；查看历史时保留滚动位置，离开当前页面或隐藏浏览器时暂停轮询。
- 发布时先启动 backend，Liquibase 自动执行新增的 `property_operation_pages.sql`，新增两个页面菜单并迁移原有角色、租户套餐的菜单关联；原权限码不变。此脚本未直接应用到运行中的数据库。
- 菜单管理直接查询数据库，导航路由则读取 Redis 中的角色菜单缓存，默认有效期为 2 小时。SQL 迁移不会触发菜单服务的缓存失效逻辑，因此旧部署可能出现“菜单管理能看到、导航看不到”，重新登录也不能保证刷新。可先在菜单管理中点击“清理菜单缓存”（会同时刷新当前用户的导航权限）；新增 `MenuCacheInitializer` 在后端数据库初始化完成后、应用就绪前清理角色菜单缓存，避免重启发布后继续读取旧菜单。该操作仅清理 `ROLE_MENU:*`，保留登录会话和其他缓存。
- 本轮验证：管理前端 TypeScript、修改文件 Biome、生产 Mako 构建、接口/菜单契约门禁通过；新增表单和消息轮询测试 8 项及相关回归测试 20 项通过。菜单迁移在 H2 MySQL 兼容模式中验证了权限节点归属、角色/套餐关联保留与未授权角色不新增授权（仅将 MySQL bit 字面量转换为 H2 布尔常量）；未直接连接实际 MySQL 验证。

## 后台房源允许不绑定房东

- 管理端 `POST /system/property` 的 `ownerId` 为选填；省略或 null 表示由后台管理，填写时仍须为有效且启用的小程序用户 ID。小程序发布房源继续绑定当前登录的小程序用户，不使用管理员 ID 代替房东。
- 新增 Liquibase 迁移 `property_optional_owner.sql`，放宽 `gf_property.user_id` 和 `gf_property_booking.owner_id` 的非空约束；上线前须先启动更新后的 backend 执行迁移，再发布前端。
- 未绑定房东的已上架房源可以浏览、收藏和预约，预约由后台统一处理；不会向空用户发送站内信。匿名用户不能查看其草稿，小程序用户不能修改该房源或冒充房东确认预约。
- 小程序中未绑定房东的房源通过“联系管家”进入官方客服对话。后台列表和详情显示由后台管理，并禁用依赖房东身份的“设为管家”操作。
- 新增、编辑房源的输入框、数字框、下拉框和描述补齐填写提示及示例。
- 验证：后端编译与 38 项回归测试、前端 6 项表单测试及 4 项头像菜单回归、前端 TypeScript/生产构建/接口契约检查、小程序 TypeScript/修改文件 ESLint 均通过。新迁移在 H2 MySQL 兼容模式中验证了已有房东数据保留及空房东房源、预约写入，未直接连接实际 MySQL 执行。

## 用户会话搜索与滚动加载

- `GET /system/im/conversations` 保留 `page`、`size` 和原响应的 `list`、`total`，新增可选参数：`keyword`（最多 64 字，忽略首尾空白，昵称模糊匹配或完整用户 ID 精确匹配）、`onlyUnread`（默认 false）、`beforeId`（正整数，上一批最后一个会话的 `lastMessageId`）。
- `list` 新增 `lastMessageId`；存在 `beforeId` 时从该消息 ID 之前继续取最近会话、忽略 page 偏移，`total` 始终表示全部匹配会话数，不受游标影响。查询参数使用绑定变量，昵称中的 `%`、`_`、`!` 按普通字符匹配。原分页调用仍兼容。
- 昵称/ID、未读条件均在数据库分页前生效，管理员发出的未读消息不计入管理员未读筛选；会话摘要仍取用户最新消息，避免误展示旧未读消息。
- 管理端每次加载 30 条，接近列表底部自动追加并按用户 ID 去重；输入搜索使用 350ms 防抖。切换条件重置游标并丢弃旧响应，失败可在原游标重试。位于列表下方时暂停会话排序刷新，回到顶部后恢复；隐藏页面时暂停请求。
- 需先更新并重启 backend，再更新管理端，使服务端搜索和游标参数生效；本次不涉及数据库结构迁移。App/server、小程序接口保持原样。
- 验证：后端 15 项测试通过，其中 5 项在 H2 MySQL 兼容模式执行实际 Mapper SQL，覆盖 65 个会话中的跨页昵称/ID 搜索、未读筛选及计数、游标续载、最新摘要与字面量搜索；前端 12 项会话/消息测试、TypeScript、修改文件 Biome 和接口契约门禁通过。未在实际 MySQL 或浏览器环境进行端到端联调。
