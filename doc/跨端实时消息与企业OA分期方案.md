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

OA BPMN 发布安全校验证据:`OaBpmnValidator` 以命名空间感知 DOM 解析,关闭 DOCTYPE、外部通用/参数实体与外部 DTD,启用 secure processing,`ACCESS_EXTERNAL_DTD`/`SCHEMA` 置空并由实体解析器直接抛错;预检 `<!DOCTYPE`/`<!ENTITY`,限制 512KB/200 个流程节点/2000 个元素/50 层嵌套,错误最多保留 40 条,并对原始 XML 计算 SHA-256。节点白名单为 definitions/process/startEvent/endEvent/userTask/serviceTask/exclusiveGateway/parallelGateway/boundaryEvent/sequenceFlow/conditionExpression/timerEventDefinition/timeDuration/timeDate/timeCycle/incoming/outgoing,另放行 bpmn-js 所需 BPMNDiagram/BPMNPlane/BPMNShape/BPMNEdge/BPMNLabel、dc:Bounds、di:waypoint;scriptTask、callActivity、subProcess、extensionElements、camunda/activiti 属性与任意 Flowable 子元素均拒绝。结构上要求单流程且 isExecutable=true、id 唯一、开始/结束事件存在、连线引用有效、开始事件不可作为目标、边界定时器只能挂用户任务、条件只能位于排他网关输出连线且默认连线有效、默认连线不能配置条件、用户任务至少配置一种审批人来源、排他网关多出口除默认连线外必须有条件。服务任务只接受白名单内的 `flowable:class`,`flowable:expression`/`delegateExpression` 一律拒绝;白名单通过 `oa.bpmn.allowed-service-handlers` 逗号配置,默认空即全部拒绝。`OaConditionRuleCompiler` 把结构化 AND/OR 条件编译为受限 EL(EQ/NE/GT/GE/LT/LE/IN/NOT_IN,字段仅标识符/点路径,值仅数字/布尔/字符串或集合),再由 `OaConditionExpressionValidator` 词法/语法复核,只允许变量、字面量、比较、括号与 &&/||/!,拒绝方法调用、构造函数、`T(...)`、Bean 引用、三元、算术与 EL 关键字。`OaBpmnValidatorTest` 17 项、`OaConditionRuleCompilerTest` 9 项、`OaBpmnValidationConfigurationTest` 2 项通过,模块合计 48 项(46 项执行通过,MySQL 2 项无环境变量跳过)。边界:尚未接入发布/部署事务与版本落库(9.7),当前只校验 XML 不实际部署;服务任务白名单不验证类是否存在或实现 JavaDelegate;条件只做安全/语法校验,未在真实引擎执行语义回归;DI 只做放行,不校验图形引用完整性;未用项目开发库启动完整应用。

OA 流程模型发布/停用证据:后端新增 `/system/oa/process/model` 控制器、服务与显式租户 SQL 仓储,提供分页/详情/创建/保存(乐观锁)/XML 校验/发布/停用/版本列表;列表 SQL 不读取 `bpmn_xml`。发布在同一事务内完成服务端 XML 复核、流程 Key 稳定性与跨模型占用检查、表单版本租户校验、Flowable 部署、不可变发布记录写入、旧 ACTIVE 版本停止与模型指针更新,任一步失败回滚;再次发布生成新版本,运行中的旧实例仍绑定原流程定义,停用只停止 ACTIVE 发布记录且之后可再次发布恢复。新增 `oa_menu.sql`(流程管理/流程模型/隐藏设计器与 list/create/update/publish/disable 权限),仅向已具备 `system:user:list` 的角色和租户套餐复制授权。`OaProcessModelServiceTest` 5 项通过:创建草稿并拒绝过期版本、发布绑定表单并激活版本、v1 实例在发布 v2 后仍按 v1 完成且新实例使用 v2、非法 XML 不产生部署或发布记录、停用后可重新发布且他模型占用同一流程 Key 被拒;OA 模块合计 54 项,52 项执行通过,MySQL 2 项无环境变量跳过。前端新增 OA 模型 service、`/oa/process` 列表页和设计页按 `id` 绑定:列表支持搜索/筛选、新建(按模型标识生成独立流程 Key)、设计、发布(表单版本/发起范围/说明并展示服务端拒绝原因)、停用与版本历史;设计页支持加载模型、保存草稿、服务端校验、发布前自动保存、停用确认及状态/版本展示。OA 相关 7 个前端套件 70 项通过,`tsc --noEmit` 与 `build:zy` 通过,两个路由均进入产物。边界:自动 MySQL Liquibase 用例仍因未设置环境变量跳过,但 `oa_menu.sql` 已随完整迁移在 3.6 的隔离 MySQL 8.4.11 落库;真实登录态浏览器链路已由 3.6 走通;发布当时未调用 `OaAssigneeResolver` 校验员工/岗位存在性与在职状态(已由 9.8 补齐静态来源校验);发起入口尚未按发布状态与可发起范围拦截(10.1);列表页暂无名称/分类/说明编辑入口。

OA 流程设计器前端证据:新增 `/oa/process/design` 页面与 `OaBpmnCanvas`/`bpmnLoader`/`flowableModdle`/`OaBpmnProperties`/`OaConditionBuilder`/`OaBpmnPreview` 组件,`src/utils/oaBpmn.ts` 提供与服务端同构的预检、默认模型和受限条件编译。bpmn-js 18.31.1 的 `Modeler`/`NavigatedViewer` 通过动态 `import()` 懒加载;`FLOWABLE_MODDLE` 只声明用户任务 assignee/candidateUsers/candidateGroups 与服务任务 class。页面支持新建、导入 `.bpmn/.xml`、导出、校验、只读预览、撤销/重做/适应画布;属性面板可编辑名称、用户任务审批范围、服务任务处理器、网关默认连线、边界定时器与排他网关输出条件,条件构建器把 AND/OR 结构化条件编译为受限 EL。前端预检拒绝 DOCTYPE/ENTITY、超 512KB、超 200 流程节点/2000 元素、非白名单节点/属性、非法审批范围/定时器/条件,并校验开始/结束事件、连线引用、边界事件挂载与默认连线;服务端仍是最终裁决方。真实 bpmn-js 运行时以临时 Node+jsdom 脚本冒烟(未入库)验证名称、候选审批人、条件、默认连线与定时器序列化,并发现 bpmn-js 对“默认连线”和“连线条件”互斥清空,因此客户端与服务端同时新增“默认连线不能配置条件”,属性面板对默认连线隐藏条件编辑,`OaBpmnValidatorTest` 增加 `rejectsConditionOnDefaultFlow` 并 17 项通过。前端 5 个套件 45 项测试通过,`tsc --noEmit` 通过,`build:zy` 12.9s 成功,页面 chunk(约 48KB)异步引用 bpmn-js chunk(约 322KB)。边界:无保存/发布 API,发布按钮禁用;`sys_menu` 无 OA 菜单,`RouteAccess` 可能拦截路由,需 9.7 补菜单与权限;未在真实登录态浏览器连接后端走通;画布 Jest 用例使用假构造器验证接线,真实 bpmn-js 未纳入 CI;设计期预检不替代服务端校验。上述 API/菜单/浏览器边界已由 9.7 与 3.6 闭环。

### 3.6 OA 流程模型浏览器实机验收

2026-11-06 在隔离 MySQL 8.4.11 库 `gf_oa_e2e_1106`、Redis DB 15 上启动完整后端(`:8000`, dev profile)与前端开发服务器(`:9002`),用内置浏览器以 admin 登录走通真实链路;隔离库内关闭登录验证码,验收后已删除库、清空 Redis DB 15 并释放端口。实测链路:登录进入工作台;`/oa/process` 通过真实 API 渲染空列表;新建模型 id=1(key `leave_e2e_1106`,名称「请假流程 E2E」);设计页加载 bpmn-js 画布并显示发起/审批/结束三节点;前端预检与服务端校验均通过;保存草稿生成 v1;发布返回「版本 v1(引擎版本 1)」,`oa_process_release` v1=ACTIVE,`ACT_RE_PROCDEF` 生成 tenant 0 的 `leave_e2e_1106:1:...` 定义;列表与版本历史显示 v1 生效中;停用后模型 DISABLED、v1 转 STOPPED、草稿 v3;停用后重新发布生成 v2(引擎版本 2),模型 `current_release_version=2`、乐观锁 `version=4`,`oa_process_release` v1=STOPPED/v2=ACTIVE,引擎侧 v1、v2 两个流程定义各自保留独立 deployment。实机同时确认 `oa_menu.sql` 已随完整 Liquibase 迁移落在真实 MySQL,OA 菜单与权限可正常访问;补 `menu.oa*` 与页签通用 i18n key 后,`/oa/process` 与 `/oa/process/design` 控制台不再出现缺键告警,设计器页签正确显示「流程设计器」。

边界:内置浏览器页面处于隐藏态,无法截图,证据来自 DOM 快照、真实网络响应与数据库查询;弹窗关闭动画因页面隐藏而冻结在 DOM,非产品缺陷。未覆盖表单填写、发起、审批、抄送与附件;发布强制绑定表单版本已由后续批次补齐,发布时的静态员工/岗位校验也已由 9.8 补齐(表单人员与直属主管仍在发起时解析),发起入口未按发布状态/发起范围拦截,异步执行器仍关闭。该批次只证明「流程模型设计-校验-发布-停用-重发」链路可用,不代表 OA 业务可交付。

OA 发布强制绑定表单版本证据:后端在 XML 复核前拒绝缺失 `formVersionId` 的发布请求,返回「发布必须绑定表单版本」;表单版本仍按租户查询,不存在或跨租户返回「绑定的表单版本不存在或不属于当前租户」;任一拒绝都不产生 Flowable 部署或发布记录,模型保持原状态。前端两处发布弹窗把「表单版本 ID」改为必填并前置校验,校验失败不发送请求;`OaProcessPublishParams` 类型同步为必填。`OaProcessModelServiceTest` 6 项通过(新增缺失表单版本被拒且无部署、无发布记录、模型仍为草稿),相关 3 个前端套件 29 项与 `tsc --noEmit` 通过。边界:表单设计器与表单版本列表 API 仍未实现,发布时仍需手工填写已存在的版本 ID;本批次未重新启动完整前后端做浏览器实机回归。上述表单设计与版本列表边界已由 9.8 闭环,发布已改为下拉选择版本。

OA 表单设计与表单版本证据(9.8):后端新增 `oa_form_draft.sql`,为 `oa_form` 增加可编辑草稿 `draft_schema_json` 与 SHA-256 `draft_schema_digest`;新增 `oa_form_menu.sql`(表单管理 `/oa/form`、隐藏设计器 `/oa/form/design` 与 list/create/update/publish/disable 权限,仅向已具备 `system:user:list` 的角色和租户套餐复制授权)。`OaFormSchemaValidator` 校验 8 类字段(text/textarea/number/date/radio/checkbox/attachment/person),限制 1MB/100 字段/50 选项,拒绝重复 id/key、单选多选缺选项与非法 JSON,草稿允许空字段、发布要求至少一个字段,并对 schema 计算 SHA-256。`/system/oa/form` 提供分页/详情/创建/保存草稿(乐观锁)/发布/停用/版本列表/版本下拉选项;创建与更新只写草稿,发布在同一事务内复核 schema、写入不可变 `oa_form_version` 快照并推进当前版本指针,非法发布不产生版本或指针变更;版本列表不返回 schema,版本选项只返回 PUBLISHED 表单且上限 500 条,所有 SQL 显式带 tenant_id。流程发布新增静态审批人来源校验:`employee:101`、`position:FINANCE`、`formField:approvers`、`deptLeader`、`supervisor` 由 `OaAssigneeSpecParser` 解析,发布时校验指定员工/岗位/部门负责人是否存在且在岗,`supervisor` 与 `formField` 只校验结构、留待发起时解析,`${var}` 与普通值仍视为运行时变量跳过;校验失败在 Flowable 部署前返回拒绝。前端新增 `/oa/form` 列表(搜索/筛选、新建、发布、停用、版本历史)与 `/oa/form/design` 设计器(8 类字段面板、HTML5 原生拖拽排序、属性面板、实时预览、保存草稿/发布/停用/版本历史、未保存离开确认);流程列表页与设计页的发布弹窗改为从 `version-options` 下拉选择表单版本,不再手工输入版本 ID。测试证据:本轮新增 `OaFormSchemaValidatorTest` 7 项、`OaFormServiceTest` 6 项、`OaAssigneeSpecParserTest` 4 项,`OaProcessModelServiceTest` 增至 7 项(含发布前拒绝不可用静态审批人且不部署);OA 模块全量 73 项运行,0 失败/错误,2 项 MySQL Liquibase 用例因未设置环境变量跳过。前端 `oaForm.test.ts`、`oaFormList`、`oaFormDesign`、`oaProcessList`、`oaProcessDesign` 5 个套件 29 项通过,前端全量 Jest 27 套件 185 项通过,`tsc --noEmit` 通过,改动文件 lint 干净(仅保留一处既有的 `${...}` 字面量告警)。边界:表单人员与直属主管只在发起时解析,尚无表单填写/发起入口(10.1);员工/岗位仍无 CRUD API 与页面;本批未启动完整前后端做浏览器实机验收,前端证据仅为 Jest/类型/静态检查;自动 MySQL Liquibase 用例仍因未设置环境变量跳过;不得据此宣告 OA 可交付。

## 4. 尚未完成的边界

- **App IM 实机**:同步与生命周期已修复并通过单测,但尚未做设备/模拟器联调,发送、推送、切后台与重连仍需真机验收。
- **server 消息协议**:提交后推送、会话行锁、单页 100 条与实际已读范围已完成;无 `afterId` 时仍返回最早一页,分页/已读协议的版本化发布顺序、App 对齐及真实 MySQL 并发/慢查询验证尚未完成。
- **backend 客服会话**:回复已改为提交后推送并有回滚/提交顺序单测;客服侧没有会话行可锁,并发回复依赖插入顺序,真实 MySQL 并发与多副本推送仍未验证。
- **多副本**:当前连接表在进程内,跨节点实时到达未实现;单副本交付需明确限制。
- **鸿蒙**:安全随机数已改为无安全源即拒绝生成(拒绝采样,不降级 `Math.random`),并有单测;持久化驱动仍未接入,原生存储包安装命令在本机无输出后终止,未产生依赖改动。上传契约已修复为 `/app/file/upload`(登录、目录/类型/大小/归属校验),但尚未连接真实存储;原生能力与真机验收仍未完成。
- **个人工具**:灵感便笺与物品收纳均已完成服务端分页 CRUD、归属/版本冲突测试与 App 页面接线,尚未连真实服务或真机验收;物品数量并发依赖真实 MySQL 复验。
- **后台性能**:尚无隔离数据集和前后基线,未改索引或缓存。
- **OA/BPMN**:生产引擎已接入并由 Liquibase 独占迁移,隔离 H2 与真实 MySQL 8.4 验证了租户隔离、事务回滚、关闭自动建表后的重启与定时器恢复;流程模型/发布/表单/表单版本/申请/审批审计/抄送回执/附件归属/outbox/命令幂等/岗位/员工/员工岗位十三张表已建并验证租户与全局唯一约束,员工组织与五种审批人来源的解析、发布校验和发起前拦截已打通,但无实体/Mapper/API 与员工岗位 CRUD 页面,outbox 投递与幂等清理作业未实现,候选解析只覆盖发起时已知变量(后续节点任务创建监听未接),离职员工存量待办不会自动转交,各业务表未建。异步执行器未启用、完整应用未在项目开发库启动,多副本与长稳未验证。BPMN 发布前 XML 安全校验、节点/属性白名单、受限条件编译与受控服务任务已完成并单测覆盖,发布/部署事务、不可变发布版本、停用与 OA 菜单/权限种子已接通并有旧实例不迁移回归,但发起入口未按发布状态/可发起范围拦截,表单人员与直属主管只在发起时解析,服务任务类未验证存在性,条件未在真实引擎执行语义回归;流程设计器已有画布/属性/导入导出/预检/预览,并可按模型加载、保存草稿、发布与停用;完整链路已在隔离 MySQL 8.4.11 + 真实浏览器走通(3.6),菜单 SQL 已随完整迁移落库;表单设计器、不可变表单版本、版本下拉选择与发布前静态员工/岗位校验已实现并有单测,但 9.8 尚未做浏览器实机验收,表单填写/发起入口、审批中心及任何 OA 业务模块均未实现;不得把引擎接入、建表、XML 校验、发布/停用接口、设计器页面或表单配置当作 OA 可交付。

## 5. 后续分期与验收门槛

1. **A 消息正确性(继续)**:App 已修发送/去重/生命周期,server 已完成提交后推送、会话行锁与有界读取,backend 客服回复已收口事务与推送顺序;下一步统一分页/已读协议并在真实数据库验证并发写入、已读与游标。
2. **B 鸿蒙与个人工具**:修复持久化与安全随机数,经 `expo-harmony-cli` 检查兼容并生成原生工程;实现便笺与物品收纳的归属隔离和乐观锁;鸿蒙以签名包真机验收。
3. **C 性能与 BPMN 底座**:建立可复现性能基线后再优化;已完成 Flowable 7.2.0 生产接入(Liquibase 托管、租户/事务/重启/定时器验证)、十三张 OA 业务表迁移、员工/岗位/主管候选解析、BPMN 发布安全校验、bpmn-js 设计器与流程模型发布/停用,并在隔离 MySQL 8.4.11 + 真实浏览器走通设计-校验-发布-停用-重发链路(3.6);已完成表单设计与表单版本(9.8,含 8 类字段、拖拽设计器、不可变版本、版本下拉选择与发布前静态员工/岗位校验),下一步发起入口与发起范围拦截(10.1),并补员工/岗位 CRUD 与 9.8 浏览器实机验收。
4. **D 请假/报销试点**:额度并发、审批幂等、审批与支付分离、浏览器全流程。
5. **E 其余 OA**:出差借款、假勤、采购资产、合同用印、公文知识、会议车辆、报表运维逐模块验收;不得以通用表和菜单冒充完成。

每个批次必须同时具备页面、API、持久化、权限、失败路径和验收证据;只有静态检查或 mock 测试时,只能报告为"静态验证",不能报告为上线完成。

## 6. 实施前仍需确认

- 真实测试 MySQL/Redis 与可回滚迁移环境。
- 当前移动端是否存在无法同步升级的用户,以及 server/backend 是否多副本运行。
- Flowable 异步执行器启用时机、定时任务在多副本下的归属,以及 OA 业务表迁移的评审窗口。
- 组织/主管示例、工作日历、假期与金额阈值。
- 私密附件存储方案;鸿蒙签名、设备;微信合法域名。
