# 管理后台前端对齐 ContiNew 契约方案

> 状态：阶段 0/1/2/3/4 已实施完成（2026-09-24，均未提交）；阶段 5 起待动工
> 当前门禁指标：`matched 136 / verbMismatch 0 / noPath 0 / registeredOffline 4 / unknownPermissions 0`
> 范围：`guard-find-frontend`（React + UMI4 + Ant Design Pro）向 `guard-find-backend`（ContiNew Admin 派生 Java 服务）的接口契约靠齐；功能范围以 `continew-admin-ui`（ContiNew 官方 Vue3 + Arco 实现）为参照基线。
> 结论来源：脚本静态比对 175 个后端路由 与 155 个前端调用点（详见 [附录：接口契约差异清单](./appendix-接口契约差异清单.md)），关键项已人工复核到文件行号。

## 1. 三方关系与对齐基线

```text
guard-find-app      → guard-find-server     用户侧（移动/Web）
guard-find-frontend → guard-find-backend    管理后台（本方案对象）
continew-admin-ui                            ContiNew 官方 Vue 前端，仅作功能与调用习惯参照，不改动
```

`guard-find-backend` 保留了 ContiNew 的 CRUD 代码生成约定（`@CrudRequestMapping` + `AbstractCrudController`）与 `top.continew.starter` 全套响应/异常/权限机制，因此**后端是唯一事实源**。`guard-find-frontend` 的历史血统是 ballcat（`system:organization:*`、`notify:announcement:*`、`errorCode/errorMessage/showType`、`records` 等命名残留），`vue-to-umi-migration` 变更已完成 92/97 个任务的页面搬运，但**服务层与权限码未做同一次契约清洗**——这就是当前"接口返回异常"的根因。

## 2. 现状量化

计数由 `guard-find-frontend/scripts/check-api-contract.mjs`（TypeScript AST 扫描，`npm run check:contract`）产出：

| 指标 | 数值 | 说明 |
|---|---|---|
| 后端注册路由（去重 verb+path） | 184 | CRUD 约定展开 + 方法级注解 |
| 前端调用点（去重 verb+path） | 160 | `src/services/web/**` + `src/pages/**` |
| ✅ 完全匹配 | 90（56%） | 动词与路径都存在 |
| ⚠️ 动词不匹配 | 26（16%） | 路径存在、HTTP 方法错误 → 后端返回 `code:"405"` |
| ❌ 路径不存在 | 44（28%） | 后端无该路由 → `code:"404"` |
| 后端有、前端未调用 | 94 | 功能未落地或走错路径 |
| 前端页面权限码 | 66 个不同 code，83 处引用 | 与后端 `system:x:create/update/delete` 派生规则大面积不一致 |

> 机器清单 `appendix-接口契约差异清单.md` 由改造前的一版正则扫描生成（155 / 84 / 26 / 45），是上表的**下界**；AST 版多捕获 5 处（`system/dict/data`、`dashboard` 的 4 处泛型换行调用）。

最痛的三类：

1. **动词错**：13 个模块的"编辑"都是 `PUT <base>`（id 在 body），后端只有 `PUT <base>/{id}`；5 个模块的"删除"是 `DELETE <base>/{id}`，另有 13 处删除请求把 body 写成裸数组 `[id]`（后端要求 `{"ids":[...]}`，见 `services/web/file/index.ts:25`、`system/dict/dict.ts:29` 等）。结果：所有新增可用，**改和删全部静默失败**。
2. **路径不存在**：`system/user/scope/{id}`、`system/user/pass/{id}`、`system/user/status`、`system/role/permission/code/{code}`、`system/role/select`、`system/menu/grant-list`、`system/dept/revised`、`system/dict/invalid-hash`、`i18n/*`、`notify/announcement/*`、`system/access-log/page`、`chat/*`。其中约 12 条属于**路径写法差异**（后端 `POST /schedule/job/trigger/{id}`，前端写成 `POST /schedule/job/{id}/trigger`；`GET /user/profile/basic` 实为 `PATCH /user/profile/basic/info` 等），可直接改正；其余 30 余条后端从未实现。
3. **权限码命名空间错**：前端 `:add/:edit/:del` ↔ 后端 `:create/:update/:delete`；前端 `system:organization:*` ↔ 后端 `system:dept:*`；前端 `notify:announcement:*` ↔ 后端 `system:notice:*`；前端 `monitor:sms:log:*` ↔ 后端 `system:sms:log:*`；前端 `system:config:*` ↔ 后端 `system:option:*`。后果分两层：UI 侧按钮被 `AccessControl` 直接隐藏（非超管看不到任何操作入口），即使手动调用接口也会被 `@SaCheckPermission` 判 403。

> 计数口径说明：附录由正则静态扫描生成，对"泛型换行"与"模板串内含引号"的调用点覆盖不完全，实测漏采 6 处（`services/web/dashboard/index.ts:17,24,37,44`、`services/web/system/dict/dict.ts:41` 等），真实失配数 **≥** 表中数字。阶段 0 会将该脚本改为 TypeScript AST 解析后再固化基线。

## 3. 为什么"接口返回异常"看不到原因

这是独立于路径问题的**请求层缺陷**，且是本次最优先项，因为它是所有其他问题的可观测性前提。

后端契约（已核实）：

```jsonc
// 成功
{ "code": "0",   "msg": "操作成功", "data": {...}, "success": true,  "timestamp": 1691453288000 }
// 失败（HTTP 状态恒为 200，见 application.yml:154 default-http-status-code-on-error: 200）
{ "code": "500", "msg": "名称为 [技术部] 的部门已存在", "data": null, "success": false, ... }
```

`code` 是**字符串**，消息字段是 **`msg`**，响应体**没有** `errorCode` / `errorMessage` / `showType`。

前端现状（`src/utils/RequestConfig.ts`）：

| 位置 | 现状 | 后果 |
|---|---|---|
| `:154` | `throw new BizError(errorMessage \|\| msg \|\| '业务处理失败', res)` | 消息放进 `Error.message`，但 `info` 里传的是原始 `res` |
| `:167-191` | `errorHandler` 只读 `errorInfo.errorMessage` 与 `errorInfo.showType` | 后端不返回这两个字段 → `showType` 落 `default` → `message.error(undefined)`，**弹出空提示框**，真实 `msg` 被丢弃 |
| `:133-135` | 命中 `code==='401'` 调 `Notify.logout()` 后不 `return` | 继续走 `isBizError` 抛错 → 登出弹窗与错误提示叠加 |
| `:69-80` | `_showError` 去重函数**从未被调用**；`MessageErrorWrapper.ts` 零引用 | 多请求同时失败时提示刷屏 |
| `:125` | 非 JSON 直接返回 | 导出/下载接口返回业务错误（JSON）时无法识别，页面 `catch` 后仅 `console.error` |
| `src/typings.d.ts:28-32` | `R<T> = { code: number; message: string; data: T }` | 与实际信封不符（`code` 为 string、字段为 `msg`、含 `success/timestamp`），类型层面无法暴露错误 |

另有两处一致性缺陷：`RequestConfig.ts:105-108` 发 `X-Tenant-Id`，而登录走 `X-Tenant-Code`（`services/web/login/index.ts:33,49,57`），两者未与后端的租户切换流程（`GET /tenant/common/id`）打通；`utils/Encrypt.ts:44-52` 的 RSA 公钥硬编码，当前与后端 `application-dev.yml:136` 一致（登录可用），但环境切换/密钥轮换会静默失效。

> 说明：登录字段 `username` 经复核与后端 `AccountLoginReq.username`（`AccountLoginReq.java:43`）**一致**，不需要改；`QueryParam {page,size,sort}` 与 `PageResult {list,total}` 也**已经正确**，前一次 `fix(system): align admin table pages with backend API contract` 提交已修好分页。

## 4. 与 ContiNew CRUD 约定的对齐矩阵

后端 `AbstractCrudController` + `BaseController.preHandle`（`guard-find-backend-common/.../BaseController.java:55-84`）派生出的统一形态：

| 能力 | 后端 | 前端现状 | 待改 |
|---|---|---|---|
| 分页列表 | `GET <base>?page&size&sort=a,desc` → `data:{list,total}` | ✅ 一致 | 无 |
| 非分页列表/树 | `GET <base>/list`、`GET <base>/tree`（返回数组，不分页） | 树页 `total: list.length` 凑数 | 用 `list/tree` 且关闭分页 |
| 详情 | `GET <base>/{id}` | 部分页面缺失 | 补 |
| 新增 | `POST <base>` → `data:{id}` | ✅ | 消费返回 id |
| 修改 | `PUT <base>/{id}` | ❌ `PUT <base>` | 13 处 |
| 单条删除 | `DELETE <base>` + `{"ids":[id]}`（无 `/{id}` 形式） | ❌ `DELETE <base>/{id}` | 6+ 处 |
| 批量删除 | `DELETE <base>` + `{"ids":[...]}` | ❌ 传裸数组 `[id]` | 5 处 |
| 导出 | `GET <base>/export`（octet-stream，绕过信封） | 仅 3 个模块有 | 补齐 + 统一下载助手 |
| 字典下拉 | `GET <base>/dict`、`GET <base>/dict/tree`（免权限） | ❌ 自建 `system/dict/data?dictCodes=`、`system/role/select` | 改用 `/system/common/dict/{code}` |
| 权限码 | `<prefix>:list/get/create/update/delete/export` + 显式码 | ❌ `:add/:edit/:del` | 66 码映射表 |

## 5. 功能范围对照（以 `continew-admin-ui` 为基线）

参考实现具备、当前 React 端**后端有接口但前端未接**的能力（即真实的功能缺口）：

| 能力 | 后端接口 | Vue 参考实现 | React 现状 |
|---|---|---|---|
| 客户端管理 | `GET/POST/PUT/DELETE /system/client` | `views/system/config/client` | ❌ 无页面 |
| 用户导入 | `/system/user/import/template`、`/import/parse`、`/import` | `user/ImportDrawer.vue` | ❌ 无 |
| 用户/部门导出 | `/system/user/export`、`/system/dept/export` | 工具栏导出 | ❌ 无 |
| 日志导出 | `/system/log/export/login`、`/export/operation` | 日志页导出 | ❌ 无（且 `access-log` 页指向不存在接口） |
| 角色授权 | `GET /system/role/permission/tree`、`PUT /system/role/{id}/permission`、`GET/POST /{id}/user`、`DELETE /system/role/user` | 功能权限/角色用户 Tab | ⚠️ 走了不存在的 `permission/code/{roleCode}` |
| 文件目录/新建 | `POST /system/file/dir`、`GET /system/file/dir/{id}/size` | 文件管理左树 + 新建目录 | ⚠️ 仅列表/上传 |
| 秒传 + 分片上传 | `GET /system/file/check`、`/system/multipart-upload/{init,part,complete,cancel}` | `useMultipartUploader` | ❌ 无（大文件走整包上传） |
| 字典/菜单缓存清理 | `DELETE /system/dict/cache/{code}`、`DELETE /system/menu/cache` | 工具栏按钮 | ❌ 无（前端自造 `dict/invalid-hash`） |
| 站点配置注入 | `GET /system/common/dict/option/site` | `initSiteConfig()` 改标题/favicon | ❌ 无 |
| 租户选择 | `GET /tenant/common/id` + `X-Tenant-Id` | 登录页租户下拉 | ⚠️ header 不一致 |
| 数据分析看板 | `/dashboard/analysis/{overview/pv,overview/ip,timeslot,access/trend/{days}}` | 完整 Analysis 页 | ⚠️ 只接了 4 个中的 geo/os/browser/module |
| 消息中心（公告） | `/user/message/notice*` | 公告列表/详情 | ❌ 未接 |
| 调度任务分组 | `GET /schedule/job/group` | 分组筛选 | ❌ 未接 |
| 验证码类型 | `GET /captcha/{image,sms,mail}`、`POST /captcha/behavior` | 行为/短信/邮箱 | ⚠️ 仅 image + sms/mail 路径拼接方式有误 |

反之，前端存在但**后端从未实现**、且菜单已在 `guard-find-backend-api/src/main/resources/db/changelog/mysql/menu_convention_routing.sql` 里种入数据库的入口（点击即报错）：

- `/i18n`（国际化管理，menu id 1260）→ 后端无 i18n 模块
- `/monitor/log/access`（访问日志，menu id 2036）→ 后端无 access-log
- `/chat`（智能助手，menu id 9200）→ 后端（backend 侧）无 chat 接口
- `notify/announcement/*`（新版公告模型）→ 后端只有 `system/notice`
- 用户数据权限 `system/user/scope`、部门层级校正 `system/dept/revised`、`system/role/select`、`system/menu/grant-list` → 后端无

按已确认的处置策略：这些能力**暂时下线**（前端路由过滤器隐藏入口 + 服务代码归档标注），后端补实现或恢复菜单另立变更。

## 6. 分阶段推进方案

阶段按"先让错误可见 → 再让改动能用 → 再让权限可见 → 再对齐字段 → 最后补功能"排序，每阶段可独立合并、独立回滚，验收指标可被脚本量化。

### 阶段 0 · 建立契约回归基线（✅ 已完成）

把本方案的比对脚本固化为 `guard-find-frontend/scripts/check-api-contract.mjs`，输出 `docs/api-contract-report.md`，并在 `package.json` 加 `npm run check:contract`。指标基线：`matched=84 / verbMismatch=26 / noPath=45`。之后每阶段跑一次，数字只准下降。

验收：脚本在本地与 CI 均可运行，输出可 diff。

### 阶段 1 · 请求层与异常适配（✅ 已完成，除 1.7 租户选择流程与 1.12 真实后端联调）

改动清单：

1. `src/typings.d.ts`：`R<T>` 改为 `{ code: string; msg: string; data: T; success: boolean; timestamp: number }`，删除 `GLOBAL.Router` 中后端不存在的字段（`targetType/uri/hidden/keepAlive/remarks/status`）。
2. `src/utils/RequestConfig.ts`：
   - `errorHandler` 读取 `info.msg`（保留 `errorMessage` 兼容），按 `code` 分派 UI 行为；
   - `401` 分支 `return` 短路，避免与登出弹窗叠加；
   - 启用 `_showError` 去重（1.5s 窗口），删除死代码 `MessageErrorWrapper` 或让其真正被引用；
   - 增加 `code==='404'/'405'` 的开发期告警（`console.error('[api-contract] ...')`），让契约漂移在开发时立刻暴露；
   - blob 响应若 `type` 以 `application/json` 开头，用 `FileResponseReader` 解析出 `msg` 再抛错（对齐 Vue `http.ts:76-91`）；
   - 新增统一 `download(url, params, filename)` 工具，替换各页 `URL.createObjectURL` 手写逻辑。
3. 错误码 → UI 行为映射表（后端恒返回 HTTP 200）：

   | `code` | 后端语义 | UI 行为 |
   |---|---|---|
   | `"0"` | 成功 | 无 |
   | `"400"` | 参数/业务校验失败 | `message.warning(msg)`；表单类请求把 msg 回填到对应字段 |
   | `"401"` | 登录态失效 | 单次登出弹窗 + 跳登录（携带 redirect） |
   | `"403"` | 无权限 / 租户不匹配 | `message.error(msg)`，页面级 403 走异常页 |
   | `"404"/"405"` | 路由不存在/方法不匹配 | 开发期 `console.error` + 提示"接口未实现"，生产期静默计数 |
   | `"500"` | 业务异常（含唯一性冲突、数据不存在） | `message.error(msg)` |
   | `"1"` | 未捕获异常（含数据库异常） | `notification.error` + 提示联系管理员 |

4. `X-Tenant-Id` / `X-Tenant-Code` 统一由 `Tenant` store 管理，登录流程接 `GET /tenant/common/id`；RSA 公钥从 `config/config.*.ts` 的 `define` 注入，去掉 `Encrypt.ts` 硬编码与 ballcat 遗留的 AES 分支。

验收：① 任一失败请求能看到后端真实 `msg`；② 401 只触发一次登出；③ 同一错误 1.5s 内只弹一次；④ `npx tsc --noEmit` 与 `npm run lint` 通过；⑤ 手动验证"编辑用户/删除部门"两条链路，提示从空白变为具体原因。

### 阶段 2 · CRUD 动词与批量操作（✅ 已完成，UI 冒烟待真实后端）

新增 `src/services/web/crud.ts` 工厂（`page/list/tree/get/create/update(id,body)/remove(ids)/dict/treeDict/exportUrl`，
删除统一为 `DELETE <base>` + `{ids}`），并迁移 12 个 CRUD 服务；`system/option`、`system/file`、
`user/message`、`user/profile`、`monitor/online` 等非 CRUD 形态按真实端点手写对齐；
`code/generator`、`schedule/*` 属后端自定义端点，保持手写但改正路径。
服务函数名与签名保持不变，因此页面基本零改动；额外改了 3 处功能性错误：
角色授权（`roleCode` → `roleId` + 权限树/`menuIds`）、用户分配角色（`PATCH /system/user/{id}/role`）、
字典取值（`GET /system/common/dict/{code}` 适配为原 `SysDictData` 结构）。
6 个树/列表页去掉 `total: list.length` 伪造分页并显式 `pagination={false}`。

结果：`matched 136 / verbMismatch 0 / noPath 31`（31 条全部为后端未实现的 B 类，归阶段 3）。
`npx tsc --noEmit` 通过、`biome lint` 仅剩 3 处历史告警、`npm run build:zy` 通过。

**须知**：动词与路径已对，但请求体字段名仍是 ballcat 风格（如 `SysUserDto.pass` 对后端
`UserReq.password`），后端 `fail-on-unknown-properties: false` 会静默忽略多余字段，
表现为"200 成功但字段没生效"。这属阶段 5 范围。

### 阶段 3 · 路径修正与失效模块下线（✅ 已完成，UI 冒烟待真实后端）

结果：`matched 136 / verbMismatch 0 / noPath 0`，另有 4 个 chat 调用点登记为"已下线"（`scripts/offline-endpoints.json`，不计入门禁）。

- 删除后端从未实现的模块：`i18n`（服务 + 页面）、访问日志 `monitor/log/access`（服务 + 页面）、零引用的重复公告模块 `services/web/notify/announcement/**`
- 移除死接口封装与对应 UI：`menu.listRoleGrant`、`dict.validHash`、部门"层级校正"按钮、用户列表头像上传（管理员改他人头像无接口）、`user.getScope/putScope`
- 两处**保功能而非砍功能**：公告"发布/撤回"改为 `GET /system/notice/{id}` 取详情 + `PUT /system/notice/{id}` 回写 `status`（`NoticeReq` 有非空校验，不能只传状态）；用户"启用/禁用（含批量）"同样走"取详情 → 回写 `status`"
- `RouteUtils` 增加 `OFFLINE_MENU_PATHS = ['/i18n','/monitor/log/access','/chat']` 过滤并导出 `isOfflinePath`；`ChatFloat` 挂载点同步下线
- 原 Open Question 已用证据回答:`guard-find-backend` 与 `guard-find-server` **都**没有 `/chat` 控制器,所以 chat 不是"改代理"而是确无后端;`pages/chat/**` 代码保留,等后端能力落地后摘掉过滤即可恢复

遗留:`sys_menu` 中 `/i18n`、`/monitor/log/access`、`/chat` 三行仍在库裡,需要一次后端 Liquibase 变更清理(否则新环境仍靠前端过滤兜底)。

### 阶段 4 · 权限码映射与按钮可见性（✅ 已完成，受限账号实机验证待做）

权威来源改为**后端种子 SQL 的 `sys_menu.permission` + `@SaCheckPermission` 字面量**,由扫描脚本每次现算(共 126 条),而不是手写清单——这样 ContiNew 升级或后端改名时门禁会立刻报警,不会留第二份会漂移的真相。

这一步也纠正了我此前方案里的猜测错误(实际命名是 camelCase 模块名,不是路径直译):

| 此前方案写的 | 后端实际用的 |
| --- | --- |
| `system:sms:config:*` | `system:smsConfig:*` |
| `monitor:sms:log:*` / `system:sms:log:*` | `system:smsLog:*` |
| `system:config:*` → `system:option:*` | 按类目拆成 `system:siteConfig / securityConfig / loginConfig / mailConfig` 的 `:get`/`:update` |
| （未提及） | `system:dictItem:*`、`system:fileRecycle:*` 独立命名空间 |

已完成:`src/config/permissionCodes.ts` 由脚本 `--emit-permission-manifest` 自动生成(派生产物,供补全与核对,页面仍写字面量);18 个页面文件共 50 个不合规码改写完毕,在用 55 个码全部命中权威集合;`check:contract:ci` 增加 `--max-unknown-permissions 0` 门禁。

两处**去掉门禁而不是换个码**:`user/message` 批量删除(后端 `UserMessageController` 只做登录校验,套任何权限码都会让按钮永久隐藏)、驾驶舱两个"新建任务"按钮(`task:item:create` 是 ballcat 遗留,且按钮本就没有 `onClick`,属死控件,一并删)。

判定逻辑收敛:新增 `src/utils/access.ts` 作唯一实现(`*:*:*` + 超级角色),`src/access.ts` 与 `useAccess` 都委托它;`AccessControl` 增加 `mode="disable"`(禁用态 + Tooltip 说明原因)与 `role` 维度,避免"按钮凭空消失"式的难排查。

验收(未做):用非超管账号登录,确认各页按钮可见性与后端放行一致 —— 这依赖 `sys_role_menu` 里受限角色确实被授予了这些新码,若受限角色的授权数据仍是 ballcat 旧码,前端改对了也照样看不到按钮,需要先核对那份数据。

### 阶段 5 · 菜单/路由语义与字段级契约（3 天，P1）

1. 修正 `typings.d.ts` 与 `pages/system/menu` 的 `type` 注释（后端 `MenuTypeEnum`：1 目录 / 2 菜单 / 3 按钮；`RouteUtils` 现有过滤逻辑正确，注释误导）。
2. `RouteResp` 带 `@JsonInclude(NON_EMPTY)`：`isHidden/isExternal/isCache` 为 `false` 时**字段整体缺失**，`sort=0` 亦缺失 → `RouteUtils` 必须显式 `=== true` 判断与默认值，不能依赖字段存在。
3. 约定式路由与 `component` 字符串（如 `monitor/log/login/index`）建立一致性校验：脚本比对 `sys_menu.component` 与 `src/pages/**/*.tsx`，缺失即构建失败，避免"菜单点开白屏"。
4. 逐模块把 ballcat 风格 `*Vo/*Dto/*Qo` 类型替换为后端 `*Resp/*DetailResp/*Req` 字段名（用户、部门、菜单、角色、字典、文件、公告、调度、租户、开放应用），重点核实枚举（`GenderEnum` 等）与 Long 精度（后端 `big-number-serialize-mode: FLEXIBLE`，雪花 ID 以字符串下发 → 前端 `id: number` 需改为 `string | number`）。

验收：`npx tsc --noEmit` 通过；抽查 10 个页面网络面板无字段缺失/类型告警；雪花 ID 用户不再出现精度截断。

### 阶段 6 · 功能补齐到 ContiNew 基线（按里程碑排期，P2）

按第 5 节缺口表逐项补齐，建议顺序：客户端管理 → 角色授权（权限树） → 用户导入/各模块导出 → 文件目录 + 分片与秒传 → 站点配置注入 → 数据分析看板完整模块 → 消息中心公告 → 调度分组。每项一个 OpenSpec change，避免大爆炸式 PR。

## 7. 风险与约束

- **只改前端**：本轮不动 `guard-find-backend`。因此后端已种入但无接口的菜单（`/i18n`、`/monitor/log/access`、`/chat`）在前端隐藏后，数据库仍存在这些行；需要后续一次后端 liquibase 变更清理，否则新环境部署仍会出现相同漂移。
- **过时的 OpenSpec 变更**：`openspec/changes/system-management-module`（0/44）与 `full-project-replication`（0/41）是以 **ballcat-ui-react** 为参照写的，其接口命名（`system/dict/data`、`notify/announcement`、`system/log` 单页）正是本次要清除的漂移来源，**不能作为实现依据**。本方案与 `align-backend-contract` 取代二者，建议评审后将其标记为 superseded 或归档。
- **`continew-admin-ui` 是第三方参考**：只读，不改，不引入其 Vue 代码；本方案借鉴的是它的契约处理方式（`http.ts` 的 401/blob 处理、`useTable` 的 `list/total` 归一化、`v-permission` 的权限码命名）。
- **`pages/chat` 依赖 `guard-find-server`**：若聊天能力实际由用户侧服务提供，正确做法是给它单独的代理前缀与 request 实例，而不是并入后台契约；阶段 3 的"下线"仅指后台菜单入口。
- **构建配置遗留**：`package.json` 的 `build:zy / build:gj` 与 `UMI_TAG`、`namespaceName` 属于其他项目（三生制药）遗留，`requestPrefix`、`clientId`、RSA 公钥只在 `UMI_ENV=dev|prod` 时注入，默认构建会得到 `undefined` 前缀与 `"undefined"` 命名空间的 localStorage key。建议在阶段 1 一并清理，否则任何环境相关修复都不可信。
- **进度可信度**：阶段 0 的脚本是唯一能防"自称改完"的手段，务必先做。

## 8. 评审需要确认的事项

1. 阶段 1+2+3 已作为同一轮实施完成（未提交），是否按一个 PR 合并还是按阶段拆三个提交。
2. ~~`chat` 是否由 `guard-find-server` 承载~~ 已核实：`guard-find-backend` 与 `guard-find-server` 均无 `/chat` 接口，故按"无后端"下线处理（代码保留、入口隐藏、登记于 `scripts/offline-endpoints.json`）。若日后接入，需先确定它属于哪一侧服务。
3. `sys_menu` 中 `/i18n`、`/monitor/log/access`、`/chat` 三行仍需一次后端 Liquibase 变更清理，否则新环境靠前端过滤兜底。
4. 阶段 6 的功能补齐是否与 ContiNew 版本升级节奏绑定（后端 `continew-starter 2.15.0`）。
5. 非超管角色的 `sys_role_menu` 权限码是否已按 ContiNew 刷库 —— 决定阶段 4 是否需要与后端数据同步。

---

OpenSpec 变更工件见 `guard-find-frontend/openspec/changes/align-backend-contract/`（proposal / design / tasks / specs）。
