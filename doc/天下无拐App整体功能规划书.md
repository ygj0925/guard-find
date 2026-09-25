# 天下无拐 App 项目整体功能规划书

| 项目 | 内容 |
| --- | --- |
| 产品名称 | 天下无拐(guard-find) |
| 文档版本 | v1.0 |
| 编写日期 | 2026-09-24 |
| 覆盖范围 | `guard-find-app`(iOS / Android / HarmonyOS 三端)、`guard-find-server`(用户侧服务)、寻亲数据采集子系统 |
| 关联仓库 | `guard-find-frontend` / `guard-find-backend`(运营后台,本规划书只定义对其的接口与治理要求) |
| 当前代码基线 | App 与 Server 均为**零业务代码**:App 是 Expo SDK 52 官方模板页,Server 是 ContiNew Admin 4.2.0 改名分支。本规划书是"从空库建业务"的施工图,不是改造说明 |

---

## 1. 产品定位

### 1.1 一句话定位

把"失踪儿童与走失人员的公开信息"变成一张**可信、可检索、可就近扩散**的协作网络:家属能在 3 分钟内发布一条结构化寻人档案,普通人能在刷广场时把线索送到对的人手里,志愿者和民警能在后台把散落各站点的历史档案收敛成一份可去重、可追溯、可下架的统一数据。

### 1.2 与"信息转发群"的差异

现在寻亲信息主要靠微信群和短视频转发,有三个致命问题:信息**非结构化**(没有统一的失踪地点、时间、体貌字段,无法按地域检索)、**不可信**(拼接假信息、借寻亲做募捐诈骗、"找到即删帖"导致线索断流)、**不沉淀**(转发完即失联,没人知道这条线索有没有被核实)。本产品用结构化档案 + 审核状态机 + 线索回执把这三件事补上。

### 1.3 目标用户与角色

| 角色 | 说明 | 关键诉求 | 一期是否支持 |
| --- | --- | --- | --- |
| 寻亲家属 | 失踪/走失人员的直系亲属 | 快速发布、只披露可控信息、收到可核实的线索 | ✅ |
| 普通用户(好心人) | 刷信息流、提供线索、扩散 | 就近看到身边案例、一键提供线索、能追踪结果 | ✅ |
| 志愿者 | 宝贝回家等志愿体系的志愿者 | 认领跟进、批量核对采集候选、更新档案状态 | 二期(一期仅注册标识) |
| 民警 / 机构 | 打拐民警、救助管理站 | 权威发布、跨库比对、状态权威确认 | 三期 |
| 运营审核员 | 后台 `guard-find-frontend` 使用者 | 审核发布与采集候选、处理举报、下架与删除 | 一期提供接口,后台界面二期 |

### 1.4 北极星与关键指标

北极星指标:**每周产生有效线索的寻人档案数**。

一期上线后需能采集的指标:发布成功率(目标 > 92%)、发布到审核通过的中位时长(目标 < 30 分钟人工时段)、档案 7 日线索提交数、按省市检索的档案覆盖度、采集候选去重准确率(抽样 > 98%)、单条档案的来源链接留痕率(必须 100%)。

---

## 2. 技术基线与三端约束

### 2.1 已确定的技术栈(不可随意替换)

**App(`guard-find-app`)**

| 项 | 值 | 来源 |
| --- | --- | --- |
| Expo SDK | 52.0.49 | `package.json` |
| React Native | 0.77.1(new architecture 已开) | `package.json` |
| HarmonyOS 运行时 | `@react-native-oh/react-native-harmony` 0.77.71 | `package.json` |
| 路由 | expo-router 4.0.22(经 425 行 patch 支持 harmony 平台) | `patches/expo-router+4.0.22.patch` |
| 样式 | NativeWind 4.1.23 + Tailwind 3.4.19,shadcn "new-york" 语义 token | `tailwind.config.js` / `global.css` |
| UI 原语 | react-native-reusables(`@rn-primitives/*` × 24,`components/ui/` × 31) | `components.json` |
| 包管理 | pnpm,`node-linker=hoisted`(否则 release 包 PNG 资源解析失败) | `.npmrc` |

**Server(`guard-find-server`)**

| 项 | 值 |
| --- | --- |
| Java / Spring Boot | 17 / 3.4.10 |
| 框架底座 | ContiNew Admin 4.2.0-SNAPSHOT + `top.continew.starter` 2.15.0 |
| 持久层 | MyBatis-Plus 3.5.14 + Liquibase 4.29.2 + MySQL(p6spy dev)+ CosId 雪花 ID |
| 鉴权 | Sa-Token 1.44.0(jwt-simple)+ Redis 会话 + JustAuth 三方登录 |
| 缓存 / 消息 | Redisson 3.52.0 + JetCache;站内信 + WebSocket(`/websocket?token=`) |
| 短信 / 邮件 | SMS4J 3.3.5 + `continew-starter-messaging-mail`,FreeMarker 模板 |
| 文件 | X File Storage 2.2.1(本地 / S3)+ 分片上传 |
| 调度 | SnailJob(dev 环境 `snail-job.enabled: false`,走 Spring `@Scheduled` 分支) |
| HTTP 客户端 | Hutool `HttpRequest`(项目内无 RestTemplate/WebClient Bean) |

### 2.2 三端能力矩阵与降级策略

鸿蒙侧的真实约束来自 RNOH 适配现状,必须在选型阶段就决定降级路径,不能等到联调。

| 能力 | iOS | Android | HarmonyOS | 一期方案 |
| --- | --- | --- | --- | --- |
| 路由 / 列表 / 表单 / 上传 UI | ✅ | ✅ | ✅ | 直接做 |
| 网络请求 | ✅ | ✅ | ✅ | 纯 JS `fetch`,三端一致(不引 axios 原生依赖) |
| 本地 KV(token 存储) | ✅ | ✅ | ⚠️ `expo-secure-store` 未在兼容表 | 抽象 `lib/storage`,harmony 走 `@react-native-ohos` 或 JS 实现,用 `.harmony.ts` 分派 |
| 定位 / 地图 | `expo-location` ✅ | ✅ | ❌ 无已验证适配 | **一期不做地图**,用"省 / 市 / 区三级选择器 + 文字地址";二期以 `.harmony.tsx` 分派单独适配 |
| 相机 / 相册 | `expo-camera` ❌ 无验证;`expo-image-picker` ⚠️ 有 patch 路径 | ✅ | ⚠️ | 一期用 `expo-image-picker`(需 `pnpm dlx expo-harmony-cli install` 并验证),失败则降级为"仅后端表单 + 相册系统选择器" |
| 图片加载 | `expo-image` ⚠️ | ⚠️ | ❌ `expo-asset` 在 harmony 被 shim 成 no-op,**远程 asset 下载不可用** | **一律用 `react-native` 原生 `Image` + `uri`**,禁止依赖 `expo-asset` / `Asset.downloadAsync` |
| 推送 | APNs | FCM | ❌ 无验证的 Expo 推送适配 | 一期站内消息 + WebSocket + 前台轮询;厂商推送二期 |
| JS 错误红框 | ✅ | ✅ | ❌ `shims/expo-metro-runtime.ts` 关掉了 error overlay | 自建全局 ErrorBoundary + 前端错误上报接口 |

**硬规则(写代码前必读)**

1. 安装任何原生依赖只能走 `pnpm dlx expo-harmony-cli install <pkg>`;`pnpm add` / `expo install` / `expo prebuild` 会破坏 patch、alias、原生注册与 `.expo-harmony/managed-state.json`。
2. 新引原生包必须审计 `grep -l "requireNativeModule" node_modules/<pkg>/build/*.js` —— Expo 包在 JS 顶层 `requireNativeModule('ExpoXXX')` 就会在鸿蒙崩,和有没有调用 API 无关。
3. 版本优先级 `@react-native-ohos/*`(RN 0.77)> `@react-native-oh-tpl/*`(RN 0.72,多数已弃用)。
4. `react-native-svg` 已整体 alias 到 `@react-native-ohos/react-native-svg`,不要再加与 SVG transformer 冲突的库。
5. UI 需要平台分叉时用 `.harmony.tsx` 文件后缀(`RN_BUNDLE_PLATFORM=harmony` 生效),零 Metro 配置。

### 2.3 架构总览

```
                    ┌──────────────────────────────┐
                    │  guard-find-app              │
                    │  iOS / Android / HarmonyOS   │
                    └───────┬──────────────┬───────┘
                  HTTPS/JSON │              │ WebSocket ?token=
                            ▼              ▼
      ┌──────────────────────────────────────────────────────┐
      │ guard-find-server  (com.guardfind.server, :8082 dev) │
      │  /app/**  C端接口   /system/** 管理接口   /open/**     │
      │  ┌────────────┬─────────────┬──────────────┐         │
      │  │ app 域     │ findcase 域 │ message 域   │         │
      │  │ (Sa-Token  │ (档案/线索/ │ (站内信/     │         │
      │  │  app 登录域)│  广场/举报) │  WS 广播)    │         │
      │  └────────────┴─────────────┴──────────────┘         │
      │  collect 域:采集框架(可插拔 Source) + 调度任务       │
      └───────┬──────────────┬───────────────┬───────────────┘
              ▼              ▼               ▼
        MySQL          Redis          对象存储/本地
     guard_find_server  DB:1        (gf 文件 bucket)
              ▲
              │  Hutool HttpRequest(限速 + robots + 重试)
      ┌───────┴─────────┐        ┌──────────────────────┐
      │ 公开寻亲站点     │        │ guard-find-backend   │
      │ rangaihuijia 等 │        │ + guard-find-frontend│
      └─────────────────┘        │  (审核候选/下架/统计) │
                                 └──────────────────────┘
```

采集数据**不直接对外可见**:先进候选表,由人工审核合并后才发布到广场。这是本规划书在数据侧最重要的设计约束(见 §4.12、§7.1)。

---

## 3. 功能模块总览

| 编号 | 模块 | 一句话职责 | 期次 |
| --- | --- | --- | --- |
| M1 | 账号与登录 | App 用户注册、验证码/密码登录、会话与注销、资料与实名 | 一期 |
| M2 | 寻人档案 | 结构化失踪/走失人员档案:字段、媒体、状态机、找回闭环 | 一期 |
| M3 | 信息发布 | 家属发布与编辑入口、草稿、提交审核、媒体上传 | 一期 |
| M4 | 广场 | 公共信息流:最新/就近/紧急/已找回,筛选与搜索 | 一期 |
| M5 | 消息与通知 | 站内信、审核/线索回执、WebSocket 实时、通知偏好 | 一期(站内+WS),二期(厂商推送) |
| M6 | 线索与联系 | 对某条档案提交线索、不暴露家属联系方式、线索回执 | 一期 |
| M7 | 内容治理 | 举报、敏感词/图片预检、审核队列、下架与删除权 | 一期接口,二期界面 |
| M8 | 数据采集 | 可插拔采集框架、宝贝回家适配器、候选-合并-发布 | 一期 |
| M9 | 搜索与发现 | 关键词 + 多维筛选 + 拼音/别名,搜索历史与热词 | 一期基础 |
| M10 | 个人中心 | 我的发布、我的线索、收藏、设置、关于、注销 | 一期 |
| M11 | 社区 | 圈子/帖子/评论/点赞/关注、经验与志愿招募 | 二期 |
| M12 | 地图与就近 | 地理编码、半径/行政区检索、扩散热力 | 二期 |
| M13 | 紧急扩散与 SOS | 一键 SOS、时效广播、周边推送、联动警方 | 三期 |
| M14 | 智能比对 | 人脸检索、跨年龄老化、线索图像比对 | 三期 |
| M15 | 系统与基础 | 字典、参数、文件、日志、限流、错误上报、国际化 | 一期打底 |

---

## 4. 模块详细设计

> 每个模块给出:职责边界 → 页面清单(App)/接口清单(Server)→ 数据模型 → 关键业务规则 → 边界与异常 → 验收标准。接口一律用统一响应包裹 `{code, msg, data}`(成功 `code=0`),分页用 `PageResp{records, total}`。

### M1 账号与登录

**职责**:建立与后台 RBAC 完全隔离的 C 端身份体系。**决策已定**:新建 `gf_app_user` 表 + 独立 Sa-Token 登录域(`StpAppUtil`,loginType=`app`),不在 `sys_user` 上加 `user_type`。原因:`sys_user.dept_id NOT NULL` 且 `AbstractLoginHandler.checkUserStatus()` 会在部门停用时硬失败;`GET /auth/user/route` 把角色映射成后台菜单;两条链路混用会让 C 端用户有机会碰到 RBAC 边界。

**页面(App)**

| 页面 | 路由 | 要点 |
| --- | --- | --- |
| 启动判定 | `app/_layout.tsx` | 读 token → 有效则进 `(tabs)`,否则 `/welcome`;需 `GestureHandlerRootView` + `SafeAreaProvider` + ErrorBoundary(当前模板 `_layout.tsx` 缺,且第 15-18 行有个被 CLI 改坏的空 `useEffect`) |
| 欢迎/登录 | `app/auth/login.tsx` | 手机号 + 验证码为主;密码登录折叠在"其他方式";图形验证码按需出现 |
| 注册即登录 | 同上 | 验证码校验通过且手机号未注册时**自动建档**,不单独设注册页(减少流失) |
| 资料完善 | `app/me/profile-edit.tsx` | 昵称、头像、身份(家属/志愿者/好心人)、常驻地 |
| 账号安全 | `app/me/security.tsx` | 改密、换绑手机、注销(冷静期 15 日)、退出登录 |

**接口(Server,`/app/auth/**`)**

| 方法 | 路径 | 鉴权 | 说明 |
| --- | --- | --- | --- |
| POST | `/app/auth/sms-code` | 公开 | `phone` + `scene`(LOGIN/RESET/REGISTER);Redis `CAPTCHA:app:{phone}`,6 位,5 分钟;叠加 `@RateLimiter`:同机 1/分、同号 1/分、同号 10/日 |
| POST | `/app/auth/login` | 公开 | `{authType: PHONE \| ACCOUNT, ...}`;PHONE 校验验证码 → 未注册则自动建档;返回 `{token, tokenName, expiresIn, isNewUser, user}` |
| POST | `/app/auth/password-login` | 公开 | `account + password`,BCrypt 校验 + 失败 5 次锁 15 分钟 |
| POST | `/app/auth/logout` | 登录 | 当前 token 注销;`is_concurrent=false` 时踢同端旧会话 |
| GET | `/app/auth/me` | 登录 | 当前用户资料 + 角色标记 + 未读数 |
| POST | `/app/auth/password` | 登录 | 设置/修改密码(需旧密码或验证码) |
| POST | `/app/auth/reset-password` | 公开 | 验证码 + 新密码 |
| POST | `/app/auth/deactivate` | 登录 | 申请注销,进入冷静期 |

**数据模型**

`gf_app_user`:`id, nickname, avatar, gender, phone(加密), password(可空, BCRYPT), account_status(NORMAL/LOCKED/CANCELLED), identity_type(GUARDIAN/VOLUNTEER/PUBLIC), real_name_status(UNCHECKED/CHECKING/PASSED/REJECTED), province/city/district_code, volunteer_verified, registration_source(APP/COLLECT_IMPORT), last_active_time, deactivate_apply_time` + 审计列。唯一索引 `uk_phone(phone, deleted)`。

`gf_app_device`:`id, user_id, platform(IOS/ANDROID/HARMONY), device_id, push_token(nullable 一期), app_version, os_version, bind_time` —— 一期只落库,二期推送直接可用。

**业务规则**

1. 手机号是唯一自然人锚点,必须加密存储(`continew-starter.encrypt.field` 已具备 AES 字段加密能力,和 `sys_user.phone` 同机制)。
2. 自动建档的昵称默认 `无脚星友 + 4 位随机`,允许立刻改;头像为空时用文字头像组件,不留空。
3. token 策略取自 `sys_client` 新增的 `APP` 客户端行(`auth_type=["PHONE","ACCOUNT"]`、`timeout` 30 天、`is_concurrent=true`、`max_login_count=3`),不硬编码。
4. 未登录可浏览广场与档案详情;**任何写操作、线索提交、收藏**都需登录。
5. 未成年人不得使用家属身份发布(见 §7.1)。

**验收**:三端完成验证码登录与自动建档;token 重启后仍在(secure store / harmony 存储);同账号 3 台设备并发策略正确;短信接口在压测下限流生效且不重复计费。

---

### M2 寻人档案(核心域)

**职责**:全产品的中心数据结构 —— 一条"人在找"的记录。**档案与帖子(M11)不同**:档案有强结构字段、状态机、时效性和找回闭环。

**字段设计(结构化优先,便于检索与比对)**

| 分组 | 字段 | 约束 |
| --- | --- | --- |
| 身份 | 姓名/小名/别名、性别、出生日期、失踪时年龄、近照 | 别名支持多个(口音/乳名对识别很关键) |
| 时空 | 失踪时间、失踪地点(省/市/区 + 详细地址 + 可选经纬度)、户籍地、走失时行动方向 | 失踪时间不得晚于当前时间;地点至少到区 |
| 体貌 | 失踪时身高(cm)、体型、发型、口音、疤痕/胎记、衣着、随身物品 | 至少填体貌描述一项才可提交审核 |
| 情境 | 失踪情形(拐卖/走失/被拐疑似/失联/走失老人)、经过描述、家庭背景与线索资料、其他说明 | 描述 10–2000 字 |
| 关联 | 报警情况(是否立案/派出所/警官联系方式)、血样采集情况(打拐 DNA 库是硬指标)、跟进志愿者 | 血样字段用字典,不自由填 |
| 状态 | `status`、`urgency_level`、时效、来源、审核意见 | 见状态机 |

**状态机**

```
DRAFT ──提交──▶ PENDING_REVIEW ──通过──▶ PUBLISHED ──家属/民警确认──▶ FOUND(已找回)
   ▲                  │  │                                          └─▶ FOUND_SUSPECT(疑似待核实)
   │              驳回│  │撤回(家属)                                      │核实不通过
   └──────────────────┘  └────────▶ WITHDRAWN ◀──超时自动降级──┘            ▼
                       │                                              PUBLISHED
                   涉法/重复 ──▶ BLOCKED ──申诉──▶ PENDING_REVIEW
   PUBLISHED ──举报达阈值/管理员──▶ OFFLINE(下架,保留数据,不对外)
   任何状态 ──权利人删除请求/注销──▶ ERASED(物理清除个人识别字段,保留匿名审计)
```

- `urgency_level`:NORMAL / URGENT(失踪 ≤ 72 小时自动标紧急,黄金 72 小时)/ STANDING(长期未破)。
- 时效:`PUBLISHED` 后 90 天无人确认 → 自动转 `ARCHIVED`(不再进广场首屏,可手动续期)。这是防止广场被陈年信息淹没的关键。
- `FOUND` 必须填"确认人 + 确认方式(公安/家属/DNA 比对)",且**保留原档案**做喜报展示,不删帖(避免"找到即删"造成线索断流)。

**接口(`/app/find/case/**`)**

| 方法 | 路径 | 鉴权 | 说明 |
| --- | --- | --- | --- |
| POST | `/app/find/case` | 登录 | 创建(草稿或直接提交审核);`dryRun=false` |
| PUT | `/app/find/case/{id}` | 本人 | 仅 `DRAFT/REJECTED` 可全量编辑;`PUBLISHED` 只能改补充信息 |
| POST | `/app/find/case/{id}/submit` | 本人 | 触发审核,写 `gf_case_version` 快照 |
| GET | `/app/find/case/{id}` | 公开 | 详情(公开视图,已脱敏:不返回家属手机号) |
| GET | `/app/find/case/{id}/full` | 本人/审核 | 含联系方式的完整视图,记录访问日志 |
| POST | `/app/find/case/{id}/found` | 本人 | 标记找回 + 确认信息 |
| POST | `/app/find/case/{id}/renew` | 本人 | 续期,重置时效 |
| POST | `/app/find/case/{id}/withdraw` | 本人 | 撤回 |
| GET | `/app/find/case/mine` | 登录 | 我的档案 + 各状态计数 |
| POST | `/app/file/upload` | 登录 | 复用现成分片上传,业务上限制图片 ≤ 9 张、单张 ≤ 5MB |

**数据模型**

`gf_find_case`(主表,列见上文字段设计)、`gf_find_case_media`(`case_id, url, storage_id, type(PHOTO/VIDEO), sort, thumb_url, sha256`)、`gf_find_case_alias`(`case_id, alias, kind(小名/口音叫法)`)、`gf_case_version`(`case_id, seq, snapshot_json, reason, operator`)—— 审核留痕与"信息被改过"的可追溯性、`gf_case_status_log`。

关键索引:`idx_status_urgency_missing_time(status, urgency_level, missing_time)`、`idx_region(province_code, city_code)`、`idx_create_time`、`uk_source(source_type, source_id, deleted)`。

**业务规则**

1. 一个自然人只应有一条活跃档案:提交时按"姓名 + 性别 + 失踪日期 ±7 天 + 区划"做**疑似重复检测**,命中则要求确认或走认领,不硬拦。
2. 家属手机号/微信号只在 `/full` 视图与线索回复中出现,公开详情只显示"通过 App 提交线索"。这是防骚扰和防诈骗的关键。
3. 所有状态变更写 `gf_case_status_log`,并给相关人发站内信(M5)。
4. 档案图片本地化到对象存储,不长期存外链(外链会失效且会暴露来源站点)。

**验收**:能完成"发布 → 审核 → 广场可见 → 收线索 → 标记找回"全链路;驳回可见原因并可修改重提;紧急档案在广场首屏排序正确;90 天自动降级有测试覆盖。

---

### M3 信息发布

**职责**:把 M2 的建档动作做成低摩擦的表单流程。**这一层是"发布体验",不是新数据结构。**

**页面**:`app/publish/index.tsx`(选类型:寻人/喜报/提供协助)、`app/publish/case.tsx`(分步表单)、`app/publish/preview.tsx`(发布前预览 + 合规告知勾选)、`app/publish/drafts.tsx`(草稿箱)。

**分步表单(降低流失,允许断点续填)**

1. 基本信息:与谁的关系、姓名/别名、性别、出生日期、近照(先给 1 张就能进下一步)。
2. 时空信息:失踪时间(日期时间选择器)、失踪地点(省市区三级 + 详细地址)、户籍地。
3. 体貌特征:身高、体型、口音、疤痕胎记、衣着、随身物品、特征描述。
4. 情境与联络:失踪情形、经过描述、报警与血样采集情况、可公开的联络方式(单选:仅 App 内 / 手机号)。
5. 确认发布:合规告知页(确认已获得权利人同意、承诺信息真实、知晓虚假信息责任)+ 提交。

**接口**:`POST /app/find/case/draft`(存草稿)、`PUT /app/find/case/draft/{id}`、`GET /app/find/case/drafts`、`POST /app/find/case/{id}/submit`;草稿合并进主表用 `status=DRAFT`,不另建表(减少状态同步问题)。

**业务规则**:草稿 30 天未提交自动清理并提示;每用户每日提交审核上限 5 条(防灌);图片上传前客户端压缩到长边 1600px(三端都用 `Image` 手工 scale,不依赖 `expo-image`);离线时草稿本地暂存(`lib/storage`)并在联网后同步。

**验收**:5 步表单在中端机上单字段失焦不重新挂载整页;断网能保存;从草稿到提交审核全程无重复填写;必填校验错误提示定位到具体字段。

---

### M4 广场(公共信息流)

**职责**:无门槛可浏览的公共入口,承担"就近扩散"和"喜报正反馈"。

**页面与 Tab**:`app/(tabs)/plaza.tsx`

| 流 | 排序/筛选 | 说明 |
| --- | --- | --- |
| 最新 | `PUBLISHED` 按 `missing_time` 与发布时间加权 | 默认流 |
| 紧急 | `urgency_level=URGENT` | 首屏红色视觉带倒计时("已失踪 XX 小时") |
| 就近 | 用户所选常驻地 / 手动切换城市 | 一期用**行政区**(无地图),经纬度字段已预留 |
| 本地志愿 | 区划 + 志愿者数 | 一期给静态入口占位 |
| 喜报 | `status=FOUND` | 找回案例展示,是留存和传播的关键 |

筛选维度:性别、年龄段、失踪年份区间、情形(拐/走失/失联)、是否有 DNA 血样。搜索:姓名/别名/地点关键词(M9)。

**接口**

| 方法 | 路径 | 鉴权 | 说明 |
| --- | --- | --- | --- |
| GET | `/app/plaza/feed` | 公开 | 参数 `sort(latest/urgent/nearby/found)`、`provinceCode/cityCode`、`keyword`、`page/size`;返回卡片视图 |
| GET | `/app/plaza/filters` | 公开 | 字典项(区划树、情形、年龄段)用于筛选器 |
| GET | `/app/plaza/statistics` | 公开 | 累计建档 / 已找回 / 近 30 日新增,首页信任背书 |
| POST | `/app/plaza/track` | 登录可选 | 曝光与点击埋点,给推荐和运营 |
| GET | `/app/find/case/{id}` | 公开 | 卡片点击进详情(复用 M2) |

**业务规则**:`ARCHIVED`、`OFFLINE`、`BLOCKED`、`ERASED` 一律不进任何流;`FOUND` 不混在最新流里(单列喜报),避免"已解决"占掉黄金位;卡片渲染必须脱敏(姓名 + 年龄 + 区划到市 + 失踪年月,不显示详细地址);下拉刷新 + 游标分页(不用 offset,深翻会漂)。

**验收**:空态/加载失败态齐备;首屏 P95 < 1.5s(4G);切城市后列表与筛选器一致;已下架档案在详情直链访问返回 404 且带说明。

---

### M5 消息与通知

**职责**:让"有人提供线索""审核通过了"这类事件真的被看到。**一期只做站内信 + WebSocket + 前台轮询,厂商推送二期**(鸿蒙侧无已验证的 Expo 推送适配,一期不承诺离线推送)。

**事件目录(一期必发)**

| 事件 | 收件人 | 渠道 | 落库 |
| --- | --- | --- | --- |
| 审核通过 / 驳回 | 发布人 | 站内信 + WS | ✅ |
| 新线索提交 | 档案所有人 | 站内信 + WS + 短信(家属可选订阅) | ✅ |
| 线索被回复 / 被标记无效 | 线索提交人 | 站内信 | ✅ |
| 档案被确认找回 | 所有关注/提交线索者 | 站内信 | ✅ |
| 临近时效降级(7 日前) | 档案所有人 | 站内信 | ✅ |
| 举报处理结果 | 举报人 / 被举报人 | 站内信 | ✅ |
| 系统公告 / 政策 | 全体 | 复用 `sys_notice` 广播 | 复用现成 |

**接口**:`GET /app/message/unread`(角标)、`GET /app/message`(分页,支持 `onlyUnread`)、`PATCH /app/message/{id}/read`、`PATCH /app/message/read-all`、`GET /app/notification/preference`、`PUT /app/notification/preference`(站内/短信订阅开关)、WebSocket `ws(s)://host/websocket?token={appToken}`。

**关键实现约束(需先验证再写)**:现有 `WebSocketUtils.sendMessage(token, payload)` 与 `MessageServiceImpl` 是按后台 Sa-Token 登录标识寻址的,而 M1 用的是 `app` 独立登录域。**实施第一步必须确认 WS 寻址能否承载 app 域 token**;若不能,则新增按 `user_id` 的 topic 注册表(`gf_ws_session`),App 侧连接后发送 `subscribe` 帧。这是本模块最大的技术不确定性,不能假设现成可用。

**数据模型**:新增 `gf_message`(`receiver_id, type, title, content, biz_type(FIND_CASE/LEAD/AUDIT/SYSTEM), biz_id, path, read_status, read_time`)与 `gf_notification_preference`。不复用 `sys_message`:它是后台广播语义 + 租户隔离,且收件人指向 `sys_user`。

**验收**:App 在前台时线索通知到达延迟 < 3s(WS);WS 断开时轮询兜底(30s,进后台降频);未读角标与服务端一致;关闭某类订阅后确实不发。

---

### M6 线索与联系

**职责**:把"我好像看到了这个人"变成**可核实的结构化线索**,同时不暴露双方联系方式。

**页面**:`app/case/{id}/lead.tsx`(线索提交:时间、地点、描述、图片、我的可回拨方式)→ `app/case/{id}/leads.tsx`(家属视角线索列表,可标"有价值/无效/已联系")。

**接口**:`POST /app/find/case/{id}/lead`、`GET /app/find/case/{id}/leads`(仅本人/志愿者)、`PATCH /app/lead/{id}/status`(ACCEPTED/REJECTED/CONTACTED)、`POST /app/lead/{id}/reply`。

**规则**:提交线索必登录且绑手机(防恶意灌);同IP 同档案 1 小时限 3 条;线索描述走敏感词 + 图片预检(M7);家属**不公开手机号**,沟通经 App 内回复,家属可主动选择公开一个联系方式;疑似诈骗话术("先转钱""加微信验证")命中词库即标高风险并提示审核员。

**验收**:线索从提交到家属可见 < 5s;家属可标注状态并回复,提交人收到站内信;非法参数(空描述、纯图)被拒。

---

### M7 内容治理与审核

**职责**:寻亲信息的真实性与合法性是产品存亡线。一期提供接口与数据模型,审核队列 UI 在 `guard-find-frontend` 二期做。

**能力**:文本敏感词/辱骂/诈骗话术(DFA 词库,`gf_sensitive_word`)、图片复用 `continew-starter` 已有内容检测扩展点(无则先接人工队列)、举报(`gf_report`:档案/线索/用户,理由枚举 + 证据图)、处置(`OFFLINE/BLOCKED` + 用户禁言/封禁)、**删除权**:`POST /app/find/case/{id}/erase`(权利人请求或账号注销触发,物理清除识别字段,`ERASED` 状态保留匿名审计)。

**接口(`/app/moderation/**`,登录 + 角色)** 与后台 `/system/moderation/**`:`GET /pending`(审核队列)、`POST /{id}/approve`、`POST /{id}/reject{reason}`、`GET /reports`、`POST /reports/{id}/handle`。

**规则**:发布与采集候选**都要过人审**才能对外;驳回理由必须来自预设模板(可附自由文本),保证体验一致和可统计;举报达 3 次自动降权进队列顶部;同一档案被下架后再次发布需更高优先级人工确认。

**验收**:审核动作产生状态日志 + 站内信 + 广场消失;举报到处置闭环;erase 后任何流与详情直链均不可见。

---

### M8 数据采集与整合(爬虫)

**职责**:把公开站点的历史寻亲档案收敛进自己的库,成为候选 → 审核 → 发布。**这是全规划书合规风险最高的模块,约束按硬性处理。**

**已验证的事实(2026-09-24 实测)**

- 站点 `https://www.rangaihuijia.com/` 的 `robots.txt` 内容为 `User-agent: *` / `Allow: /`(全站放行)。
- 列表 `GET /xunren/category/{categoryId}?p-page={N}`,分页带首页/上下页/尾页与"当前区间/总数"计数。
- 详情 `GET /xunren/view/{numericId}`。
- 卡片字段:姓名、性别、失踪地点、失踪日期、出生日期、籍贯;详情字段:姓名、类别、编号、性别、生日、年龄、出生地(籍贯)、失踪时身高、失踪日期、失踪地址、特征描述、发生经过、家庭背景及线索资料、其他说明、添加日期、寻亲联系人。
- 图片:`/upload/image/[YYYYMMDD]/{hash}.jpg` 原图,`/upload/thumb/image/.../{hash}_{W}x{H}.jpg` 缩略图。
- 注:`www.baobeihuijia.com/robots.txt` 实测返回 **521**(源站不可达或 CDN 拦截)。因此一期适配器以 `rangaihuijia.com` 结构为准,并把"域名不可达 → 记录失败并跳过"作为常规路径处理。

**框架抽象(换站点不改主流程)**

```
gf_collect_source (源配置: code/name/baseUrl/robots 缓存/enabled/rate limit/cron/last_cursor)
        │
CollectJob (NoticePublishJob 式双模式: snail-job.enabled=false → @Scheduled;否则 @JobExecutor)
        │
CollectOrchestrator ──▶ CollectSource 接口 (list(page)/detail(id)/map(RawPage)→Candidate)
                            ├─ RangAiHuiJiaSource   (一期实现)
                            └─ LocalFixtureSource   (一期实现,离线样例,供测试与演示)
        │  RobotsPolicy(必查) → RateLimiter(域名级 QPS + Crawl-delay) → HttpFetcher(Hutool HttpRequest, 重试/退避)
        ▼
gf_collect_raw (原始快照: source_code+source_id 唯一, html/etext/sha256/fetch_time, 保留来源 URL)
        ▼  解析 + 字段映射 + 归一化(姓名/日期/区划/性别枚举) + 指纹去重
gf_collect_candidate (候选: 规范化字段 + dedupe_key + status)
        ▼  人工审核(M7) + 与 gf_find_case 合并(claim/merge)
gf_find_case (status=PUBLISHED, source_type=COLLECT, source_id, source_url)
```

**接口**

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/system/collect/source` | 源配置 CRUD(`@CrudRequestMapping("/system/collect/source")`) |
| POST | `/system/collect/run/{sourceCode}` | 手动触发一轮采集(带 `maxPages`,默认 5,防误触发全量) |
| GET | `/system/collect/log` | 采集日志分页:批次、页数、抓到/新增/重复/失败计数、错误摘要 |
| GET | `/system/collect/candidate` | 候选列表(按状态/来源/区划筛选) |
| POST | `/system/collect/candidate/{id}/approve` | 审核通过 → 写 `gf_find_case`,幂等按 `dedupe_key` |
| POST | `/system/collect/candidate/{id}/merge` | 与已有档案合并(保留来源链) |
| POST | `/system/collect/candidate/{id}/reject` | 驳回/丢弃 |
| DELETE | `/system/collect/candidate/{id}` | 删除,并写 `gf_collect_erase_log`(响应权利人删除请求) |

**合规硬约束(代码级实现,不是文档口号)**

1. **robots 必查**:每轮开始读 `robots.txt`,Redis 缓存 24h;解析失败或明确 Disallow 命中当前路径 → **中止本任务并告警**,不静默绕过。
2. **限速**:域名级串行,**默认 Crawl-delay 2s 且不低于站点声明值**;单轮上限 `maxPages`(默认 5);并发固定 1(采集不追求快)。
3. **只采公开页**,不做登录绕过、不做验证码识别、不破解接口签名、不采非公开的联系方式与身份信息;发现详情有需登录内容即跳过并记录。
4. **最小化与留痕**:只存寻亲必要字段;每条候选必须保留 `source_url` + `fetch_time` + `raw_sha256`,可追溯"这条数据从哪个公开页面、什么时候来的"(既是审计也是权利人删除的依据)。
5. **删除与撤回**:站点方或当事人要求下架时,按 `source_url`/`source_id` 批量置 `ERASED`,并写入 `gf_collect_erase_log`(审计留存)。
6. **用途限定 + 授权开关**:配置项 `collect.sources.{code}.enabled` 默认 **false**;上线前需站点方书面授权或法务确认。不将采集数据用于对外转售、模型训练或任何与寻亲无关的用途。
7. 图片本地化到自有存储,避免长期热链对方站点(既降低对方负载,也避免外链失效)。
8. UA 明示:`GuardFindCollector/1.0 (+https://…/contact)` + 联系邮箱,便于站方找到我们。

**去重规则**:`dedupe_key = SHA256(姓名规范化 + 性别 + 失踪日期(精度到日,空则留空) + 失踪地区划码到市)`。同 key 的候选只保留信息最全的一条为主候选,其余标 `DUPLICATED` 并链到主候选;与站内已有档案命中同样规则 → 提示 merge。

**测试策略**:采集解析不依赖外网 —— `LocalFixtureSource` 读取仓库内 HTML 样例(从真实结构构造的合成样例,含缺失日期、字段错位、分页边界、脏 HTML、404、超时等 case),`Jsoup` 解析路径与真实适配器共用同一套 `CandidateMapper`,用单测保证解析正确性;真实网络适配器另做集成测试(可 `@Disabled` / 显式 profile 触发),不放进 CI 默认路径。

**验收**:离线样例源单测覆盖解析、归一化、去重、幂等(同批次重跑 0 新增);robots 拒绝时能中止并告警(单测 + 手工验证);限速日志可见实际间隔 ≥ 配置值;候选审核通过后广场能查到,且详情带来源链接;`maxPages=5` 时不会全量拉。

---

### M9 搜索与发现

一期:关键词(姓名/别名/地点文本)+ 多维筛选,MySQL `FULLTEXT`(或 LIKE + 覆盖索引兜底,数据量 < 50 万时够用)。**不预先引入 Elasticsearch** —— Server 目前无 ES/Easy-ES 依赖,加索引组件的运维成本要单独评估(触发条件:档案 > 50 万 或 需要以图搜图)。

接口:`GET /app/search`(keyword + 筛选 + 分页)、`GET /app/search/hot`(热词)、`POST /app/search/history/clear`。规则:搜索词过敏感词与合规过滤;结果同样遵守状态过滤与脱敏。

### M10 个人中心

页面 `app/me/index.tsx`:我的发布(状态分组 + 续期入口)、我的线索、我的收藏、通知偏好、资料与安全、帮助与举报入口、关于(数据来源与免责声明、隐私政策、用户协议)、注销。接口沿用 M1/M2/M6,新增 `GET /app/me/statistics`。

### M11 社区(二期)

`gf_post`(求助帖/经验帖/志愿招募/喜报)+ 评论树 + 点赞 + 关注 + 圈子。**与档案双向关联**:帖子可 `attach` 档案卡(帖子流里挂档案卡片,给档案导流)。一期只在 `app/(tabs)/community.tsx` 留占位与说明,不写半成品。

### M12 地图与就近(二期)

地理编码(地址 → 经纬度,写 `gf_find_case.lng/lat`,一期已留列)、半径检索(`ST_Distance_Sphere`,MySQL 原生,无需 PostGIS)、按区划聚合热力、失踪点周边扩散。鸿蒙侧 `expo-location` 无已验证适配 → 用 `.harmony.tsx` 分派或直连 TurboModule,并准备"用户手选城市"的兜底(一期已具备)。

### M13 紧急扩散与 SOS(三期)、M14 智能比对(三期)

SOS:失踪 ≤ 72h 的一键升级扩散(周边强提醒 + 喜报式海报生成 + 警方接口对接),依赖厂商推送(M5 二期)先落地。智能比对:人脸检索与跨年龄老化,需先解决授权、算法备案与误识风险,**不在一期承诺范围内**。

### M15 系统与基础能力

复用现成:字典 `sys_dict`/`sys_dict_item`(情形、年龄段、血样状态、举报理由)、参数 `sys_option`、文件与分片上传、登录与操作日志 `sys_log`、限流 `@RateLimiter`、图形/行为验证码、全局响应包裹。新增:`lib/env.ts` 接口基址配置(禁止硬编码 host)、全局 ErrorBoundary、错误上报 `POST /app/client-log`、埋点 `gf_user_track`。

---

## 5. 数据模型总汇

一期新建表(前缀 `gf_`,统一带 `create_user/create_time/update_user/update_time/deleted bigint DEFAULT 0`,`ENGINE=InnoDB utf8mb4`,业务唯一键**必须包含 `deleted`**):

| 表 | 用途 | 关键索引 |
| --- | --- | --- |
| `gf_app_user` | C 端账号 | `uk_phone(phone,deleted)` |
| `gf_app_device` | 设备与二期 push token | `uk_user_device(user_id,device_id,deleted)` |
| `gf_find_case` | 寻人档案主表 | `idx_status_urgency_missing_time`、`idx_region`、`uk_source(source_type,source_id,deleted)` |
| `gf_find_case_media` / `gf_find_case_alias` | 图片 / 别名 | `idx_case` |
| `gf_case_version` / `gf_case_status_log` | 审核快照 / 状态轨迹 | `idx_case_seq` / `idx_case_time` |
| `gf_case_lead` / `gf_lead_reply` | 线索 / 回复 | `idx_case`、`idx_ip_time` |
| `gf_message` / `gf_notification_preference` | 站内信 / 偏好 | `idx_receiver_read` |
| `gf_report` / `gf_sensitive_word` / `gf_user_punishment` | 举报 / 词库 / 处置 | `idx_status` |
| `gf_collect_source` / `gf_collect_raw` / `gf_collect_candidate` / `gf_collect_log` / `gf_collect_erase_log` | 采集配置 / 原文快照 / 候选 / 批次日志 / 删除审计 | `uk_source_id`、`idx_dedupe_key`、`idx_batch` |
| `gf_favorite` / `gf_user_track` | 收藏 / 埋点 | `uk_user_target` |

二期:`gf_post`、`gf_post_comment`、`gf_post_like`、`gf_circle`、`gf_user_follow`、`gf_geo_area`。

**Liquibase 落地约定(必须遵守,否则启动即校验失败)**:已应用过的 changeset 不可修改(MD5 校验)。每个现有 SQL 文件恰好只有一个 `-- changeset <author>:1`,所以**新增表要新建文件**:首行 `-- liquibase formatted sql`,然后 `-- changeset <you>:N` / `-- comment …`,再在 `guard-find-server-api/src/main/resources/db/changelog/db.changelog-master.yaml` 追加 `include`(建表文件在数据文件之前)。同步写 `postgresql/` 镜像(include 保持注释,但文件要成对)。若表不参与租户隔离,需登记到 `continew-starter.tenant.ignore-tables`。

---

## 6. 接口与安全规范

| 项 | 约定 |
| --- | --- |
| 路径前缀 | C 端 `/app/**`,采集与审核后台 `/system/collect/**`、`/system/moderation/**`,三方开放 `/open/**` |
| 响应 | 控制器返回裸 DTO,由 `@EnableGlobalResponse` 包成 `{code,msg,data}`;成功 `code=0`;HTTP 状态保持 200(现框架约定) |
| 分页 | `PageResp{records,total,size,current}`;信息流用游标(见 M4) |
| 鉴权 | C 端 `Authorization: {app token}`(Sa-Token jwt,loginType=`app`);管理端沿用现有 `@SaCheckPermission`;公开接口 `@SaIgnore` **且**登记进 `sa-token.extension.security.excludes` |
| 包路径硬约束 | 新增代码必须在 `com.guardfind.server.<module>.{controller,service,service.impl,mapper,model.*}`;`.mapper` 与 `.model` 段名不可改(Mapper 扫描与类型别名 glob 依赖它们) |
| 参数校验 | jakarta `@NotBlank/@Length` + `@SpelValid` 条件校验,中文消息内联(项目未启用 i18n) |
| 敏感字段 | 手机号/身份证/详细地址用 `continew-starter.encrypt.field` AES 列加密;响应侧用 `security-mask` 脱敏注解 |
| 限流 | 验证码、发布、线索提交、搜索、举报全部挂 `@RateLimiter`,按 用户/IP/设备 三维度 |
| 审计 | 写操作类级 `@Log(module="…")`;高频只读方法 `@Log(ignore=true)` |

**安全红线**:越权(他人档案的 `/full`、线索、编辑)、批量导出个人信息的接口一律不存在且不开放;所有按 id 的读接口必须在服务层校验可见性(不能只在前端隐藏);`/app/**` 全部走 HTTPS + 证书校验,App 侧禁明文降级;后台管理接口不对公网暴露,保留租户与 RBAC。

---

## 7. 非功能、合规与风险

### 7.1 合规与隐私(最高优先级)

失踪儿童姓名与照片属**敏感个人信息**。落地要求:

1. **法律基础**:发布人须为监护人或权利人本人;M3 第 5 步强制勾选"已获得权利人同意且信息真实"并留 `agreement_version` + 时间戳 + 发布人身份;未成年人档案不得由未成年账号发布。
2. **最小必要**:公开视图不显示详细地址、家属手机号、身份证;`/full` 视图需权限并记访问日志。
3. **删除权与撤回**:`ERASED` 状态支持"清除识别字段但保留匿名统计",账号注销触发级联清除(冷静期 15 日后)。
4. **数据留存**:`gf_collect_raw` 原文快照保留 180 天后自动清理正文(保留哈希与来源 URL);日志类表按字典参数配置的周期清理。
5. **采集合规**:见 M8 八条硬约束 —— robots 必查、限速、不绕过授权、来源留痕、默认关闭需开关、用途限定、UA 明示、可批量删除。
6. **内容风险**:寻亲是诈骗高发场景。产品内**不做募捐、不跳转支付**;所有"捐款/加微信验证/交钱包邮费"类话术命中词库即标高风险;档案与线索页常驻防骗提示。
7. **不实信息**:恶意虚假发布需可追责 —— 强制手机号、发布行为全量入 `sys_log`、支持公安调证。
8. **算法与备案**:三期人脸/老化能力引入前须完成算法备案、单独同意与安全评估;一期不涉及。

### 7.2 性能与容量(一期目标)

`/app/plaza/feed` P95 < 800ms;档案详情 P95 < 500ms;单轮采集(5 页 + 详情)在 Crawl-delay 2s 下 < 90s;容量按 50 万档案 / 200 万候选估算,MySQL 单表可承载,预留冷热分离与 ES 触发条件。图片走对象存储 + CDN,客户端压缩到长边 1600px。

### 7.3 可用性与可观测

Server 已有 TLog 链路、`sys_log`、p6spy SQL 日志;新增:采集批次成功率与耗时直方图、审核队列积压告警、WS 在线连接数、C 端错误上报面板。降级:Redis 不可用时 token 校验失败必须拒绝而非放行;采集失败不影响主服务;搜索不可用时广场仍可翻页。

### 7.4 无障碍与多语言(规划要求,一期部分落地)

紧急/找回等状态不能仅靠红绿色区分(需文字 + 图标);点击区 ≥ 44×44;图片必须有 `alt`/无障碍标签;表单错误提示与字段关联。一期中文简体,`lib/i18n` 预留字典式文案 key,繁体与英文二期。

### 7.5 工程质量基线(现状缺口,须在实施前补齐)

| 缺口 | 现状 | 处理 |
| --- | --- | --- |
| App 无 `typecheck` 脚本 | 只有 `lint`(且仓库无 eslint 配置)、`test` 是 `jest --watchAll` 不可 CI | 加 `typecheck: tsc --noEmit`、`test:ci: jest --ci`,提交 jest 配置 |
| App 无 CI | 无 `.github/workflows` | 加 lint + typecheck + test 三段流水线 |
| App `tailwind.config.js` `plugins: []` | `skeleton.tsx` 用了 `animate-pulse` 但不会动,且未扫描 `lib/hooks/shims` | 补 `tailwindcss-animate`,扩 content glob |
| App 品牌仍是模板 | `scheme: myapp`、`com.example.guard_find_app`、tint 还是 Expo 蓝 | 定品牌 token,合并 `constants/Colors.ts`(hex)与 `global.css`(HSL)为单一来源 |
| Server 测试全局关闭 | 根 pom `surefire <skip>true</skip>`,CI 只 compile/package,唯一测试是需要活的 MySQL | 采集与业务服务层的单测用纯 JUnit + Mockito 切片,开 `-Dsurefire.skip=false` 的本地/CI 路径 |
| Server `spotless:apply` 在 compile 阶段 | 会改写新文件为上游 Apache 头 + P3C 格式 | 接受它(不绕过),提交前跑一次 `mvn compile` 让格式落定 |
| 鸿蒙工程未生成 | `harmony/` 目录不存在但 `pnpm codegen` 已引用其路径 | 首步执行 `pnpm dlx expo-harmony-cli prebuild --platform harmony` + `ohpm install`,并修 `app/_layout.tsx` 的空 `useEffect` |

---

## 8. 分期计划与里程碑

| 期次 | 范围 | 交付与验收 |
| --- | --- | --- |
| **一期(MVP,本规划书实施目标)** | M1 账号 · M2 档案 · M3 发布 · M4 广场 · M5 站内+WS 通知 · M6 线索 · M7 审核与举报接口 · M8 采集框架 + `rangaihuijia` 适配器 + 离线样例源 · M9 基础搜索 · M10 个人中心 · M15 基础设施与质量基线 | 端到端可用:三端起 App → 注册 → 发布 → 过审 → 广场可见 → 提交线索 → 家属收到通知并标记找回;采集任务能把离线样例与真实站点数据落成候选并过人审发布;后端编译通过、单测绿、App typecheck + jest 绿、接口与前端字段逐项核对 |
| **二期** | M11 社区 · M12 地图就近 · 厂商推送(FCM/APNs/Harmony Push Kit)· 收藏与关注 · 后台审核队列与治理界面 · 推荐排序 · 多语言 | 就近推送半径、鸿蒙推送实机验证、治理看板 |
| **三期** | M13 SOS 紧急扩散 · M14 人脸比对与跨年龄老化 · 警方/救助站机构协同 · 公益志愿体系 | 需前置完成备案、授权与安全评估 |

**一期实施顺序(依赖驱动,自下而上)**

1. 质量与工程基线:App 补 typecheck/jest/CI/品牌 token;Server 明确测试与格式策略。
2. OpenSpec 变更规格落地(`openspec/changes/…` 双仓),与本文档对齐后作为实施清单。
3. Server 数据层:`gf_*` 建表 changelog + master include,字典与区划数据,`APP` 客户端行。
4. Server M1:`StpAppUtil` 独立登录域 + 验证码 + 自动建档 + token 策略 —— **先验证 WS 能否按 app token 寻址**(M5 的前置)。
5. Server M2/M3:档案、状态机、媒体、版本与状态日志。
6. Server M4/M9:广场流、游标分页、筛选、搜索。
7. Server M6/M7:线索、举报、审核动作与通知联动。
8. Server M5:站内信 + WS 广播 + 事件目录接线。
9. Server M8:采集框架、robots/限速、离线样例源、`rangaihuijia` 适配器、候选审核与合并、调度任务。
10. App 基础设施:env、fetch 客户端与拦截器、`lib/storage`(含 harmony 分派)、状态管理、ErrorBoundary、补 `form/toast/sheet` 等 UI 原语。
11. App 页面:auth → plaza → case 详情 → publish 分步表单 → message → me → lead。
12. 测试与验证:后端单测 + 打包、App 用例 + typecheck、接口契约逐项核对、三端运行验证(鸿蒙需实机/模拟器,报告里必须区分"实机验证"与"静态检查")。

---

## 9. 风险登记

| 风险 | 影响 | 应对 |
| --- | --- | --- |
| 采集未获站方授权 | 法律与声誉 | 一期源默认 `enabled:false`,需书面授权/法务确认后才打开;只走 robots 允许路径并留痕 |
| 目标站点结构或可达性变化(`baobeihuijia` 已实测 521) | 采集中断 | 解析失败降级为"记录并跳过 + 告警";`Source` 可插拔 + 离线样例源保证链路常绿 |
| 鸿蒙原生能力缺口(定位/相机/推送) | 三端体验不一致 | `.harmony.tsx` 分派 + 城市手选兜底;推送退到站内+WS+轮询;不依赖 `expo-asset` |
| WebSocket 无法按 app 域 token 寻址 | M5 实时能力落空 | 第 4 步先验证;不通则改自建 `gf_ws_session` topic 订阅 |
| 借寻亲实施诈骗 | 用户受损、品牌崩塌 | 禁募捐与支付跳转、话术词库高风险标记、防骗常驻提示、可追责发布 |
| 敏感信息泄露(地址/手机号) | 合规事故 | 列加密 + 响应脱敏 + `/full` 权限与访问日志 + 无导出接口 |
| 广场被陈年信息淹没 | 有效信息不可见 | 90 天自动 `ARCHIVED` + 续期机制 + 紧急档加权 |
| ContiNew 上游演进导致合并困难 | 维护成本 | 业务代码全部落 `gf_*` 与新包路径,不改模板既有类语义,升级面收敛 |
| 一期范围再次膨胀 | 交付失焦 | 社区/地图/推送/比对明确划入二三期,一期不写半成品 |

---

## 10. 附录:实施时的逐条检查表

- [ ] 新增原生依赖用了 `pnpm dlx expo-harmony-cli install`,并检查 `requireNativeModule` 顶层调用。
- [ ] 远程图片一律 `react-native` `Image`,未使用 `expo-asset` / `Asset.downloadAsync`。
- [ ] 新表放在**新建**的 changelog 文件并 include,`deleted` 进了所有业务唯一索引,同步 `postgresql/` 镜像。
- [ ] 新代码包路径含 `.model` 与 `.mapper`,未改段名。
- [ ] 公开接口同时加 `@SaIgnore` 与 `security.excludes`,写接口有 `@Log`、`@RateLimiter`、参数校验与列级加密。
- [ ] 采集任务实现 robots 检查、域名串行限速、`maxPages` 上限、UA 明示、来源 URL 与抓取时间留痕。
- [ ] 状态机过滤覆盖所有对外读接口(服务端校验,不是前端隐藏)。
- [ ] `mvn -B -Dspotless.apply.skip=true -DskipTests package` 通过,新增单测通过。
- [ ] App `tsc --noEmit` 与 `jest --ci` 通过,接口路径/字段与后端 Controller 逐项核对。
