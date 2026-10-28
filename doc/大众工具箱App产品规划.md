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

**P2 · 观察后做（依赖重或同质化竞争激烈）**
| 工具 | 说明 |
| --- | --- |
| 日记/心情日历 | 结合 AI 做月度情绪总结（M2 能力复用），但需内容审核能力跟上 |
| 快递查询 | 第三方聚合 API 有成本，且频次不如前几项 |
| 账单导入（支付宝/微信 CSV） | 记账深度用户才需要，放记账工具成熟后 |
| OCR 截图记账 | 依赖 OCR 服务；先用 AI 对话的视觉模型试点（上传截图→提取金额） |

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
