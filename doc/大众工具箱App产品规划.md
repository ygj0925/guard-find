# 大众工具箱 App 产品规划（转型规划书 v1）

> 2026-10 起项目由"天下无拐"寻人公益应用转型为**面向全体用户的生活工具箱 App**。
> 定位：一个 App 装下多个高频小工具——记账、AI 对话、论坛（圈子）、待办、天气等。
> 现有项目结构（父仓库 + app/server/backend/frontend 四个子仓库）**保持不变**，在现有
> 骨架上叠加新模块；寻人相关页面一期保留可访问，后续按数据情况决定是否归档。

## 一、市场分析结论（为什么这么设计）

对标随手记、鲨鱼记账、钱迹等头部记账产品的共识：

| 竞品共识 | 我们的做法 |
| --- | --- |
| 极简记账（鲨鱼"3 秒记账"） | 记一笔页：金额大输入 + 分类宫格单页完成，无多级跳转 |
| 分类体系是记账的骨架 | 内置 13 个收支分类，一期不做自定义分类（降低理解成本） |
| 趋势图表 / 收支汇总 | 周报、月报、年报三档 + 分类占比条 + 逐日趋势条形图 |
| 预算管理（随手记强项） | 二期做月度总预算 + 分类预算、超支提醒 |
| 云端同步 | 服务端持久化（gf_ledger_record），登录即同步 |
| 无广告纯净（钱迹定位） | 工具箱内不投放广告、不推荐理财 |

差异化：我们不做一个独立记账 App，而是把记账做成工具箱里的**第一个工具**，
用工具箱形态承载后续 AI 对话、论坛等轻功能，覆盖"工具型用户"。

## 二、模块地图与里程碑

### M1（本期已实现）：工具箱框架 + 记账
- 新增"工具"Tab → 工具箱宫格页
- 记账工具全量：记一笔 / 日账单 / 周月年报 / 分类统计 / 云端同步

### M2：AI 对话（已落地首版）
- 服务端 OpenAI 兼容代理转发（`guardfind.ai.*` 配置，Key 不落端上；Key 为空 = 测试模式占位回复）
- 一期非流式（POST 整包回），流式待 RN/鸿蒙侧 SSE 适配验证后放开
- 页面：会话列表（最近活跃倒序 + 新建 + 长按删除）、聊天页（气泡流 + 发送中占位 + 预设快捷语：翻译/写作/总结）
- 表：`gf_ai_conversation` / `gf_ai_message`（changeset 3/4），会话归属校验与记账同口径
- 二期后半：SSE 流式渲染、助手预设服务端化、会话置顶/重命名

### M3：论坛（圈子）
- 帖子发布/信息流/点赞评论/关注，服务端复用现有审核（moderation）能力
- 页面：圈子信息流、帖子详情、发布器、我的帖子
- 与现有"社区(占位)"Tab 合并落地

### M4+：工具箱扩充 backlog（按"实现成本 × 留存价值"排序）

**P0 · 快速可上（纯本地或轻服务端，两周内一个迭代一个）——已全部落地**
| 工具 | 理由 | 关键设计 |
| --- | --- | --- |
| 倒数日/纪念日 ✅ | 实现成本最低、打开率高 | `gf_tool_countdown` + `/app/tool/countdown`；daysLeft 服务端统一计算 |
| 待办清单 ✅ | 工具箱基本盘 | `gf_tool_todo` + `/app/tool/todo`；未完成在前排序服务端出；AI 拆解留到 M2 后半 |
| 单位换算 ✅ | 纯本地零接口 | `lib/units/convert` 纯函数（长度/重量/温度/数据/面积），温度走独立对数路径；汇率因需外部数据源降级 P1 |

**P1 · 中成本（需要第三方数据源或服务端逻辑）**
| 工具 | 理由 | 依赖 |
| --- | --- | --- |
| 天气 | 高频打开型工具，工具箱门面 | 天气 API（免费档即可）+ 定位降级为城市手选（与广场同模式） |
| 番茄钟/专注 | 学习/办公人群留存；与待办联动 | 纯本地计时 + 专注统计周报（复用记账报表的 View 图表方案） |
| 记账预算 | 记账的留存关键（随手记强项），补齐 M1 留的口子 | 月度总预算+分类预算、超支提醒（复用消息通道） |
| 密码生成器/安全笔记 | 纯本地 + 设备内加密（复用 lib/storage/token 体系） | 生成器零成本；笔记加密存储 |

**P1.5 · 纯本地纯函数类（零接口零表）—— 房贷/文本/随机已落地,二维码生成待定**
| 工具 | 理由 | 关键设计 |
| --- | --- | --- |
| 房贷/利息计算器 ✅ | 工具类 App 搜索词 Top 级刚需;等额本息/本金纯公式 | 纯函数 `lib/calc/mortgage.ts` + 单测;月供/总息/逐年还款表(View 拼图) |
| 文本工具箱 ✅ | 字数统计/时间戳互转/进制转换/JSON 格式化,程序员+普通用户通吃 | 每个子功能一个纯函数,复用单位换算页的骨架 |
| 随机决定器 ✅ | 转盘/抽签/掷骰子,轻娱乐+社交传播成本低 |
| 时钟秒表计时器 ✅ | 纯本地;真闹钟被鸿蒙通知验证阻塞 |
| 计算器 ✅ | 纯函数引擎,除零错误态,离线高频 | 纯本地;转盘动画用 Animated(已验证依赖) |
| 二维码生成 | 需新增 react-native-qrcode-svg 依赖,须走 expo-harmony-cli 流程,单独排期 | 仅生成(扫码需相机,鸿蒙侧未验证,归入 P2) |

**P1.5 协同类（搭现有数据底座的便车,差异化所在）—— AI 周报/拆解/点评已落地**
| 工具 | 理由 | 协同点 |
| --- | --- | --- |
| 习惯打卡 | 打卡类留存全工具箱最强(streak 视觉激励) | 一张 `gf_tool_habit` 表;月度网格视图;超 streak 走消息通道提醒 |
| AA 分账 ✅ | 聚餐/旅游分账高频痛点 | 结果一键写回记账(复用 ledger 的 create 接口);参与人本地存 |
| AI 周报 | 工具箱的"护城河"功能:本周记账+待办完成度+心情一句话 | 调用现有 AI 代理(新增一种 system prompt 即可);每周日晚推送 |
| 愿望清单 | 想买的东西攒着,买完转记账 | 表结构近似待办;「实现愿望」按钮直接生成一笔支出 |

**P2 · 观察后做（依赖重或同质化竞争激烈）**
| 工具 | 说明 |
| --- | --- |
| 日记/心情日历 | 结合 AI 做月度情绪总结（M2 能力复用），但需内容审核能力跟上 |
| 快递查询 | 第三方聚合 API 有成本，且频次不如前几项 |
| 账单导入（支付宝/微信 CSV） | 记账深度用户才需要，放记账工具成熟后 |
| OCR 截图记账 | 依赖 OCR 服务；先用 AI 对话的视觉模型试点（上传截图→提取金额） |
| 二维码扫描 | 依赖相机权限,鸿蒙侧 expo-camera 未验证;生成已上,扫描随后 |
| 农历/黄历 | 需引农历换算库(纯算法可选),受众偏传统场景 |
| 指南针/水平仪 | 需磁力计/陀螺仪传感器(expo-sensors),走 expo-harmony-cli 且 RNOH 未验证 |
| 真闹钟/打卡提醒 | 需系统通知后台触发(expo-notifications),鸿蒙侧未验证 |

原则：每个新工具都按 M1 记账的标准落地——服务端领域包 + Liquibase + 归属校验 + 单测，
客户端走工具箱宫格接入，禁止绕过能力分派层。

## 三、页面设计规格

### 3.1 信息架构（Tab 层）
```
广场(寻人,保留) | 发布 | 社区(M3 落地论坛) | 工具(M1 新增) | 消息 | 我的
```
工具 Tab 使用宫格布局，每个工具一个入口卡片（图标 + 名称 + 一句话描述），
未上线工具显示"即将上线"角标，点击弹 Toast。

### 3.2 工具箱页 `app/(tabs)/tools.tsx`
- 顶部：问候语 + 今日概要（今日支出/收入，来自 ledger summary）
- 宫格区：已上线工具（记账）大卡片；规划中工具小卡片置灰
- 布局：`SafeAreaView(top)` + `ScrollView`，`paddingBottom: 104`（悬浮导航适配）

### 3.3 记账 · 日账单 `app/tools/ledger.tsx`
- 头部卡片：当月支出 / 收入合计 + 月份切换
- 列表按天分组：日期头（几月几日 + 星期 + 当日小计），组内每笔：分类图标 + 分类名 +
  备注 + 金额（支出主色外红/收绿，用语义 token）
- 右下角悬浮"记一笔"按钮 → `app/tools/ledger/add`
- 分页加载：`useFeed` 游标模式，服务端按 `record_date desc, id desc` 排序

### 3.4 记账 · 记一笔 `app/tools/ledger/add.tsx`
- 收/支分段切换（Segmented）
- 金额：大字号数字键盘输入（TextInput 数字键盘，两位小数校验）
- 分类宫格：9 宫格（支出）/ 4 宫格（收入），单选高亮
- 备注 + 日期选择（默认今天，改期用原生日历弹出）
- 提交后返回账单页并刷新；编辑模式复用本页（路由参数带 id）

### 3.5 记账 · 报表 `app/tools/ledger/report.tsx`
- 周期切换：周 / 月 / 年（对应服务端 period=week|month|year）
- 汇总卡：总收入、总支出、结余
- 分类占比：支出分类横向条形图（纯 RN View 宽度百分比实现，不引图表库——
  鸿蒙侧依赖需走 expo-harmony-cli 审批，一期用原生组件拼图）
- 趋势图：逐日（周/月）或逐月（年）支出条形图，同一纯 View 方案
- 明细入口：点击分类条 → 按分类过滤的账单列表（二期）

### 3.6 二期页面（设计先行，本期不实现）
- AI 对话：会话列表（最近会话 + 新建）→ 聊天页（顶部模型名、气泡左右分侧、
  流式打字机效果、长按复制/重发），底部输入条 + 发送按钮
- 论坛：信息流（双列瀑布流卡片：封面/标题/作者/点赞数）→ 帖子详情（正文 +
  评论楼层 + 底部评论输入条）→ 发布器（标题/正文/板块选择/图片）

## 四、服务端设计（guard-find-server）

### 4.1 表结构（M1 已实现）
```sql
gf_ledger_record (
  id           bigint AUTO_INCREMENT,
  user_id      bigint       NOT NULL COMMENT '所属用户',
  type         tinyint      NOT NULL COMMENT '类型（1：支出；2：收入）',
  amount       decimal(12,2) NOT NULL COMMENT '金额（元）',
  category     varchar(20)  NOT NULL COMMENT '分类码（见契约）',
  record_date  date         NOT NULL COMMENT '记账日期',
  remark       varchar(200) NULL,
  create_user/create_time/update_user/update_time/deleted  -- BaseDO 审计列
  KEY idx_user_date (user_id, record_date)
)
```

### 4.2 API（C 端 `/app/ledger/**`，全部需登录）
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/app/ledger/record` | 新增记录 {type, amount, category, recordDate, remark} |
| PUT | `/app/ledger/record/{id}` | 修改（校验归属） |
| DELETE | `/app/ledger/record/{id}` | 删除（校验归属） |
| GET | `/app/ledger/record/page` | 分页 `?month=yyyy-MM&page=&size=`，按日期倒序 |
| GET | `/app/ledger/summary` | `?period=week\|month\|year&date=yyyy-MM-dd` → 汇总+分类占比+趋势 |

汇总计算在 Java 侧纯函数 `LedgerSummaryCalculator` 完成（区间内记录聚合），
一期数据量下无需 SQL 聚合，便于单测。

### 4.3 二期表设计（AI 对话 / 论坛）
```sql
gf_ai_conversation(id, user_id, title, model, create_*, deleted)
gf_ai_message(id, conversation_id, role ENUM('user','assistant'), content TEXT, tokens, create_*)
gf_forum_post(id, user_id, board, title, content, cover_url, like_count, comment_count, status, create_*, deleted)
gf_forum_comment(id, post_id, user_id, parent_id, content, like_count, create_*, deleted)
```

## 五、客户端契约（guard-find-app）

- 端点定义：`lib/api/endpoints/ledger.ts`（分类码为契约常量，前端不硬编码文案映射表——
  文案映射放 `lib/ledger/categories.ts`，与后端枚举一一对应）
- 纯逻辑：`lib/ledger/stats.ts`（按天分组、分类占比、趋势补零）+ `__tests__` 单测
- 状态：不走 zustand，直接 `useRequest` + 路由返回后 `refresh`（数据单一来源在服务端）

## 六、验收清单（M1）
- [x] 记一笔：支出/收入、两位小数金额、分类、日期、备注
- [x] 日账单按天分组 + 日/月小计，分页加载
- [x] 周报/月报/年报：汇总、分类占比、趋势
- [x] 编辑/删除自己的记录，越权访问返回 404 语义
- [x] 未登录访问记账页跳登录
- [x] 端上纯函数单测 + 服务端汇总单测全绿

## 七、家庭共享记账(M1.5,已落地)

- **组**:创建者=组长,自动生成 6 位邀请码;成员凭码加入(每组上限 12 人)
- **账本切换**:记账页顶部 chip 切「个人账本 / 家庭组」,新账单记入当前作用域;报表同步按组聚合
- **权限口径**:每人只能改删自己记的账单;组长可移除成员/重置邀请码/解散(解散后账单自动回归各成员个人账本,数据不丢);组长不能退出(一期不做转让)
- **表**:gf_ledger_group / gf_ledger_group_member / gf_ledger_record.group_id(changeset 10-12)
- **隐私**:组内成员互相可见账单金额;不展示备注给非本人?一期全量可见(家庭场景默认信任),二期可加"仅自己可见"标记

## 八、留存与增长功能设计(产品经理视角)

**为什么留存的抓手在"关系 + 提醒 + 成就"三件事上:**
单机工具用完即走;把"数据积累(账单/打卡 streak)+ 社交关系(家庭组)+ 定期触达(周报/提醒)"
组合起来,才构成打开理由。

### 8.1 留存功能 backlog(按优先级)
| 功能 | 机制 | 依赖 | 优先级 |
| --- | --- | --- | --- |
| 连续记账徽章 | 记账连续 N 天给徽章(3/7/30/100 天),打卡 streak 同一套徽章体系 | 纯本地可做,徽章文案服务端下发 | P0 |
| 每周账单报告推送 | AI 周报已有;缺"周日晚 8 点自动生成+站内信触达" | 服务端定时任务(SnailJob 已接入)+ 消息通道 | P0 |
| 预算超支提醒 | 月度总预算+分类预算,超 80%/100% 时站内信 | gf_ledger_budget 表 + 消息通道 | P1 |
| 家庭组账单动态 | 成员记一笔时给其他成员发站内信("妈妈记了一笔 35 元") | 复用消息通道;需开关(通知偏好) | P1 |
| 月度回忆分享 | 报表生成分享图片(消费结构+金句) | 端上截图方案 or 服务端渲染,复杂度高 | P2 |
| 打卡提醒 | 每日固定时间提醒未打卡的习惯 | 系统通知(鸿蒙未验证)——被依赖阻塞 | P2 |

### 8.2 现有功能优化清单(体验债)
| 位置 | 问题 | 优化 | 成本 |
| --- | --- | --- | --- |
| 记一笔 | 系统数字键盘无"完成"键,收起麻烦 | 自定义金额键盘组件(大按键+完成键+±) | 中 |
| 记一笔 | 每次都要选分类 | 常用分类学习:按历史频次排前两位 | 低 |
| 日账单 | 无搜索/筛选 | 顶部加关键字搜索(备注/分类) | 低 |
| 记一笔 | 保存后必返回 | 提供"保存并再记一笔"(连续录入多笔场景) | 低 |
| 待办 | 无排序自定义 | 长按拖拽排序(复用工具箱置顶的偏好存储) | 中 |
| 工具箱 | 新用户不知道工具怎么用 | 每个工具首访浮层提示(一句话) | 低 |
| 报表 | 年报无 AI 点评 | 服务端补 year 口径的汇总 prompt | 低 |
| 全局 | 操作无触感反馈 | 打卡/记账成功加 Haptics(需鸿蒙验证) | 低 |

### 8.3 不做清单(明确拒绝,防止范围蔓延)
- 多账本(已有家庭组,概念已够多)
- 记账转账/报销拆分(家庭场景出现频率低)
- 自定义分类(内置 13 个已覆盖,自定义提高理解成本)
- Web 版工具箱

## 九、任务分派 + 目标 OKR lite(三期,已落地)

### 9.1 市场调研结论(飞书 OKR / Teambition / Worktile)
共识:① Objective 定方向、KR 必须可量化;② 任务 = 具象化的 KR,必须有负责人、截止日、优先级;
③ 进度全员可见(透明对齐);④ 定期复盘打分形成闭环。
参考:飞书 OKR 用户手册、Teambition 最佳实践(KR 拆解为任务看板)。

### 9.2 消费级轻量化适配(为什么不照搬企业版)
本 App 是家庭/个人工具箱:砍掉对齐树、评分矩阵、跨部门可见性——保留
「目标→KR 量化进度」「任务=负责人+截止+优先级+状态流转」两个最小闭环。

### 9.3 功能设计
**任务分派(挂家庭组)**:
- 组内任何成员可创建任务:标题 + 详情 + 负责人(仅限组成员) + 截止日 + 优先级(高/中/低)
- 状态流转:待办 → 进行中 → 已完成(负责人或创建者可推动)
- 两个视图:「指派给我的」「组内全部」;编辑/删除仅创建者
**目标 OKR lite(个人)**:
- 目标(Objective)+ 周期(2026Q4 或 2026 全年)+ 关键结果(KR:标题 + 目标值 + 当前值 + 单位)
- 进度条 = 当前值/目标值(封顶 100%);当前值 ≥ 目标值自动完成;支持手动标记完成
- 期末复盘打分、KR 关联任务、家庭共享目标 → 二期

### 9.4 合规与边界自查
- 权限:任务可见性=组内成员(与记账同口径,加入组即默认信任);KR/目标=仅本人
- 越权:非组成员创建/查看任务 → NOT_FOUND 同话术;任务编辑仅创建者
- 数据边界:不做考勤/绩效/薪资等敏感人力场景;OKR 数据默认私有
- 无障碍:所有操作按钮带 accessibilityLabel;状态不单靠颜色(带文字)
- 依赖:零新增原生依赖;未做任务提醒(鸿蒙通知未验证,与 P2 打卡提醒同批)

## 十、音乐播放器(已落地 Web 形态,鸿蒙待原生接入)

### 10.1 研究结论:这个功能的"隐藏的坑"(用户问的正是这些)
1. **RN 核心没有音频播放**——必须依赖原生模块(expo-av / expo-audio / react-native-sound);
   鸿蒙侧每个原生模块都要 `pnpm dlx expo-harmony-cli install` + RNOH 真机验证,这是最大的坑。
2. **选取手机音频文件**需要文档选择器(expo-document-picker),又一个原生依赖;
   且 Android SAF 授权的 URI 是临时的,重启 App 后可能失效,需要 takePersistableUriPermission。
3. **后台播放与锁屏控制**需要音频会话(Android MediaSession / 鸿蒙 AVSession),
   纯前端做不到,是"能播"与"像播放器"之间的差距,二期再做。
4. **音频格式**各端差异:iOS 偏好 AAC/CAF,Android/鸿蒙 AVPlayer 支持 mp3/aac/flac/ogg;
   用户手机里的无损格式(wav/flac)在部分端需要软解。
5. **Web 端可以零原生依赖完整实现**(HTML5 Audio)——这正是本期的落地形态。

### 10.2 本期落地形态(遵循能力分派层,与 image-picker 同模式)
- 能力:`lib/capabilities/music.ts`(Web=HTML5 Audio 完整可用)/ `music.harmony.ts`(鸿蒙 unsupported 降级)
- 页面:播放列表(本地持久化)+ 播放控制(上一首/播放暂停/下一首)+ 可点按进度条
- 鸿蒙接入路径已写在 capability 文件 TODO:装 expo-av → 替换 harmony 实现 → 调用方零改动
- 诚实标注:鸿蒙/安卓原生侧入口会显示"暂不支持"降级文案,而不是假可用

### 10.3 二期路线
expo-av 接入(播放+选文件)→ 后台播放 + 锁屏控制(AVSession)→ 均衡器/倍速 → 歌词显示

## 十一、在线签到(已落地)

### 11.1 产品决策:为什么用"动态码"而不是二维码
用户原始诉求是"生成二维码扫码签到",但技术+防作弊双重视角下改成了动态码:
- **扫码端**:相机扫描需 expo-camera(鸿蒙未验证);**生成端**:QR 编码需新增依赖(也要走 expo-harmony-cli)。两端都被原生依赖卡住。
- **防代签**:静态二维码能被截图转发给不在场的人代签——恰恰违背了这个功能的核心诉求。
- **动态码方案**(学校/会议系统同款):发起人屏幕展示 6 位码,每 30 秒轮换;拿到截图/旧码也过期。
  纯服务端校验,零权限依赖,比二维码更防代签。二维码在原生依赖验证后可作为"静态入口"(扫码进页面仍需输动态码)补充。

### 11.2 防代签四道闸(全部服务端校验)
1. **动态码**:FNV-1a(secret+时间窗) 6 位数字,30 秒轮换,容忍上一窗时钟偏移;
   算法在 Java/TS 两端同源实现,共享测试向量互验(漂移即 CI 失败)。
2. **时间窗**:窗口外拒签(未开始/已截止)。
3. **一人一次**:唯一约束兜底。
4. **地点名校验**:发起人设置活动地点后,签到人须填写一致(用户主动填写,不申请定位权限)。
签到记录保存:签到人、时间(服务端时钟)、所在地点名。

## 十二、健康计算器 + 日期计算器(已落地)
- **首页快捷入口(已落地)**:广场页顶部横滑条,与工具箱置顶共用偏好(store/tools-order);长按可上移/下移/移除;工具清单统一抽到 lib/tools/registry.ts,新增工具两处入口自动出现。

- **健康计算器**(纯本地):BMI(中国标准分级:偏瘦<18.5/正常/超重/肥胖)、BMR 基础代谢
  (Mifflin-St Jeor 公式,按性别/年龄/身高/体重)、每日饮水建议(体重×30-35ml)。
  免责声明常驻:仅参考,不构成医疗建议。
- **日期计算器**(纯本地):两日期间隔天数(含"排除周末"口径)、N 天后是哪天、
  当年进度条(第几天/共几天/百分比)。
- 说明:均为零接口零表纯函数(lib/calc/health.ts、lib/calc/dates.ts),jest 直测。

### 12.1 下一批 backlog(待排期)
| 工具 | 依赖 | 备注 |
| --- | --- | --- |
| 亲戚称呼计算器 | 纯本地 | 关系图三代内,实现量中等,传播性强 |
| 证件照规格/图片压缩 | 复用 image-compress 能力 | 鸿蒙侧随 image-picker 一并验证 |
| 白噪音/专注音 | 需音频资源包 | 与音乐播放器共用 expo-av 接入成果 |
| 世界时钟 | 纯本地(Intl 时区) | 与时钟工具合并为子页 |
| 目标关联任务(OKR→任务) | 纯服务端 | KR 进度自动按关联任务完成度推进 |

## 十三、体验动效批次(已落地)

全部基于已通过 RNOH 验证的 react-native-reanimated 与手势库,零新增原生依赖:
| 效果 | 落点 | 实现 |
| --- | --- | --- |
| 圆形扩散转场 | 首页快捷入口 + 工具箱宫格点击 | 透明 Modal 圆形遮罩从点击处扩散,动画中段跳转(components/ui/circular-reveal.tsx) |
| 勾选联动 | 待办勾选 | 勾选弹跳(spring 缩放)+ 删除线从左向右延伸 + 文字淡出 |
| 叠卡展开 | 待办「已完成」分组 | 完成项折叠为叠卡,点击展开 |
| 吸附卡+画线 | 音乐播放进度条 | 拖动时填充线与圆点跟随手指,松手弹簧吸附 seek(components/ui/draggable-progress.tsx) |
| 惯性滚轮 | 计时器分钟选择 | ScrollView snapToInterval + fast 减速,惯性滑动吸附刻度(components/ui/wheel-picker.tsx) |

## 十四、全设备响应式适配(已落地)

### 14.1 断点口径(行业标准)
phone < 640 ≤ tablet < 1024 ≤ desktop。CSS 层用 NativeWind v4 的 sm:/md:/lg: 变体
(v4 全平台生效,走窗口宽度);JS 层用 hooks/useBreakpoint.ts(同口径),供结构性决策。

### 14.2 适配规则(对标 Twitter/掘金/iPad 分栏模式)
| 设备 | 导航 | 内容 |
| --- | --- | --- |
| 手机 | 底部悬浮胶囊 | 全宽内容流 |
| 平板竖屏(md) | 底部胶囊 | 内容限宽 720 居中(阅读宽度) |
| 平板横屏/PC/折叠屏展开(lg) | 左侧竖排侧边栏(tabBarPosition:left,标签常显) | 内容限宽 960 居中 |
- 工具宫格列数:手机 2 列 / md 3 列 / lg 4 列(w-[47%] md:w-[31%] lg:w-[23%])
- 落点:hooks/useBreakpoint.ts + components/ui/floating-tab-bar.tsx(竖排 rail 变体)+ 五大 Tab 页

### 14.3 Web 交互与性能
- hover 反馈:工具卡/快捷入口 hover:opacity-90 + web:cursor-pointer(触屏设备不受影响)
- 按需加载:expo-router Web 端按路由自动分包;图片走 RemoteImage 单一通道;
  宽窄屏两种导航形态是同一组件条件渲染,无额外资源;未引任何响应式专用库

### 14.4 typed-routes 修复
expo-env.d.ts 重新生成后,运行时拼接的 push(string) 全部暴露;统一在
lib/nav.ts loginHref 返回 Href + 运行时拼接处 as never 断言收口(13 处)。

## 十五、加载骨架与按压触感批次(已落地)

### 15.1 骨架屏(Loading 状态规范)
13 个页面的"加载中…"空卡占位(SectionCard 空渲染)换成形状匹配的骨架:头像圆 + 两行文字。
- 单一来源:骨架块复用既有 components/ui/skeleton.tsx 的 Skeleton(bg-accent + animate-pulse),
  在其内新增 SkeletonList(rows) / SkeletonSummary;中途另建的 skeleton-loader.tsx 与它职责重复
  (两个同名 Skeleton 导出、且自身无脉冲动画),已合并删除
- 落点:动态、AI 会话、动态详情、用户主页、打卡、倒计时、目标、习惯、记账、记账分组、AA 分账、任务、待办
- SkeletonSummary 当前无调用方,是汇总卡骨架的预留位

### 15.2 按压触感(components/ui/pressable-scale.tsx)
按下 spring 缩到 0.97、松手回弹,Reanimated 驱动,零新增依赖。
- 落点:动态列表卡 1 处 + 广场 7 处(搜索框/筛选/排序与城市 Chip/快捷入口卡)+ 工具箱 2 处(AI 周报卡、工具宫格卡)
- 关键约束:className 与 transform 必须打在**同一个节点**。NativeWind 只为核心组件
  (View/Text/Pressable/ScrollView…)注册过 className→style,而 Animated.createAnimatedComponent()
  产出的是新组件对象;不显式 `cssInterop(AnimatedPressable, { className: 'style' })` 注册,
  className 会被当未知 prop 丢掉,mb-2、w-[47%]、flex-1 这类布局类静默失效。
- 不采用嵌套包裹:外层裸 Pressable 会丢尺寸(百分比与 flex-1 相对内容尺寸父节点解析为 0),
  内层多包一层又会打断父容器的 flex-row / items-center 排版。
- 站点 className 不含 transition-*/animate-*:否则 NativeWind 会对已是 animated 的组件再包一层,
  形成双层 animated。本批 10 个站点均为 hover:/web: 伪类,不触发该路径。

### 15.3 遗留
- 广场信息流卡片本体在共享组件 components/business/case-card.tsx(搜索页、我的寻亲等同用),
  本批未接按压触感,待实机确认缩放手感后统一收口
- CI 的 `lint:ci` 是 `--max-warnings=0`,仓内仍有 22 条既有 warning(_layout/message/
  case-card 相邻页/me/publish/tools 的 clock+music/feedback/pickers/menubar/monitor/session),
  与本批无关,属独立清理项

## 十六、视觉地基批次一(审计驱动的一致性收口)

### 16.1 审计结论(优化前置)
54 屏清点结果:页头只有 5 种形态且 66% 是同一种"大标题 + 副标题";三件套(SectionCard 56 处、
EmptyState 43 处、ErrorState 29 文件)复用 100+ 次,EmptyState 是纯文字,项目内 0 张业务插画。
模块辨识度仅 时钟/签到 6 位码/随机转盘/计算器 为高,记账-待办-习惯-目标-分账-任务-社区-我的 全是同构卡片流。
**定性为双轨**:寻亲端是 trust-first 公共服务(低变化、克制动效、清晰度优先),工具箱端才允许每模块一个签名交互。
同一套 token 与阶梯,不做"每模块各起一套风格"。

### 16.2 token 同源与真守卫
四处写值:global.css(HSL 变量)/ constants/Colors.ts TOKENS / NATIVE_BRAND 十六进制 / app.json 字面色。
原注释声称由 `__tests__/tokens.test.ts`、`token-sync.test.ts` 防漂移,这两个文件并不存在(谎报 CI 保护)。
现落地 `constants/__tests__/token-sync.test.ts` 五条断言:明暗两套 CSS 变量逐条对齐 TOKENS、
NATIVE_BRAND 每个十六进制必须由 HSL 派生、app.json 原生底色取自 NATIVE_BRAND、--radius 仍为 rem 阶梯源。
首跑即抓到 `app.json` 的 `#f8fafc` ≠ `--background`(`#f4f7fb`),已修。

### 16.3 圆角阶梯(单一来源)
`--radius` 12 → 14px,其余全部 `calc(var(--radius) ± n)`:xs 6 / sm 8 / md 10 / lg=xl 14 / 2xl 20 / full。
用途规则:xs 小控件(复选框)、sm 图标容器、md 控件(button/input)、lg-xl 卡片与容器、
2xl 底部 sheet 与大块 hero、full 胶囊(chip、分段器、悬浮 tab bar)。不再允许 `rounded-[Npx]` 字面值。

### 16.4 遮罩、深色与可读下限
新增 `overlay` 语义 token(明 `222 47% 8%` / 暗 `224 44% 3%`),18 处 `bg-black/*` 清零
(纯黑在深色下是"涂黑"而非"压暗");RemoteImage 三处写死灰底改走 `bg-muted` 随主题翻转;
收支报表图表标签 8px → 10px,全站最小字号现为 10px。

### 16.5 动效常量与无障碍降级
`constants/Motion.ts`:`SPRING(tap/release/snap/bounce)` + `DURATION` + `timing()`,统一带 `ReduceMotion.System`。
补齐此前 6 处"真实在跑但零降级"的动效:pressable-scale、circular-reveal、draggable-progress、
todo 勾选弹跳与画线、random 转盘(转盘用 RN 核心 Animated,以 `useReducedMotion` 置零时长降级)。
对照事实:仓内原有 11 处 ReduceMotion 全写在零引用的模板组件里,等于没有。
顺带修两个缺陷:draggable-progress 曾在 `useAnimatedStyle` 内逐帧重发 `withSpring`(改为独立 SharedValue)、
音乐进度外部 500ms 轮询裸赋值导致进度点硬跳(改 `withTiming` 平滑跟随)。

### 16.6 库级地雷(实测确认,立为约定)
- `shadow-*` / `shadow-black/5` 这类 class 在**原生端不渲染**:css-interop 解析器只有 shared/shorthand/text-shadow,
  原生只认 `shadowColor/shadowOffset/shadowOpacity/shadowRadius/elevation`。即卡片目前在设备上只有边框没有海拔。
- `Animated.createAnimatedComponent(X)` 产出的是新组件对象,不在 NativeWind 注册表内,`className` 会被静默丢弃,
  必须显式 `cssInterop(AnimatedX, { className: 'style' })`;`animate-pulse` 是例外(走 keyframes 通道,原生真会跑)。

### 16.7 尚未落地
海拔层级(card/dialog 的 native shadow 常量)、19 个工具页的双重页头(`_layout.tsx:64` 未关原生 header + 页内大标题 + `edges=['top']`)、
EmptyState 图形化与模块插画、主色由"电光蓝 + 深色霓虹青"收敛为信任基调(已定方向,下一批)。

## 十七、品牌色收敛(信任基调,对比达标)

### 17.1 为什么改
原基调自陈是"冷调冰白 + 电光蓝 / 深空藏蓝 + 霓虹青"(旧 global.css:9-14 注释)。寻亲端的受众是焦急的家属、
志愿者与民警,信任与可读优先于科技观感;霓虹青在深色端属于"发光"表达,与严肃场景相冲。
审计同时暴露两个可访问性缺陷:`text-pending` / `text-found` 在浅色底上做小字时对比不足,深色端
`destructive-foreground` 用近白字配亮红底(约 2.9:1)本来就不及格。用户批准收敛主色方向。

### 17.2 新色板(HSL 三元组,值以 constants/Colors.ts 与 global.css 同源)
| 语义 | 浅色 | 深色 | 说明 |
| --- | --- | --- | --- |
| primary | 208 58% 33%(#235785 墨蓝) | 202 52% 62%(#6cacd0) | 取消霓虹青,饱和度 58/52 |
| background | 210 36% 97%(#f5f7fa 冷白) | 224 40% 6% | 不用纯白纯黑 |
| urgent | 354 72% 43%(#bd1f2e) | 352 72% 62% | 文字态对比 5.76 |
| found | 160 62% 27%(#1a7053) | 160 60% 48% | 文字态对比 5.61 |
| pending | 32 82% 34%(#9e5b10) | 34 78% 58% | 文字态对比 4.94,与 urgent 色相距离拉大 |
| overlay | 222 45% 8% | 224 40% 3% | 遮罩走前景色深墨蓝,不是纯黑 |

### 17.3 对比是反推的,不是估的
用 WCAG 相对亮度对 `--background` 逐对验算后才定值:主按钮文字 7.24:1,正文 16.65:1,
次要文字 5.29:1,三个语义色文字态 4.94 / 5.61 / 5.76(均过 AA,正文级接近 AAA)。
深色端 `destructive-foreground` 改 `352 60% 10%`(亮红底配深墨字),与已有的 urgent/found/pending 深色做法一致。
三色仍受"必须配文字标签,不允许只靠颜色表意"的约束。

### 17.4 同源校验已生效
`NATIVE_BRAND` 五个十六进制与 `app.json` 两处 `backgroundColor` 一并改为 `#f5f7fa`,
`constants/__tests__/token-sync.test.ts` 五条断言全过,证明手写字面色与 HSL 严格对应、四处未漂移。

### 17.5 边界
这批只动 token 层,未做真机/鸿蒙观感核对;`shadow-*` 在原生不渲染的约定见 16.6,海拔层级仍未落地。
