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
| App 测试 | `npx jest --ci --watchAll=false --runInBand` | 16 套件 109 测试通过(新增聊天同步 8 项、连接生命周期 6 项) |
| App ESLint | 对本批改动文件执行 `npx eslint --max-warnings=0` | 通过 |
| 后端 IM 回归 | `mvn ... -Dtest=SupportUnreadRegressionTest,ImRegressionTest` | 7 + 11 项通过,0 失败/错误/跳过 |
| 服务端 WebSocket 回归 | `mvn ... -Dtest=ImWebSocketSecurityTest,ImAppBoundaryTest,AdminImServiceImplTest` | 57 + 2 + 5 项通过,0 失败/错误/跳过 |
| 服务端事务/分页回归 | `mvn ... -Dtest=ImTransactionPagingTest,...` | 新增 4 项,IM 合计 68 项通过 |
| backend 客服事务回归 | `mvn ... -Dtest=ImRegressionTest` | 15 项通过,含提交前不推送/提交后推送/回滚不推送 |
| server 客服事务/分页回归 | `mvn ... -Dtest=AdminImServiceImplTest` | 9 项通过,含最近100条/增量100条/只标返回ID/提交后推送 |

浏览器联调使用页面内注入的模拟数据,不是真实 Java 服务;未验证真实 MySQL/Redis、微信真机、鸿蒙真机或多副本部署。App 本批仅完成静态检查与单元测试,未在 Android/iOS/鸿蒙真机或模拟器运行,也未连接真实 server。

工具批证据:server 便笺接口 `ToolNoteServiceImplTest` 7 项通过;App `/tools/notes` 通过类型检查、109 项既有测试与改动文件 ESLint,未连接真实服务,7.2 仍需浏览器/鸿蒙验收。物品收纳已补齐 `gf_tool_item`、`/app/tool/item` 分页 CRUD、数量非负 CAS 与归档;`ToolItemServiceImplTest` 9 项通过,工具类合计 16 项。App 新增 `/tools/items` 页面与入口,数量调整冲突时回刷;同样未连真实服务或真机,7.4 保持未完成。

## 4. 尚未完成的边界

- **App IM 实机**:同步与生命周期已修复并通过单测,但尚未做设备/模拟器联调,发送、推送、切后台与重连仍需真机验收。
- **server 消息协议**:提交后推送、会话行锁、单页 100 条与实际已读范围已完成;无 `afterId` 时仍返回最早一页,分页/已读协议的版本化发布顺序、App 对齐及真实 MySQL 并发/慢查询验证尚未完成。
- **backend 客服会话**:回复已改为提交后推送并有回滚/提交顺序单测;客服侧没有会话行可锁,并发回复依赖插入顺序,真实 MySQL 并发与多副本推送仍未验证。
- **多副本**:当前连接表在进程内,跨节点实时到达未实现;单副本交付需明确限制。
- **鸿蒙**:持久化、安全随机数、上传契约、原生能力与真机验收均未完成。
- **个人工具**:灵感便笺与物品收纳均已完成服务端分页 CRUD、归属/版本冲突测试与 App 页面接线,尚未连真实服务或真机验收;物品数量并发依赖真实 MySQL 复验。
- **后台性能**:尚无隔离数据集和前后基线,未改索引或缓存。
- **OA/BPMN**:仅完成设计与规格;未安装引擎、未建表、未实现任何审批业务。

## 5. 后续分期与验收门槛

1. **A 消息正确性(继续)**:App 已修发送/去重/生命周期,server 已完成提交后推送、会话行锁与有界读取,backend 客服回复已收口事务与推送顺序;下一步统一分页/已读协议并在真实数据库验证并发写入、已读与游标。
2. **B 鸿蒙与个人工具**:修复持久化与安全随机数,经 `expo-harmony-cli` 检查兼容并生成原生工程;实现便笺与物品收纳的归属隔离和乐观锁;鸿蒙以签名包真机验收。
3. **C 性能与 BPMN 底座**:建立可复现性能基线后再优化;核验 Flowable 与 Spring Boot 3.4/JDK 21/Jakarta 兼容,隔离库验证引擎建表与重启定时器恢复,接入 bpmn-js 与 XML 白名单。
4. **D 请假/报销试点**:额度并发、审批幂等、审批与支付分离、浏览器全流程。
5. **E 其余 OA**:出差借款、假勤、采购资产、合同用印、公文知识、会议车辆、报表运维逐模块验收;不得以通用表和菜单冒充完成。

每个批次必须同时具备页面、API、持久化、权限、失败路径和验收证据;只有静态检查或 mock 测试时,只能报告为"静态验证",不能报告为上线完成。

## 6. 实施前仍需确认

- 真实测试 MySQL/Redis 与可回滚迁移环境。
- 当前移动端是否存在无法同步升级的用户,以及 server/backend 是否多副本运行。
- Flowable 具体版本兼容结果与许可证。
- 组织/主管示例、工作日历、假期与金额阈值。
- 私密附件存储方案;鸿蒙签名、设备;微信合法域名。
