# 跨端实时消息与企业 OA 分期方案

日期:2026-11-06 | 范围:`guard-find-app`、`guard-find-server`、`guard-find-miniapp`、`guard-find-backend`、`guard-find-frontend`

设计源:`guard-find-frontend/openspec/changes/cross-platform-im-and-bpmn-oa/`(proposal/design/tasks 与五份能力规格,已通过 `openspec validate --strict`)。本文记录跨仓库结论、已验收批次的证据,以及尚未完成的边界;不把规划当实现。

## 1. 身份与职责边界

| 功能 | 客户端 | 服务端 | 身份域 |
| --- | --- | --- | --- |
| App IM、个人工具、鸿蒙能力 | guard-find-app | guard-find-server | `/app/**` 公众用户 |
| 公寓 IM | guard-find-miniapp | guard-find-backend | `/miniapp/**` 公众用户 |
| 企业 OA、后台性能 | guard-find-frontend | guard-find-backend | `/system/**` 内部员工与租户 |

两个 Java 后端不默认共库或共享令牌。OA 首期只使用后台 Web 员工身份,公众账号不因登录 App/小程序而获得审批权。

WebSocket 的定位是"数据已变化"的低延迟通知;鉴权、事务、幂等、落库、分页与补拉仍由 REST 负责。不把所有 HTTP 请求改成 WebSocket。

## 2. 审计结论

以下问题均在实施前从源码复核,不是仅凭文件名或 README 推断。

| 级别 | 证据 | 问题 |
| --- | --- | --- |
| P0 | `guard-find-miniapp/src/utils/imSocket.ts`、uni-mp-weixin 运行时 | 无 callback 的 `uni.connectSocket` 返回 Promise;旧实现把 Promise 误判为不支持,连接实际未建立 |
| P0 | `guard-find-app/app/im/chat.tsx` | 发送回包推进拉取游标,可能越过尚未读取的对方消息;追加无去重 |
| P0 | `guard-find-server/.../websocket/ImWebSocketHandler.java` | 推送只检查连接开启和握手时的 userId,令牌撤销后仍可能继续收到消息 |
| P1 | `guard-find-server/.../im/service/impl/ImServiceImpl.java` | 消息无界读取、读前整会话标已读、未读清零与并发发送竞态、事务提交前推送 |
| P1 | `guard-find-miniapp/src/pages/message/chat.vue` | start/poll 可并发,旧请求的 finally 会污染新请求;登录/隐藏生命周期处理不完整 |
| P1 | `guard-find-app/lib/storage/kv.ts`、`lib/calc/password.ts` | 持久化驱动接线待核验;缺少安全随机源时降级 `Math.random` |
| P1 | `guard-find-app/lib/capabilities/*.harmony.ts` | 鸿蒙选图、压缩、分享明确 unsupported,工作区无完整原生工程 |
| 待测 | `guard-find-backend/.../UserServiceImpl.java`、`PropertyServiceImpl.java`、`FileMapper.java` | 用户角色装配疑似 N+1、房源索引、文件聚合成本需真实执行计划确认 |

静态契约扫描(`npm run check:contract:ci`)结果:226 条后台路由、178 个调用点,174 匹配、4 个已登记下线;方法不匹配、未知路径、未知权限、缺失菜单页面均为 0。仍有 6 个无法解析的 CRUD 模板;52 条未被扫描到前端使用的后台路由不等于 52 个确定缺漏。

## 3. 已实施并验证的批次

### 3.1 小程序实时连接

- 显式传 callback 获取真实 `SocketTask`,不再把 Promise 当平台不支持。
- 每次连接使用代次隔离,迟到的 open/message/close/fail 一律忽略。
- 增加建连超时、pong 超时、send fail、1–30 秒指数退避;最后订阅者退订后 2 秒关闭。
- 新增连接恢复通知,页面可在重连成功后立即补拉。
- 服务端端点仍为 `/miniapp/ws/im?token=...`,本次未改协议。

### 3.2 小程序聊天与消息列表

- 聊天同步提取到 `chatSync.ts`:按可见页面和 token 代次单飞补拉,100 条分页续拉,读取游标单调前进,旧响应不再生效。
- 发送回包只合并展示,不推进读取游标;发送期间的新输入不会被旧请求清空。
- 退出、换号、隐藏会停止轮询与订阅并清空旧账号消息;显示时重新鉴权再启动。
- 列表页在可见期间轮询兜底,连接恢复立即补拉。新增只读接口 `GET /miniapp/im/support/unread`,按当前用户和 `ADMIN` 方向统计,不标已读、不写库;客户端在公寓管家入口显示未读角标。

### 3.3 服务端 WebSocket 安全与消息事务

- 握手保存原始 token 和 userId;每次推送、心跳都重新校验 token 与账号状态。
- 失效连接关闭并移除,不影响同用户其他设备;关闭失败也保证移除。
- 推送内容中的会话、发送者、消息 ID 统一序列化为字符串,避免 Java Long 超过 JS 安全整数。
- 同一连接加锁串行发送,发送异常关闭连接。
- IM 推送改为事务提交后执行,回滚不再向对方推送不存在的消息;发送前锁会话行,串行更新摘要与未读。
- 普通消息与客服消息查询统一 `LIMIT 100`,只把本次返回的对方/客服未读消息标已读并按数量递减,不再一次性清零未返回历史。
- 后台客服回复补事务边界:消息落库提交后才通过同一 C 端连接推送 `admin_message`,回滚不推送;读取客服记录按返回 ID 标记 USER 消息已读。
- server 后台客服读取与 backend 对齐:首屏最近 100 条、增量最多 100 条、只标记本次返回消息,禁用账号不允许后台发送。

### 3.4 App 聊天同步与连接生命周期

- 聊天同步拆到 `lib/im/chat-sync.ts`:按页面代次单飞补拉,消息按十进制字符串 ID 去重排序,100 条整页续拉且游标单调。
- 发送回包只并入列表,不推进读取游标;发送失败仅在输入框仍为空时恢复原文,不覆盖新草稿。
- 聊天页以“可见 + AppState 前台”双重门控读取、轮询与订阅;失焦、切后台、退出或换号立即停读并清空旧账号消息。
- `im-realtime` 在 token 变化时关闭旧连接、用新 token 重建;后台断开、回前台恢复;重连定时器可取消,迟到 socket 帧不派发。
- 会话列表增加通配推送刷新、可见期间 15 秒兜底轮询、前后台判断和退出清空;客服回复也会触发列表刷新。

### 3.5 验证证据

| 项 | 命令 | 结果 |
| --- | --- | --- |
| 小程序类型检查 | `pnpm type-check` | 通过 |
| 小程序测试 | `pnpm test:run` | 11 文件 90 测试通过 |
| 小程序 ESLint | 对相关 8 个文件执行 `pnpm exec eslint` | 通过 |
| 微信构建 | `pnpm build:mp` | Build complete |
| 小程序浏览器联调 | H5 + 浏览器内模拟 REST 响应 | 105 条跨页补齐、发送期间对方消息未漏、发送未推进读取游标、退出后停止请求并清空消息、客服页可显示客服消息 |
| App 类型检查 | `npx tsc --noEmit` | 通过 |
| App 测试 | `npx jest --ci --watchAll=false --runInBand` | 18 套件 114 测试通过(新增聊天同步 8 项、连接生命周期 6 项、存储驱动 3 项、无安全随机源 1 项、上传契约 1 项) |
| App ESLint | 对本批改动文件执行 `npx eslint --max-warnings=0` | 通过 |
| 后端 IM 回归 | `mvn ... -Dtest=SupportUnreadRegressionTest,ImRegressionTest` | 7 + 11 项通过,0 失败/错误/跳过 |
| 服务端 WebSocket 回归 | `mvn ... -Dtest=ImWebSocketSecurityTest,ImAppBoundaryTest,AdminImServiceImplTest` | 57 + 2 + 5 项通过,0 失败/错误/跳过 |
| 服务端事务/分页回归 | `mvn ... -Dtest=ImTransactionPagingTest,...` | 新增 4 项,IM 合计 68 项通过 |
| backend 客服事务回归 | `mvn ... -Dtest=ImRegressionTest` | 15 项通过,含提交前不推送/提交后推送/回滚不推送 |
| server 客服事务/分页回归 | `mvn ... -Dtest=AdminImServiceImplTest` | 9 项通过,含最近100条/增量100条/只标返回ID/提交后推送 |
| C 端上传契约 | `mvn ... -Dtest=AppFileControllerTest` | 5 项通过,含未登录 401/非图片/超限/非法目录 |
| BPMN 引擎隔离兼容 | `mvn ... -Dtest=FlowableEngineCompatibilityTest,FlowableSpringBootCompatibilityTest` | 4 项通过,含 H2 建表/部署/完成/计时器/引擎重启与 Boot 自动配置 |
| BPMN 生产接入 | `mvn ... -Dtest=OaEngineIntegrationTest` + `OaEngineLiquibaseMySqlTest` | H2 7 项 + MySQL 2 项通过,含租户隔离/事务回滚/重启/定时器恢复/关闭自动建表 |
| OA 表迁移(流程/表单/申请/审计/抄送/附件/outbox/幂等/员工组织) | `mvn ... -Dtest=OaEngineLiquibaseMySqlTest` | MySQL 8.4 隔离库 14 个 changeset 通过,十三表字段/索引与租户/全局唯一约束生效 |
| OA 候选解析与执行拦截 | `mvn ... -Dtest=OaAssigneeResolverTest,OaProcessLauncherTest` | 6+5 项通过,含五种审批人来源/离职与跨租户拒绝/空候选不写引擎 |
| OA BPMN 发布安全校验 | `mvn ... -Dtest=OaBpmnValidatorTest,OaConditionRuleCompilerTest,OaBpmnValidationConfigurationTest` | 17+9+2 项通过,含 XXE/节点白名单/受控服务任务/受限条件/容量限制与白名单配置 |
| backend-system 全量回归 | `mvn -pl guard-find-backend-system -am -DskipTests=false test` | 90 项通过,0 失败/错误/跳过 |

浏览器联调使用页面内注入的模拟数据,不是真实 Java 服务;未验证真实 MySQL/Redis、微信真机、鸿蒙真机或多副本部署。App 本批仅完成静态检查与单元测试,未在 Android/iOS/鸿蒙真机或模拟器运行,也未连接真实 server。

工具批证据:server 便笺接口 `ToolNoteServiceImplTest` 7 项通过;App `/tools/notes` 通过类型检查、109 项既有测试与改动文件 ESLint,未连接真实服务,7.2 仍需浏览器/鸿蒙验收。物品收纳已补齐 `gf_tool_item`、`/app/tool/item` 分页 CRUD、数量非负 CAS 与归档;`ToolItemServiceImplTest` 9 项通过,工具类合计 16 项。App 新增 `/tools/items` 页面与入口,数量调整冲突时回刷;同样未连真实服务或真机,7.4 保持未完成。

上传契约证据:审计发现 App 原先请求的 `/file/upload` 在 server 无处理器(后台为 `/system/file/upload`,公共为 `/system/common/file`),真机上传必然 404。已新增 `/app/file/upload`:强制登录态用户 ID,目录白名单 `case|lead|avatar|report|post`,仅图片 MIME + `jpg/jpeg/png/webp/gif`,5 MB 上限,路径按 `app/{userId}/{folder}/yyyy/MM/dd` 隔离归属;App 全部上传调用统一改走该端点。`AppFileControllerTest` 5 项通过(成功、非图片、超限、非法目录/空文件、未登录 401),App 上传契约单测钉死路径/字段/超时,类型检查、18 套件 114 项测试与改动文件 ESLint 通过。未连接真实存储服务或真机。

BPMN 底座证据:引擎候选锁定 Flowable 7.2.0(BOM 基于 Boot 3.5.4 / Spring 6.2.10,与本项目 Boot 3.4.10 / Spring 6.2 同代;Jar 清单 Apache-2.0;8.0.0 面向 Boot 4 不可用)。以 test 范围依赖在 backend-system 增加隔离测试,4 项通过:H2 建表后部署 BPMN、排他网关完成并落历史;计时器 Job 转可执行并推进到用户任务;文件库引擎重启后任务仍在并可完成;仅装配 DataSource + 事务管理器时 Boot 自动配置能创建 ProcessEngine。同时发现 process starter 干净类路径缺少 `flowable-spring-configurator` 与 `flowable-app-engine-spring`,启动即 `NoClassDefFoundError`,已显式补齐并写回设计。backend-system 全量 90 项通过。未连接项目 MySQL/Liquibase,未验证多副本与长稳。

BPMN 生产接入证据:新增 `guard-find-backend-plugin-oa` 模块并由 api 依赖,迁移所有权归 Liquibase。Flowable 7.2.0 官方 MySQL 建表脚本 common/process/history 原样纳入 `db/changelog/mysql/oa`,接入 master changelog;应用配置 `flowable.database-schema-update=false`,仅启用 process 引擎,app/cmmn/dmn/idm/eventregistry 关闭;`OaFlowableConfiguration` 设历史级别 FULL、禁用 IDM/事件注册,并用 BFPP 给 `processEngine` 追加 `depends-on=liquibase`;`OaEngineTemplate` 所有入口强制 tenantId。`OaEngineIntegrationTest` 7 项通过:仅建 process 表且无 ACT_ID 表/无 AppEngine;租户 A/B 互不可见且跨租户完成被拒;跨租户领取被拒、重复领取幂等;外层事务回滚后无流程实例与任务;文件库重启且在 `schema-update=false` 下任务仍可读取并办结;重启后定时器 Job 仍在库中并可转可执行推进;存在 liquibase Bean 时 dependsOn 生效。`OaEngineLiquibaseMySqlTest` 在本地 MySQL 8.4 隔离库实测:3 个 changeset 成功,`common.schema.version`/`schema.version` 均为 7.2.0.2,关闭自动建表后引擎启动并完成部署/发起/办结,验证库已删除。异步执行器仍关闭,定时器为引擎级手工执行验证;未用项目开发库启动完整应用,多副本与长稳未验证。

OA 表迁移证据:已新增 `oa_process.sql`、`oa_form.sql`、`oa_application.sql`、`oa_approval_audit.sql`、`oa_cc_receipt.sql`、`oa_attachment.sql`、`oa_outbox.sql`、`oa_request_dedup.sql`、`oa_staff.sql` 九个 Liquibase changeset,共十三张表。`oa_process_model` 含租户内 model_key 唯一、草稿 XML 与 SHA-256、生效版本指针、绑定表单版本、乐观锁;`oa_process_release` 冻结发布 XML、引擎定义 ID 全局唯一、租户+模型+版本唯一、表单版本绑定、可发起范围,且不设 update/delete 列以保持不可变;`oa_form` 含租户内 form_key 唯一、当前版本指针与乐观锁;`oa_form_version` 含租户+表单+版本唯一、schema 快照与摘要、字段数与发布人,同样不可变;`oa_application` 含租户+业务类型+业务单号唯一、流程实例 ID 全局唯一、发布/表单版本绑定、申请人及部门快照、提交数据快照、状态与乐观锁;`oa_approval_audit` 不可变,以租户+request_id 唯一防止命令重试产生重复轨迹,记录实际操作人、原审批人、理由与决策明细;`oa_cc_receipt` 以租户+申请+接收人唯一,记录未读/已读与读取时间;`oa_attachment_link` 以租户+file_id+deleted 唯一,记录申请/业务归属、扫描状态与可下载标记,裸 fileId 不构成授权;`oa_outbox` 以 event_id 全局唯一,记录聚合、事件类型、重试时间与失败原因,按状态+重试时间及租户+聚合建索引;`oa_request_dedup` 以租户+操作人+命令+request_id 唯一,保存结果快照与过期时间供重试回放;`oa_position` 以租户内 code 唯一,含启用/停用;`oa_employee` 以租户内 user_id 与工号唯一,记录直属主管、ACTIVE/SUSPENDED/RESIGNED 与部门负责人标记;`oa_employee_position` 以租户+员工+岗位唯一。十三表均带 tenant_id 与查询索引,master changelog 和验证 changelog 已接入。MySQL 8.4 隔离库实测 14 个 changeset 全部成功,`OaEngineLiquibaseMySqlTest` 2 项通过:关闭自动建表后引擎可启动/办结;十三表字段与关键索引存在,重复 model_key/form_key/表单版本/业务单号/流程实例/request_id/抄送接收人/附件 file_id/幂等四元组/岗位 code/员工 user_id/工号/员工岗位及全局重复引擎定义 ID/outbox event_id 被拒,跨租户同名/同 fileId/同 request_id/同 user_id 按设计允许。该批次尚未编写实体/Mapper/API,也未实现 outbox 投递与幂等清理作业(表单 API 与版本服务已由 9.8 补齐)。

OA 员工组织与候选解析证据:部门沿用 `sys_user.dept_id`,新增 `oa_position`、`oa_employee`、`oa_employee_position`;`OaAssigneeResolver` 只接受结构化的五种来源(指定员工、岗位、部门负责人、直属主管、受控表单人员字段),`JdbcOaStaffDirectory` 每条 SQL 显式带 tenant_id,并以 `oa_employee.employment_status='ACTIVE'` 且 `sys_user.status=1` 判定在职,跨租户、离职、停用账号一律拒绝。发布校验拒绝不存在/停用/无在职员工的岗位与离职或跨租户的指定员工;部门负责人、直属主管、表单字段依赖具体发起人,发布时只校验结构,执行时为空即失败。`OaProcessLauncher` 在写引擎前解析全部候选变量,失败抛 `OaAssigneeUnavailableException`,不产生流程实例或任务。`OaAssigneeResolverTest` 6 项、`OaProcessLauncherTest` 5 项通过,模块合计 20 项(MySQL 2 项无环境变量时跳过)。边界:员工/岗位尚无 CRUD API 与页面,也没有实体/Mapper(当前为显式 SQL 只读目录);候选解析只覆盖发起时已知变量,后续节点需在任务创建监听复用解析器;员工离职不会自动转交存量待办(10.3);`oa_delegate_rule` 未建;未用项目开发库启动完整应用。

OA 员工/岗位管理证据(9.4 续):新增 `oa_staff_manage.sql`,为 `oa_employee`/`oa_position` 增加乐观锁 `version`,并新增员工管理 `/oa/employee`、岗位管理 `/oa/position` 菜单(9630/9640)与 list/create/update 权限(9631-9633、9641-9643),授权仍从已具备 `system:user:list` 的角色与租户套餐复制;首次沿用的 9610/9611 与 10.2 审批菜单冲突,已改为 9630 段后随 Liquibase 在本机 dev 库执行成功。后端新增 `/system/oa/staff`:员工分页/详情/创建/更新、员工选项与可建卡用户选项;岗位分页/详情/创建/更新与启用岗位选项,选项接口按 `SaMode.OR` 同时服务员工管理、岗位管理与发起页。创建员工只接受当前租户启用且尚未建卡的后台用户,工号租户内唯一;直属主管必须是在职员工且逐级回溯拒绝环路;岗位必须属于本租户且启用;ACTIVE 清空离职日期、RESIGNED 未填时默认当天;更新按 `version` 条件更新,冲突返回「员工档案已被其他人修改,请刷新后重试」/「岗位已被其他人修改,请刷新后重试」。前端新增 `queryOaEmployees` 等 service、`/oa/employee` 与 `/oa/position` 页面、`OaEmployeeSelect`/`OaPositionSelect` 选择器;表单渲染器的 person 字段和发起页的 EMPLOYEE/POSITION 审批人绑定不再手输员工 ID/岗位编码,统一从本租户选项选择,并补齐 `menu.oa.employee`、`menu.oa.position` i18n。测试证据:新增 `OaStaffServiceTest` 6 项(建卡唯一性/跨租户用户、主管环路与离职主管、岗位替换与过期版本/租户隔离、选项过滤、岗位 CRUD 与停用、非法岗位);OA 模块 105 项运行,0 失败/错误,2 项 MySQL Liquibase 用例因无隔离库环境变量跳过;后端全量 196 项运行,0 失败/错误,2 项跳过,`GuardFindBackendApplicationTests` 在 dev profile 下连本机 MySQL 启动成功,`oa_staff_manage.sql` 已随 Liquibase 实际落库(含 9630/9640 菜单)。前端新增 `oaStaffManage` 套件 6 项,`oaFormFill`/`oaLaunchFill` 改为选择器交互并补选项 mock;前端全量 Jest 38 套件 227 项通过,`tsc --noEmit`、改动文件 Biome 与契约检查(verbMismatch 0、noPath 0、unknownPermissions 0、missingPages 0)通过。边界:本批未做浏览器实机验收;岗位只支持创建/改名/排序/停用,不提供物理删除;person 字段只列已建 OA 档案的在职员工,未建卡账号需先在员工管理建档;员工离职仍未自动转交存量待办(10.3);不得据此宣告 OA 可交付。

OA BPMN 发布安全校验证据:`OaBpmnValidator` 以命名空间感知 DOM 解析,关闭 DOCTYPE、外部通用/参数实体与外部 DTD,启用 secure processing,`ACCESS_EXTERNAL_DTD`/`SCHEMA` 置空并由实体解析器直接抛错;预检 `<!DOCTYPE`/`<!ENTITY`,限制 512KB/200 个流程节点/2000 个元素/50 层嵌套,错误最多保留 40 条,并对原始 XML 计算 SHA-256。节点白名单为 definitions/process/startEvent/endEvent/userTask/serviceTask/exclusiveGateway/parallelGateway/boundaryEvent/sequenceFlow/conditionExpression/timerEventDefinition/timeDuration/timeDate/timeCycle/incoming/outgoing,另放行 bpmn-js 所需 BPMNDiagram/BPMNPlane/BPMNShape/BPMNEdge/BPMNLabel、dc:Bounds、di:waypoint;scriptTask、callActivity、subProcess、extensionElements、camunda/activiti 属性与任意 Flowable 子元素均拒绝。结构上要求单流程且 isExecutable=true、id 唯一、开始/结束事件存在、连线引用有效、开始事件不可作为目标、边界定时器只能挂用户任务、条件只能位于排他网关输出连线且默认连线有效、默认连线不能配置条件、用户任务至少配置一种审批人来源、排他网关多出口除默认连线外必须有条件。服务任务只接受白名单内的 `flowable:class`,`flowable:expression`/`delegateExpression` 一律拒绝;白名单通过 `oa.bpmn.allowed-service-handlers` 逗号配置,默认空即全部拒绝。`OaConditionRuleCompiler` 把结构化 AND/OR 条件编译为受限 EL(EQ/NE/GT/GE/LT/LE/IN/NOT_IN,字段仅标识符/点路径,值仅数字/布尔/字符串或集合),再由 `OaConditionExpressionValidator` 词法/语法复核,只允许变量、字面量、比较、括号与 &&/||/!,拒绝方法调用、构造函数、`T(...)`、Bean 引用、三元、算术与 EL 关键字。`OaBpmnValidatorTest` 17 项、`OaConditionRuleCompilerTest` 9 项、`OaBpmnValidationConfigurationTest` 2 项通过,模块合计 48 项(46 项执行通过,MySQL 2 项无环境变量跳过)。边界:尚未接入发布/部署事务与版本落库(9.7),当前只校验 XML 不实际部署;服务任务白名单不验证类是否存在或实现 JavaDelegate;条件只做安全/语法校验,未在真实引擎执行语义回归;DI 只做放行,不校验图形引用完整性;未用项目开发库启动完整应用。

OA 流程模型发布/停用证据:后端新增 `/system/oa/process/model` 控制器、服务与显式租户 SQL 仓储,提供分页/详情/创建/保存(乐观锁)/XML 校验/发布/停用/版本列表;列表 SQL 不读取 `bpmn_xml`。发布在同一事务内完成服务端 XML 复核、流程 Key 稳定性与跨模型占用检查、表单版本租户校验、Flowable 部署、不可变发布记录写入、旧 ACTIVE 版本停止与模型指针更新,任一步失败回滚;再次发布生成新版本,运行中的旧实例仍绑定原流程定义,停用只停止 ACTIVE 发布记录且之后可再次发布恢复。新增 `oa_menu.sql`(流程管理/流程模型/隐藏设计器与 list/create/update/publish/disable 权限),仅向已具备 `system:user:list` 的角色和租户套餐复制授权。`OaProcessModelServiceTest` 5 项通过:创建草稿并拒绝过期版本、发布绑定表单并激活版本、v1 实例在发布 v2 后仍按 v1 完成且新实例使用 v2、非法 XML 不产生部署或发布记录、停用后可重新发布且他模型占用同一流程 Key 被拒;OA 模块合计 54 项,52 项执行通过,MySQL 2 项无环境变量跳过。前端新增 OA 模型 service、`/oa/process` 列表页和设计页按 `id` 绑定:列表支持搜索/筛选、新建(按模型标识生成独立流程 Key)、设计、发布(表单版本/发起范围/说明并展示服务端拒绝原因)、停用与版本历史;设计页支持加载模型、保存草稿、服务端校验、发布前自动保存、停用确认及状态/版本展示。OA 相关 7 个前端套件 70 项通过,`tsc --noEmit` 与 `build:zy` 通过,两个路由均进入产物。边界:自动 MySQL Liquibase 用例仍因未设置环境变量跳过,但 `oa_menu.sql` 已随完整迁移在 3.6 的隔离 MySQL 8.4.11 落库;真实登录态浏览器链路已由 3.6 走通;发布当时未调用 `OaAssigneeResolver` 校验员工/岗位存在性与在职状态(已由 9.8 补齐静态来源校验);发起入口尚未按发布状态与可发起范围拦截(已由 10.1 补齐);列表页暂无名称/分类/说明编辑入口。

OA 流程设计器前端证据:新增 `/oa/process/design` 页面与 `OaBpmnCanvas`/`bpmnLoader`/`flowableModdle`/`OaBpmnProperties`/`OaConditionBuilder`/`OaBpmnPreview` 组件,`src/utils/oaBpmn.ts` 提供与服务端同构的预检、默认模型和受限条件编译。bpmn-js 18.31.1 的 `Modeler`/`NavigatedViewer` 通过动态 `import()` 懒加载;`FLOWABLE_MODDLE` 只声明用户任务 assignee/candidateUsers/candidateGroups 与服务任务 class。页面支持新建、导入 `.bpmn/.xml`、导出、校验、只读预览、撤销/重做/适应画布;属性面板可编辑名称、用户任务审批范围、服务任务处理器、网关默认连线、边界定时器与排他网关输出条件,条件构建器把 AND/OR 结构化条件编译为受限 EL。前端预检拒绝 DOCTYPE/ENTITY、超 512KB、超 200 流程节点/2000 元素、非白名单节点/属性、非法审批范围/定时器/条件,并校验开始/结束事件、连线引用、边界事件挂载与默认连线;服务端仍是最终裁决方。真实 bpmn-js 运行时以临时 Node+jsdom 脚本冒烟(未入库)验证名称、候选审批人、条件、默认连线与定时器序列化,并发现 bpmn-js 对“默认连线”和“连线条件”互斥清空,因此客户端与服务端同时新增“默认连线不能配置条件”,属性面板对默认连线隐藏条件编辑,`OaBpmnValidatorTest` 增加 `rejectsConditionOnDefaultFlow` 并 17 项通过。前端 5 个套件 45 项测试通过,`tsc --noEmit` 通过,`build:zy` 12.9s 成功,页面 chunk(约 48KB)异步引用 bpmn-js chunk(约 322KB)。边界:无保存/发布 API,发布按钮禁用;`sys_menu` 无 OA 菜单,`RouteAccess` 可能拦截路由,需 9.7 补菜单与权限;未在真实登录态浏览器连接后端走通;画布 Jest 用例使用假构造器验证接线,真实 bpmn-js 未纳入 CI;设计期预检不替代服务端校验。上述 API/菜单/浏览器边界已由 9.7 与 3.6 闭环。

### 3.6 OA 流程模型浏览器实机验收

2026-11-06 在隔离 MySQL 8.4.11 库 `gf_oa_e2e_1106`、Redis DB 15 上启动完整后端(`:8000`, dev profile)与前端开发服务器(`:9002`),用内置浏览器以 admin 登录走通真实链路;隔离库内关闭登录验证码,验收后已删除库、清空 Redis DB 15 并释放端口。实测链路:登录进入工作台;`/oa/process` 通过真实 API 渲染空列表;新建模型 id=1(key `leave_e2e_1106`,名称「请假流程 E2E」);设计页加载 bpmn-js 画布并显示发起/审批/结束三节点;前端预检与服务端校验均通过;保存草稿生成 v1;发布返回「版本 v1(引擎版本 1)」,`oa_process_release` v1=ACTIVE,`ACT_RE_PROCDEF` 生成 tenant 0 的 `leave_e2e_1106:1:...` 定义;列表与版本历史显示 v1 生效中;停用后模型 DISABLED、v1 转 STOPPED、草稿 v3;停用后重新发布生成 v2(引擎版本 2),模型 `current_release_version=2`、乐观锁 `version=4`,`oa_process_release` v1=STOPPED/v2=ACTIVE,引擎侧 v1、v2 两个流程定义各自保留独立 deployment。实机同时确认 `oa_menu.sql` 已随完整 Liquibase 迁移落在真实 MySQL,OA 菜单与权限可正常访问;补 `menu.oa*` 与页签通用 i18n key 后,`/oa/process` 与 `/oa/process/design` 控制台不再出现缺键告警,设计器页签正确显示「流程设计器」。

边界:内置浏览器页面处于隐藏态,无法截图,证据来自 DOM 快照、真实网络响应与数据库查询;弹窗关闭动画因页面隐藏而冻结在 DOM,非产品缺陷。未覆盖表单填写、发起、审批、抄送与附件;发布强制绑定表单版本已由后续批次补齐,发布时的静态员工/岗位校验也已由 9.8 补齐(表单人员与直属主管仍在发起时解析),发起入口未按发布状态/发起范围拦截(已由 10.1 补齐),审批中心已由 10.2 补齐,异步执行器仍关闭。该批次只证明「流程模型设计-校验-发布-停用-重发」链路可用,不代表 OA 业务可交付。

### 3.7 OA 发起/审批/附件/员工管理浏览器实机验收

2026-11-08 在隔离 MySQL 库 `gf_oa_e2e_1108`、Redis DB 15 上启动完整后端(`:8000`, dev profile)与前端开发服务器(`:9002`),用内置浏览器以 admin 与 test 两个真实账号走通员工/岗位管理、表单与流程发布、发起填写、我的申请、审批中心与附件下载;隔离库内关闭登录验证码,验收后已删除库、清空 Redis DB 15 并释放 8000/9002 端口。

- **员工/岗位与选择器**:`/oa/employee`、`/oa/position` 页面可渲染真实分页数据;admin 账号原先无 OA 档案,发起时返回「发起人不是本租户在职员工: 1」,通过已鉴权接口补建 E001(user 1,ACTIVE,部门负责人)后恢复。表单人员字段与审批人「指定员工」均从本租户选项下拉选择,实测列出 E001/E002/E003,不再手输 ID。
- **表单与流程发布(9.8)**:真实 API 创建请假表单并发布 v1;流程模型绑定「请假申请单 v1」与发起范围后发布 v1(发布对象 id=1),发起列表随即出现「请假审批」,表单显示 `请假申请单 v1 · 发布 v1`。
- **发起填写与附件(10.1/10.5)**:以 admin 发起 3 个申请,表单含日期、人员、附件与文本字段;人员字段选中测试员(E002),附件经真实 `/system/file/upload` 上传后提交。申请 1「验收请假申请」`oa_application.status=RUNNING`,`form_data` 中人员与 18 位附件 fileId 均按字符串落库,`oa_attachment_link` 写入归属行;我的申请列表与详情渲染表单快照,附件下载 `GET /dev-api/system/oa/attachment/<fileId>` 返回 200 `text/plain`。
- **审批通过(10.2)**:切换到 test 账号,待办审批出现申请 1;审批详情展示申请快照与提交轨迹;填写意见「实机验收:同意请假」后通过,页面提示「已通过」且待办清空。数据库证据:`oa_application` id=1 置 `APPROVED`、`finished_time=2026-11-08 20:53:35`;`oa_approval_audit` 新增 APPROVE 行(operator_id=547889293968801822,reason 与界面一致,detail_json `processEnded=true`);`act_ru_task` 为 0 行,对应 `act_hi_procinst` 已结束。已办页签展示「审批通过 / 实机验收:同意请假 / 已通过」,申请人详情页的审批轨迹同步出现审批通过节点。
- **撤回(10.2)**:申请 2 由 admin 发起、审批人 Charles 后,由申请人本人在 RUNNING 状态撤回;界面显示「已撤回」,库中置 `WITHDRAWN` 并写入 WITHDRAW 审计,运行任务被删除、实例结束。
- **驳回(10.2)**:申请 3 以「指定员工=测试员」发起,test 在待办中填写必填原因「实机验收:驳回链路(测试)」后驳回;界面提示「已驳回」且待办清空,库中置 `REJECTED`、写入 REJECT 审计(含 task_id/原处理人),运行任务为 0、实例结束。

验收中发现并修复的问题:(1) `OaAssigneeContext` 把默认租户 0 当作非法值,导致本部署所有发起/审批返回「租户不能为空」,已改为仅拒绝负数租户;(2) 前端 `oaAttachmentFileId` 用 `Number.isSafeInteger` 校验,18 位雪花 ID 被误判为未上传,已改为全程字符串透传,并同步测试与申请详情附件类型;(3) 审批详情与申请详情的 `fileId` 增加防御性字符串归一,避免反序列化回数字;(4) 缺失 `menu.oa.tasks`/`menu.oa.tasks.detail` 导致页签缺键告警,已补齐中英文案,实测控制台不再出现 Missing message。上述改动尚未提交。

边界:退回上一节点只由单测覆盖,本次未在浏览器验证(当前发布模型仅一个审批节点,退回语义要求上一人工节点);审批详情/申请详情的人员字段仍显示用户 ID 而非姓名(UX 待补);隐藏页面自动化下附件删除与「重置」的原生点击未触发 React 处理器,是否为产品缺陷未定论;证据来自 DOM 快照、真实网络响应与数据库查询,无截图。员工离职转交、抄送通知、异步 outbox 投递仍未实现;不得据此宣告 OA 可交付。

OA 发布强制绑定表单版本证据:后端在 XML 复核前拒绝缺失 `formVersionId` 的发布请求,返回「发布必须绑定表单版本」;表单版本仍按租户查询,不存在或跨租户返回「绑定的表单版本不存在或不属于当前租户」;任一拒绝都不产生 Flowable 部署或发布记录,模型保持原状态。前端两处发布弹窗把「表单版本 ID」改为必填并前置校验,校验失败不发送请求;`OaProcessPublishParams` 类型同步为必填。`OaProcessModelServiceTest` 6 项通过(新增缺失表单版本被拒且无部署、无发布记录、模型仍为草稿),相关 3 个前端套件 29 项与 `tsc --noEmit` 通过。边界:表单设计器与表单版本列表 API 仍未实现,发布时仍需手工填写已存在的版本 ID;本批次未重新启动完整前后端做浏览器实机回归。上述表单设计与版本列表边界已由 9.8 闭环,发布已改为下拉选择版本。

OA 表单设计与表单版本证据(9.8):后端新增 `oa_form_draft.sql`,为 `oa_form` 增加可编辑草稿 `draft_schema_json` 与 SHA-256 `draft_schema_digest`;新增 `oa_form_menu.sql`(表单管理 `/oa/form`、隐藏设计器 `/oa/form/design` 与 list/create/update/publish/disable 权限,仅向已具备 `system:user:list` 的角色和租户套餐复制授权)。`OaFormSchemaValidator` 校验 8 类字段(text/textarea/number/date/radio/checkbox/attachment/person),限制 1MB/100 字段/50 选项,拒绝重复 id/key、单选多选缺选项与非法 JSON,草稿允许空字段、发布要求至少一个字段,并对 schema 计算 SHA-256。`/system/oa/form` 提供分页/详情/创建/保存草稿(乐观锁)/发布/停用/版本列表/版本下拉选项;创建与更新只写草稿,发布在同一事务内复核 schema、写入不可变 `oa_form_version` 快照并推进当前版本指针,非法发布不产生版本或指针变更;版本列表不返回 schema,版本选项只返回 PUBLISHED 表单且上限 500 条,所有 SQL 显式带 tenant_id。流程发布新增静态审批人来源校验:`employee:101`、`position:FINANCE`、`formField:approvers`、`deptLeader`、`supervisor` 由 `OaAssigneeSpecParser` 解析,发布时校验指定员工/岗位/部门负责人是否存在且在岗,`supervisor` 与 `formField` 只校验结构、留待发起时解析,`${var}` 与普通值仍视为运行时变量跳过;校验失败在 Flowable 部署前返回拒绝。前端新增 `/oa/form` 列表(搜索/筛选、新建、发布、停用、版本历史)与 `/oa/form/design` 设计器(8 类字段面板、HTML5 原生拖拽排序、属性面板、实时预览、保存草稿/发布/停用/版本历史、未保存离开确认);流程列表页与设计页的发布弹窗改为从 `version-options` 下拉选择表单版本,不再手工输入版本 ID。测试证据:本轮新增 `OaFormSchemaValidatorTest` 7 项、`OaFormServiceTest` 6 项、`OaAssigneeSpecParserTest` 4 项,`OaProcessModelServiceTest` 增至 7 项(含发布前拒绝不可用静态审批人且不部署);OA 模块全量 73 项运行,0 失败/错误,2 项 MySQL Liquibase 用例因未设置环境变量跳过。前端 `oaForm.test.ts`、`oaFormList`、`oaFormDesign`、`oaProcessList`、`oaProcessDesign` 5 个套件 29 项通过,前端全量 Jest 27 套件 185 项通过,`tsc --noEmit` 通过,改动文件 lint 干净(仅保留一处既有的 `${...}` 字面量告警)。边界:表单人员与直属主管只在发起时解析,尚无表单填写/发起入口(10.1);员工/岗位仍无 CRUD API 与页面;本批未启动完整前后端做浏览器实机验收,前端证据仅为 Jest/类型/静态检查;自动 MySQL Liquibase 用例仍因未设置环境变量跳过;不得据此宣告 OA 可交付。

OA 表单渲染与填写页证据(9.8 续):后端新增 `GET /system/oa/form/version/{versionId}`(权限 `system:oa:form:list`),按租户关联未删除表单读取不可变版本快照,返回 formKey/名称/表单状态/版本号/schema/摘要/字段数/发布时间,不存在或跨租户统一返回「表单版本不存在」;已停用表单的历史版本仍可读取,发起资格拦截留待 10.1。`OaFormSchemaValidator` 增加字段策略:`readonly` 与 `hidden` 必须为布尔值,必填不能同时设为只读或隐藏,发布时若全部字段隐藏则拒绝并返回「表单至少需要一个可见字段才能发布」。前端新增通用渲染器 `OaFormRenderer`,按 schema 渲染 8 类控件:隐藏字段不渲染也不参与校验,只读字段禁用编辑并标注「（只读）」,必填字段带无障碍必填标记,校验错误以 `role="alert"` 呈现;附件当前只做本地选择、人员字段暂按员工 ID 输入,上传与组织人员选择器分别留待 10.5/10.1。新增 `/oa/form/fill?versionId=` 填写页:从 `version-options` 下拉切换版本并写回 URL,加载版本详情后渲染字段,支持「校验填写」「重置」与当前值摘要,页面明确提示提交/发起尚未接入且不写入申请数据,不提供提交按钮;表单列表与设计器的版本历史新增「填写」入口;新增隐藏菜单 `oa_form_fill_menu.sql`(菜单 ID 9580,组件 `oa/form/fill/index`,权限复制自表单管理)。测试证据:`OaFormSchemaValidatorTest` 增至 10 项、`OaFormServiceTest` 增至 7 项(新增版本详情读取与租户隔离),OA 模块全量 77 项运行,0 失败/错误,2 项 MySQL Liquibase 用例因未设置环境变量跳过;前端新增 `oaFormFill` 4 项,设计器补 2 项只读/隐藏策略,列表补「填写」跳转断言,前端全量 Jest 28 套件 194 项通过,`tsc --noEmit` 通过,改动文件 lint 干净。边界:本批只做渲染与本地校验,提交/发起、申请落库、审批中心、附件上传与人员选择器均未实现(10.1/10.5,其中提交/发起与申请落库已由 10.1 补齐、审批中心已由 10.2 补齐);未启动完整前后端做浏览器实机验收,`oa_form_fill_menu.sql` 未在真实 MySQL 重跑;前端证据仅为 Jest/类型/静态检查;不得据此宣告 OA 可交付。

OA 发起与我的申请证据(10.1):后端新增结构化可发起范围 `OaLaunchScope`(`{"users":[],"depts":[],"roles":[]}`,并集语义,空对象表示不限),发布时解析并规范化后落库,非法范围在 Flowable 部署前返回「可发起范围无效」,历史非法值按不可发起处理;`OaLaunchScopeTest` 5 项。新增 `/system/oa/launch` 分页/详情/提交:可发起列表只读取 `oa_process_release.status='ACTIVE'`、模型 `PUBLISHED` 且为当前生效版本、表单 `PUBLISHED` 且绑定不可变版本的发布;详情按租户与范围校验后返回表单 schema 与 BPMN 审批人变量建议(同名人员字段建议 `FORM_FIELD`,`supervisor`/`deptLeader` 别名自动识别)。提交在同一事务内完成不可变 schema 的服务端填写校验(隐藏/只读不参与,必填、数字、单选/多选、人员与日期规范化,未知字段丢弃)、可发起范围复核、BPMN 与审批人绑定复核、Flowable 发起、申请与 SUBMIT 轨迹落库;审批人支持表单人员/直属主管/部门负责人自动绑定与指定员工/岗位/表单人员兜底选择,静态审批人、`candidateGroups` 变量与无法解析的变量一律拒绝,`assignee` 变量必须恰好解析出一名员工;`oa_request_dedup` 以租户+操作人+命令+request_id 唯一,PROCESSING 行随事务回滚,SUCCEEDED 结果 JSON 直接回放,唯一键竞争后回读,避免重复发起。申请业务单号为 `OA-<releaseId>-<yyyyMMddHHmmss>-<8位随机>`,初始状态 `RUNNING`,同时写入提交轨迹;`/system/oa/application` 只按当前申请人分页与读取详情,不接受他人申请 ID 探测。新增 `oa_launch_menu.sql`:发起流程 `/oa/launch`(列表、隐藏填写页与 list/submit 权限)和我的申请 `/oa/application`(列表、隐藏详情页与 list 权限),授权仍从具备 `system:user:list` 的角色与租户套餐复制。测试证据:`OaLaunchServiceTest` 5 项(范围过滤、表单字段自动绑定与幂等回放、非法表单回滚、多人 assignee 拒绝、静态审批人拒绝)、`OaLaunchScopeTest` 5 项,`OaBpmnValidatorTest` 增至 19 项(新增审批人变量分类),`OaProcessModelServiceTest` 增至 8 项(新增非法范围不部署);OA 模块全量 89 项运行,0 失败/错误,2 项 MySQL Liquibase 用例因未设置环境变量跳过。前端新增 `/oa/launch` 发起列表、`/oa/launch/fill` 填写页(按不可变 schema 渲染、标题输入、逐变量审批人自动建议与兜底选择、会话内稳定 requestId、提交成功展示业务单号/实例并跳转申请)与 `/oa/application` 列表、`/oa/application/detail` 详情(申请人/部门/模型/表单/版本/单号/实例/时间、表单快照与审批轨迹),流程列表与设计页的发布弹窗把「dept:10,dept:11」文本输入替换为结构化范围编辑器;新增 6 个前端套件 17 项,前端全量 Jest 34 套件 211 项通过,`tsc --noEmit`、改动文件 Biome 与契约检查(verbMismatch 0、noPath 0、unknownPermissions 0、missingPages 0)通过。边界:本批未启动完整前后端做浏览器实机验收,`oa_launch_menu.sql` 未在真实 MySQL 重跑;员工/岗位仍无 CRUD 与选择器,人员字段仍按员工 ID 输入,附件上传(10.5)与抄送未实现,审批中心已由 10.2 补齐;审批人解析只在发起时执行,后续节点仍需任务创建监听;不得据此宣告 OA 可交付。

OA 审批中心证据(10.2):后端新增 `OaApprovalEngine` 与 `/system/oa/approval`:待办按租户与 `taskCandidateOrAssigned` 只取当前处理人的任务(引擎侧扫描上限 200 条,关键字在内存过滤后分页),已办按 `oa_approval_audit` 的 APPROVE/REJECT/RETURN 且 `operator_id` 为当前用户分页,任务详情对进行中任务返回申请快照,对已处理任务只允许本人按 `(tenant, task_id, operator_id)` 回看审计对应申请,跨租户、非候选人、无审计记录统一拒绝。动作语义:`approve` 先校验任务身份与申请 RUNNING,未签收任务自动 claim 后 complete,流程结束置 APPROVED,否则保持 RUNNING;`reject` 必须填写原因,删除流程实例并置 REJECTED;`return` 必须填写原因,仅允许单一活动任务退回最近一个已完成人工任务,并恢复该历史任务的 assignee 或候选身份,含并行/包容网关直接拒绝「并行流程暂不支持退回」;`withdraw` 使用 `SELECT ... FOR UPDATE` 锁定申请,仅申请人本人且 RUNNING、且审计中不存在任何 APPROVE/REJECT/RETURN 时可撤回,删除实例并置 WITHDRAWN。所有动作在 `oa_request_dedup` 上以租户+操作人+命令+request_id 幂等,PROCESSING 随事务回滚、SUCCEEDED 结果 JSON 回放、唯一键竞争后回读;引擎动作、`oa_application` 状态/版本递增、`finished_time` 与不可变 `oa_approval_audit`(含 task_id/原处理人/原因/detail_json)在同一事务提交,引擎异常统一映射为「任务状态已变化,请刷新待办后重试」;审批与撤回通过 `version = version + 1` 与 `status='RUNNING'` 条件更新串行化。新增 `oa_approval_menu.sql`:待办审批 `/oa/tasks`、隐藏详情 `/oa/tasks/detail`、`system:oa:approval:list`、`system:oa:approval:action` 与我的申请下的 `system:oa:application:withdraw`,授权仍从具备 `system:user:list` 的角色与租户套餐复制。测试证据:新增 `OaApprovalServiceTest` 6 项(待办/已办身份隔离与已办只读回看、串行通过并幂等回放、驳回原因必填且流程结束、仅申请人本人且无审批动作时撤回、退回上一人工节点并恢复处理人、并行网关拒绝退回);OA 模块全量 95 项运行,0 失败/错误,2 项 MySQL Liquibase 用例因未设置环境变量跳过;后端全量 186 项运行,0 失败/错误,2 项跳过。前端新增 `/oa/tasks` 待办/已办页签与 `/oa/tasks/detail` 详情页:待办展示申请/流程表单/申请人/当前节点/到达时间并进入处理,已办展示动作标签、原因、处理时间与申请状态;详情页复用申请快照与轨迹组件,待办可提交通过/驳回/退回上一节点(驳回与退回原因必填),成功后跳回审批中心,已处理任务显示只读提示;我的申请详情在 RUNNING 时提供撤回弹窗(原因可选、会话内稳定 requestId、成功后重载)。新增 `oaTaskList`、`oaTaskDetail`、`oaApplicationWithdraw` 3 个套件 7 项,前端全量 Jest 37 套件 218 项通过,`tsc --noEmit`、改动文件 Biome 与契约检查(verbMismatch 0、noPath 0、unknownPermissions 0、missingPages 0)通过。边界:本批未启动完整前后端做浏览器实机验收,`oa_approval_menu.sql` 与审批接口未在真实 MySQL 重跑,前端证据仅为 Jest/类型/静态检查;待办查询受引擎 200 条扫描上限约束,大数据量下需改为引擎分页或索引化查询;退回只支持最近一个已完成人工节点,不支持跨节点选择、并行/包容分支与加签转办;已办列表只含审批类动作,不含 SUBMIT/CLAIM/WITHDRAW;抄送与通知未实现;不得据此宣告 OA 可交付。

OA 候选领取、转交、代理与离职交接证据(10.3):新增 `oa_delegate_rule.sql`(按租户隔离、时间窗、`deleted=id` 软删除释放唯一键)与 `oa_delegate_menu.sql`(`system:oa:delegate:manage` 9614、`system:oa:employee:transfer` 9634,授权从具备 `system:user:list` 的角色/套餐复制)。后端新增 `OaDelegateRepository`/`OaDelegateService`:只启用且窗口覆盖当前时间的规则生效,同一委托人-代理人窗口不可重叠,窗口上限 90 天,A→B→A 代理环在写入前 BFS 拒绝;`OaApprovalEngine` 增加 `claim/reassign/removeCandidateLinks/runningTasksOfUser`;`OaApprovalService.claim/transfer` 复用 requestId 幂等与不可变审计,代理访问以 `proxyForId` 标记并在通过/驳回/退回审计详情中保留委托人;`OaTaskHandoverService` 逐条转交员工名下候选或已领取任务并清空候选链接,审计 `original_assignee_id` 保留原处理人,员工转为 RESIGNED 时优先自动转交直属主管,主管不可用不阻塞离职,管理员可调用 `POST /system/oa/staff/employee/{id}/handover` 补转。前端审批中心新增“限时代理”弹窗(创建/查看/取消、时间窗与 90 天校验)、待办“代某某”标签、详情页候选领取与已领取任务转交,员工页离职档案增加“转交待办”。测试证据:新增 `OaApprovalServiceTest` 5 项(领取幂等与他人领取被拒、未领取不可转交及转交后候选身份清理、代理待办/详情/代审批审计、代理重叠/环路/窗口上限/无权取消、离职转交候选任务),OA 模块 110 项运行 0 失败/错误、2 项 MySQL 用例无环境变量跳过;前端 `oaTaskList`、`oaTaskDetail` 扩充后 OA 相关 20 套件 117 项通过,`tsc --noEmit` 与改动文件 Biome 通过。边界:尚未做 10.3 浏览器实机验收,`oa_delegate_rule.sql` 未在真实 MySQL 重跑;代理规则只支持创建与取消,不支持编辑时间窗;代理访问仅一层,不展开二次委托;离职自动转交以更新前直属主管为准,主管同时离职时需手工指定;转交只改变当前任务处理人,不改变流程节点与历史审批记录;不得据此宣告 OA 可交付。

OA 会签策略证据(10.4):后端新增 `OaCountersignConditionValidator`,只接受 `${nrOfCompletedInstances / nrOfInstances >= 0.6}` 这类由实例计数(nrOfCompletedInstances/nrOfActiveInstances/nrOfInstances)与数字做四则运算后比较的完成条件,拒绝方法调用、属性访问、字符串、布尔逻辑与嵌套花括号,上限 200 字符;不写完成条件即全票通过。`OaBpmnValidator` 放行 `multiInstanceLoopCharacteristics` 与 `completionCondition`,要求 elementVariable 为合法变量名、collection 必须写成 `${变量List}`、会签用户任务的 assignee 必须等于 `${变量}`、不得配置 candidateUsers/candidateGroups、isSequential 仅 true/false,并对完成条件做白名单校验;会签 assignee 变量仍计入发起时需绑定的审批人变量,但不再要求只解析出一名员工。`OaProcessLauncher` 对每个审批人变量同时写入逗号串与原变量名 + List 的集合,供引擎多实例逐项创建任务。`OaApprovalEngine` 增加 `isMultiInstance` 与 `wasCancelled`(以历史任务 deleteReason 判定非正常完成),`OaApprovalService` 在通过/驳回后把 `countersign`、`cancelledTaskIds`、`cancelledAssignees`、`cancelReason` 写入不可变审计:比例通过达到阈值后引擎自动终止剩余任务,驳回删除实例实现一票否决并取消剩余任务。测试证据:`OaApprovalServiceTest` 15 项(新增比例通过完成并取消剩余、全票通过等待全部审批且幂等回放、驳回一票否决并取消剩余、比例阈值后只取消剩余任务),`OaBpmnValidatorTest` 23 项(新增接受会签与变量多值、拒绝不安全完成条件、拒绝集合/审批人不一致、拒绝会签候选范围),均 0 失败/错误。前端设计器同步支持会签:`src/utils/oaBpmn.ts` 增加多实例子节点/属性白名单、会签结构预检与比例条件编译/解析;`flowableModdle` 声明 collection/elementVariable;画布可创建/更新/关闭多实例并对 `${varList}`/`${var}` 做一致性约束,属性面板提供会签开关、全票/比例策略、比例百分比与顺序会签,候选范围在会签时禁用并提示清空。前端新增会签校验、属性面板与设计页接线用例;OA 20 个套件 127 项通过,`tsc --noEmit` 与改动文件 Biome 通过。另以临时 bpmn-moddle 脚本(未入库)验证多实例 XML 的 collection/elementVariable/completionCondition 序列化与回读。边界:尚未做 10.4 浏览器实机验收;并发场景由串行单测覆盖,未做多线程/多副本真实并发压测;会签流程的退回/撤回语义未单独回归(本轮仅覆盖通过、驳回);发起页对会签变量仍只支持单个员工或岗位/表单人员绑定,多员工会签需使用岗位或后续补多选;顺序会签只做结构与序列化支持,未在浏览器/引擎做顺序待办回归;不得据此宣告 OA 可交付。

OA 附件完整闭环证据(10.5):后端复用 `/system/file/upload`(不新增上传接口),上传仍由 `system:file:upload`(菜单 1113)控制;新增 `oa_attachment_menu.sql`(菜单 ID 9604,`system:oa:attachment:download`,挂在审批权限 9600 下,授权从已具备发起提交/审批列表/我的申请列表的角色与租户套餐复制,并给具备 `system:oa:launch:submit` 的角色与套餐补授上传权限)。`OaFormValueValidator` 把附件值收紧为 `{uid?,name?,fileId}`:必须是带正整数 fileId 的对象,同一字段重复引用直接拒绝,字符串或缺少 fileId 返回「附件未完成上传」。`OaAttachmentService.prepare` 在事务内按租户+上传人+未删除校验 `sys_file` 归属,拒绝目录、单文件超过 10MB、单表单超过 100 个附件与已被其他申请占用的附件(唯一约束 `(tenant_id,file_id,deleted)`),并用服务端 `original_name` 覆盖客户端文件名;申请落库后批量写入 `oa_attachment_link`(category=FORM、scan_status=SKIPPED、available=1,未接入杀毒扫描),任一步失败随事务回滚。新增 `GET /system/oa/attachment/{fileId}`(权限 `system:oa:attachment:download`):申请人、当前任务候选人/处理人(`OaApprovalEngine.hasTaskAccess`)与历史审批人(`oa_approval_audit`)可下载;跨租户、未关联、不可用统一返回「附件不存在或无权访问」,不泄露文件是否存在。下载按 `sys_file`/`sys_storage` 重建 x-file-storage `FileInfo`,以 UTF-8 `Content-Disposition` 流式返回,存储缺失映射为「附件文件不存在或已被清理」。测试证据:`OaLaunchServiceTest` 增至 9 项(新增附件落库与关联、他人上传被拒且无副作用、已关联/未上传被拒、申请人/当前处理人/历史审批人可下载而无关用户/跨租户/不可用被拒),`OaApprovalServiceTest` 构造同步更新;OA 模块 99 项运行,0 失败/错误,2 项 MySQL Liquibase 用例因未设置环境变量跳过;后端全量 190 项运行,0 失败/错误,2 项跳过。前端 `OaFormRenderer` 附件改为真实上传(`customRequest` 调用 `uploadFile`,10MB 前置拦截,上传中/失败/缺少 fileId 不允许提交),提交映射只保留服务端 fileId,申请与审批详情把附件渲染为鉴权下载按钮;`oaForm.test.ts` 补 fileId 解析与上传状态校验,`oaFormFill` 补真实上传回显,`oaApplicationDetail` 补下载调用断言;前端全量 Jest 37 套件 221 项通过,`tsc --noEmit`、改动文件 Biome 与契约检查(verbMismatch 0、noPath 0、unknownPermissions 0、missingPages 0)通过。边界:本批未启动完整前后端做浏览器实机验收,`oa_attachment_menu.sql` 与下载接口未在真实 MySQL/存储上重跑;未接入杀毒扫描(scan_status 固定 SKIPPED),OA 表单上传未启用秒传,上传后从表单移除文件不会清理未关联的 `sys_file`;附件不可跨申请复用,重新提交需重新上传;下载授权只覆盖当前实例待办与历史审批审计,抄送人/管理员代查未开放;不得据此宣告 OA 可交付。

OA outbox 通知、催办、超期与运维终止证据(10.6):新增 `oa_outbox`(事件 ID 唯一、`(tenant_id,dedup_key)` 业务去重、状态/重试时间索引)与 `oa_ops.sql`(outbox 去重键、`oa_notification` 站内通知表,接收人维度去重与未读索引),`oa_ops_menu.sql` 增加 OA 运维 `/oa/operations`(列表/重试/立即投递/终止权限)与通知消息 `/oa/notifications`(列表/已读权限),并在申请权限下增加催办权限,授权仍从具备 `system:user:list` 的角色与租户套餐复制。后端 `OaOutboxService` 在业务事务内写通知事件,`OaOutboxDispatcher` 原子领取待投递事件、以站内通知投递、失败按 `min(1800, base*2^(retry-1))` 指数退避并支持人工重试与立即投递,超时 PROCESSING 可恢复;`OaNotificationService` 提供分页、未读数、单条/全部已读并按租户与当前用户隔离;`OaUrgeService` 仅允许申请人催办 RUNNING 申请,按任务 10 分钟冷却、每日最多 5 次,只发送提醒事件,不修改任务到期时间;`OaOverdueService` 按任务到达后的自然小时(默认 24 小时)判定超期,每任务每天只提醒一次;`OaOpsService` 以独立 `system:oa:ops:terminate` 权限终止申请,必须填写原因,锁定申请后删除流程实例、置 `CANCELLED`、写不可变 TERMINATE 审计并通知申请人。定时任务默认开启并可按配置关闭:outbox 每 15 秒投递,超期每 10 分钟扫描。测试证据:新增 `OaOpsNotificationTest` 5 项(通知幂等投递、失败退避与人工重试、催办限频且不改任务时间、超期每日一次、终止写审计并通知),MySQL Liquibase 用例补充 outbox/notification 列、索引与去重唯一性断言;OA 模块 123 项运行 0 失败/错误,2 项 MySQL 用例因无隔离库环境变量跳过。前端新增通知收件箱与运维页,申请详情提供催办弹窗;定向 5 套件 30 项、前端全量 Jest 40 套件 252 项通过,`tsc --noEmit`、改动文件 Biome 与契约检查(verbMismatch 0、noPath 0、unknownPermissions 0、missingPages 0)通过。边界:尚未做 10.6 浏览器实机验收,迁移未在真实 MySQL 重跑;通知只做站内消息,未接邮件/短信/IM 外部通道;催办不升级、不累计 SLA;超期按自然小时而非工作日历;`oa_request_dedup` 与已发送 outbox 的清理作业仍未实现;投递原子领取仅单测覆盖,未做多节点并发压测;终止不支持指定替代审批人或恢复;不得据此宣告 OA 可交付。

## 4. 尚未完成的边界

- **App IM 实机**:同步与生命周期已修复并通过单测,但尚未做设备/模拟器联调,发送、推送、切后台与重连仍需真机验收。
- **server 消息协议**:提交后推送、会话行锁、单页 100 条与实际已读范围已完成;无 `afterId` 时仍返回最早一页,分页/已读协议的版本化发布顺序、App 对齐及真实 MySQL 并发/慢查询验证尚未完成。
- **backend 客服会话**:回复已改为提交后推送并有回滚/提交顺序单测;客服侧没有会话行可锁,并发回复依赖插入顺序,真实 MySQL 并发与多副本推送仍未验证。
- **多副本**:当前连接表在进程内,跨节点实时到达未实现;单副本交付需明确限制。
- **鸿蒙**:安全随机数已改为无安全源即拒绝生成(拒绝采样,不降级 `Math.random`),并有单测;持久化驱动仍未接入,原生存储包安装命令在本机无输出后终止,未产生依赖改动。上传契约已修复为 `/app/file/upload`(登录、目录/类型/大小/归属校验),但尚未连接真实存储;原生能力与真机验收仍未完成。
- **个人工具**:灵感便笺与物品收纳均已完成服务端分页 CRUD、归属/版本冲突测试与 App 页面接线,尚未连真实服务或真机验收;物品数量并发依赖真实 MySQL 复验。
- **后台性能**:尚无隔离数据集和前后基线,未改索引或缓存。
- **OA/BPMN**:生产引擎已接入并由 Liquibase 独占迁移,隔离 H2 与真实 MySQL 8.4 验证了租户隔离、事务回滚、关闭自动建表后的重启与定时器恢复;流程模型/发布/表单/表单版本/申请/审批审计/抄送回执/附件归属/outbox/通知/命令幂等/代理规则/岗位/员工/员工岗位十五张表已建并验证租户与全局唯一约束,员工组织与五种审批人来源的解析、发布校验和发起前拦截已打通,发起入口已按发布状态与结构化可发起范围拦截,提交已接入服务端表单校验、审批人自动/兜底绑定与请求幂等(10.1),审批中心已接入待办/已办/任务详情、通过/驳回/退回与发起人撤回(10.2),员工/岗位 CRUD API、管理页与人员/岗位选择器已接入,但仍无实体/Mapper,幂等记录与已发送 outbox 的清理作业未实现,候选解析只覆盖发起时已知变量(后续节点任务创建监听未接);候选领取、任务转交、限时代理与离职存量待办转交已实现并有单测(10.3),会签比例/全票通过、驳回取消剩余任务与设计器会签配置已实现并有单测(10.4),outbox 投递、站内通知、催办限频、超期提醒与独立授权的运维终止已实现并有单测(10.6),但 10.3/10.4/10.6 均未做浏览器实机验收,并行分支与抄送仍未实现,各 OA 业务表未建。异步执行器未启用、完整应用仅在 dev profile 的上下文测试中启动过(非独立部署验收),多副本与长稳未验证。BPMN 发布前 XML 安全校验、节点/属性白名单、受限条件编译与受控服务任务已完成并单测覆盖,发布/部署事务、不可变发布版本、停用与 OA 菜单/权限种子已接通并有旧实例不迁移回归,但发起入口已按发布状态与结构化可发起范围拦截(10.1),表单人员与直属主管只在发起时解析,服务任务类未验证存在性,条件未在真实引擎执行语义回归;流程设计器已有画布/属性/导入导出/预检/预览,并可按模型加载、保存草稿、发布与停用;完整链路已在隔离 MySQL 8.4.11 + 真实浏览器走通(3.6),菜单 SQL 已随完整迁移落库;表单设计器、不可变表单版本、版本下拉选择、发布前静态员工/岗位校验,以及按不可变版本渲染的表单填写页与本地校验已实现并有单测,发起列表/填写页、幂等提交、我的申请列表/详情与申请人隔离已接入(10.1),审批中心待办/已办、任务详情、通过/驳回/退回与发起人撤回已接入(10.2),附件上传校验、归属落库与鉴权下载已接入(10.5);9.8/10.1/10.2/10.5 已在隔离 MySQL 库 + 真实浏览器走通发起、审批通过/驳回/撤回、附件上传下载与员工/岗位选择器闭环(3.7),退回上一节点与人员姓名回显仍未实机,抄送及任何 OA 业务模块均未实现;不得把引擎接入、建表、XML 校验、发布/停用接口、设计器页面或表单配置当作 OA 可交付。

## 5. 后续分期与验收门槛

1. **A 消息正确性(继续)**:App 已修发送/去重/生命周期,server 已完成提交后推送、会话行锁与有界读取,backend 客服回复已收口事务与推送顺序;下一步统一分页/已读协议并在真实数据库验证并发写入、已读与游标。
2. **B 鸿蒙与个人工具**:修复持久化与安全随机数,经 `expo-harmony-cli` 检查兼容并生成原生工程;实现便笺与物品收纳的归属隔离和乐观锁;鸿蒙以签名包真机验收。
3. **C 性能与 BPMN 底座**:建立可复现性能基线后再优化;已完成 Flowable 7.2.0 生产接入(Liquibase 托管、租户/事务/重启/定时器验证)、十五张 OA 业务表迁移、员工/岗位/主管候选解析、BPMN 发布安全校验、bpmn-js 设计器与流程模型发布/停用,并在隔离 MySQL 8.4.11 + 真实浏览器走通设计-校验-发布-停用-重发链路(3.6);已完成表单设计与表单版本(9.8,含 8 类字段、拖拽设计器、不可变版本、版本下拉选择、只读/隐藏策略、表单渲染填写页与发布前静态员工/岗位校验);已完成发起入口与结构化发起范围拦截、服务端表单校验、审批人自动/兜底绑定、请求幂等提交、我的申请列表/详情与菜单权限(10.1),以及审批中心待办/已办、任务详情、通过/驳回/退回上一人工节点、发起人撤回与幂等审计(10.2),附件上传校验、归属落库、鉴权下载与前端上传/回显/下载已接入(10.5),员工/岗位 CRUD 与人员/岗位选择器已接入,并在隔离 MySQL 库 + 真实浏览器走通发起、审批通过/驳回/撤回、附件上传下载与员工/岗位选择器闭环(3.7,退回上一节点与人员姓名回显仍未实机);候选领取、任务转交、限时代理与离职存量待办转交已完成静态验证(10.3),会签策略与设计器会签配置已完成静态验证(10.4),outbox 通知、催办、超期与运维终止已完成静态验证(10.6),均待浏览器实机验收。
4. **D 请假/报销试点**:额度并发、审批幂等、审批与支付分离、浏览器全流程。
5. **E 其余 OA**:出差借款、假勤、采购资产、合同用印、公文知识、会议车辆、报表运维逐模块验收;不得以通用表和菜单冒充完成。

每个批次必须同时具备页面、API、持久化、权限、失败路径和验收证据;只有静态检查或 mock 测试时,只能报告为"静态验证",不能报告为上线完成。

## 6. 实施前仍需确认

- 真实测试 MySQL/Redis 与可回滚迁移环境。
- 当前移动端是否存在无法同步升级的用户,以及 server/backend 是否多副本运行。
- Flowable 异步执行器启用时机、定时任务在多副本下的归属,以及 OA 业务表迁移的评审窗口。
- 组织/主管示例、工作日历、假期与金额阈值。
- 私密附件存储方案;鸿蒙签名、设备;微信合法域名。
