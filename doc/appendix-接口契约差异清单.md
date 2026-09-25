# 附录：接口契约与权限码差异清单（机器生成）

> 生成：`cd guard-find-frontend && node scripts/check-api-contract.mjs`（TypeScript AST 扫描；识别 `createCrud()` 工厂调用并还原为真实端点；权限码以 guard-find-backend 种子 SQL + `@SaCheckPermission` 现算为权威）。

> `registeredOffline` = 已在 `scripts/offline-endpoints.json` 登记、入口已隐藏的后端未实现能力，不计入 CI 门禁。`{}` 是脚本对路径参数/模板插值的归一化。

```text
后端路由 184 条 / 前端调用点 140 处（扫描范围 ..\guard-find-backend 与 src/）

=== 已匹配 (136) ===
DELETE monitor/online/{}                                 src/services/web/monitor/online.ts:13
DELETE open/app                                          src/services/web/open/app.ts:28
DELETE schedule/job/{}                                   src/services/web/schedule/job.ts:27
DELETE system/dept                                       src/services/web/system/organization/index.ts:33[crud.remove]
DELETE system/dict                                       src/services/web/system/dict/dict.ts:30[crud.remove]
DELETE system/dict/cache/{}                              src/services/web/system/dict/dict.ts:38
DELETE system/dict/item                                  src/services/web/system/dict/dictItem.ts:28
DELETE system/file                                       src/services/web/file/index.ts:24, src/services/web/file/index.ts:32
DELETE system/file/recycle/{}                            src/services/web/file/index.ts:53
DELETE system/menu                                       src/services/web/system/menu/menu.ts:24[crud.remove]
DELETE system/menu/cache                                 src/services/web/system/menu/menu.ts:33
DELETE system/notice                                     src/services/web/notify/announcement.ts:34
DELETE system/role                                       src/services/web/system/role/index.ts:26[crud.remove]
DELETE system/role/user                                  src/services/web/system/role/index.ts:75
DELETE system/sms/config                                 src/services/web/sms/config.ts:28
DELETE system/sms/log                                    src/services/web/sms/log.ts:14, src/services/web/sms/log.ts:21
DELETE system/storage                                    src/services/web/storage/index.ts:28
DELETE system/user                                       src/services/web/system/user.ts:38[crud.remove], src/services/web/system/user.ts:42[crud.remove]
DELETE tenant/management/{}                              src/services/web/tenant/management.ts:27
DELETE tenant/package/{}                                 src/services/web/tenant/package.ts:27
DELETE user/message                                      src/services/web/user-message/index.ts:29, src/services/web/user-message/index.ts:36
DELETE user/profile/social/{}                            src/services/web/user-profile/index.ts:50
GET auth/user/info                                       src/services/web/login/index.ts:39, src/services/web/user-profile/index.ts:6
GET auth/user/route                                      src/services/web/login/index.ts:94
GET auth/{}                                              src/services/web/login/index.ts:71
GET captcha/image                                        src/services/web/login/index.ts:43
GET captcha/mail                                         src/services/web/login/index.ts:67
GET captcha/sms                                          src/services/web/login/index.ts:63
GET code/generator/config                                src/services/web/code/generator.ts:12
GET code/generator/config/{}                             src/services/web/code/generator.ts:19
GET code/generator/field/{}                              src/services/web/code/generator.ts:32
GET code/generator/preview/{}                            src/services/web/code/generator.ts:45
GET dashboard/access/trend/{}                            src/services/web/dashboard/index.ts:37
GET dashboard/analysis/browser                           src/services/web/dashboard/index.ts:63
GET dashboard/analysis/geo                               src/services/web/dashboard/index.ts:31
GET dashboard/analysis/module                            src/services/web/dashboard/index.ts:51
GET dashboard/analysis/os                                src/services/web/dashboard/index.ts:57
GET dashboard/analysis/overview/ip                       src/services/web/dashboard/index.ts:24
GET dashboard/analysis/overview/pv                       src/services/web/dashboard/index.ts:17
GET dashboard/analysis/timeslot                          src/services/web/dashboard/index.ts:44
GET dashboard/notice                                     src/services/web/dashboard/index.ts:11
GET monitor/online                                       src/services/web/monitor/online.ts:6
GET open/app                                             src/services/web/open/app.ts:6
GET open/app/export                                      src/services/web/open/app.ts:47
GET open/app/{}/secret                                   src/services/web/open/app.ts:35
GET schedule/job                                         src/services/web/schedule/job.ts:6
GET schedule/log                                         src/services/web/schedule/log.ts:6
GET system/common/dict/{}                                src/services/web/system/dict/dict.ts:46
GET system/dept/dict/tree                                src/services/web/system/organization/index.ts:42[crud.treeDict]
GET system/dept/tree                                     src/services/web/system/organization/index.ts:15[crud.tree], src/services/web/system/organization/index.ts:19[crud.tree]
GET system/dept/{}                                       src/services/web/system/organization/index.ts:37[crud.get]
GET system/dict/item                                     src/services/web/system/dict/dictItem.ts:6
GET system/dict/list                                     src/services/web/system/dict/dict.ts:16[crud.list]
GET system/file                                          src/services/web/file/index.ts:6
GET system/file/recycle                                  src/services/web/file/index.ts:39
GET system/file/statistics                               src/services/web/file/index.ts:59
GET system/log                                           src/services/web/log/loginLog.ts:6, src/services/web/log/operationLog.ts:6
GET system/menu/tree                                     src/services/web/system/menu/menu.ts:10[crud.tree]
GET system/menu/{}                                       src/services/web/system/menu/menu.ts:28[crud.get]
GET system/notice                                        src/services/web/notify/announcement.ts:11
GET system/notice/{}                                     src/services/web/notify/announcement.ts:54, src/services/web/notify/announcement.ts:82, src/services/web/user-message/index.ts:49
GET system/option                                        src/pages/system/config/components/OptionForm.tsx:27, src/services/web/system/config/index.ts:6
GET system/role/dict                                     src/services/web/system/role/index.ts:39[crud.dict]
GET system/role/list                                     src/services/web/system/role/index.ts:12[crud.list], src/services/web/system/role/index.ts:34[crud.list]
GET system/role/permission/tree                          src/services/web/system/role/index.ts:44
GET system/role/{}                                       src/services/web/system/role/index.ts:30[crud.get], src/services/web/system/role/index.ts:49[crud.get]
GET system/role/{}/user                                  src/services/web/system/role/index.ts:64
GET system/sms/config                                    src/services/web/sms/config.ts:6
GET system/sms/log                                       src/services/web/sms/log.ts:6
GET system/sms/log/export                                src/services/web/sms/log.ts:28
GET system/storage/list                                  src/services/web/storage/index.ts:6
GET system/user                                          src/services/web/system/user.ts:18[crud.page]
GET system/user/list                                     src/services/web/system/user.ts:15[crud.list]
GET system/user/{}                                       src/services/web/system/user.ts:14[crud.get], src/services/web/system/user.ts:77[crud.get]
GET tenant/management                                    src/services/web/tenant/management.ts:6
GET tenant/package                                       src/services/web/tenant/package.ts:6
GET tenant/package/menu/tree                             src/services/web/tenant/package.ts:34
GET tenant/package/{}                                    src/services/web/tenant/package.ts:44
GET user/message                                         src/services/web/user-message/index.ts:6
GET user/message/unread                                  src/services/web/user-message/index.ts:43
GET user/profile/social                                  src/services/web/user-profile/index.ts:38
PATCH open/app/{}/secret                                 src/services/web/open/app.ts:41
PATCH schedule/job/{}/status                             src/services/web/schedule/job.ts:41
PATCH system/option/value                                src/services/web/system/config/index.ts:28
PATCH system/user/{}/password                            src/services/web/system/user.ts:50
PATCH system/user/{}/role                                src/services/web/system/user.ts:61
PATCH user/message/read                                  src/services/web/user-message/index.ts:14
PATCH user/message/readall                               src/services/web/user-message/index.ts:22
PATCH user/profile/avatar                                src/services/web/user-profile/index.ts:18
PATCH user/profile/basic/info                            src/services/web/user-profile/index.ts:11
PATCH user/profile/email                                 src/services/web/user-profile/index.ts:34
PATCH user/profile/password                              src/services/web/login/index.ts:87, src/services/web/user-profile/index.ts:26
PATCH user/profile/phone                                 src/services/web/user-profile/index.ts:30
POST auth/login                                          src/services/web/login/index.ts:31, src/services/web/login/index.ts:47, src/services/web/login/index.ts:55, src/services/web/login/index.ts:75
POST auth/logout                                         src/services/web/login/index.ts:20
POST code/generator/config/{}                            src/services/web/code/generator.ts:25, src/services/web/code/generator.ts:38
POST code/generator/{}/download                          src/services/web/code/generator.ts:51, src/services/web/code/generator.ts:58
POST open/app                                            src/services/web/open/app.ts:13
POST schedule/job                                        src/services/web/schedule/job.ts:13
POST schedule/job/trigger/{}                             src/services/web/schedule/job.ts:34
POST schedule/log/retry/{}                               src/services/web/schedule/log.ts:21
POST schedule/log/stop/{}                                src/services/web/schedule/log.ts:14
POST system/dept                                         src/services/web/system/organization/index.ts:23[crud.create]
POST system/dict                                         src/services/web/system/dict/dict.ts:20[crud.create]
POST system/dict/item                                    src/services/web/system/dict/dictItem.ts:13
POST system/file/upload                                  src/services/web/file/index.ts:16
POST system/menu                                         src/services/web/system/menu/menu.ts:14[crud.create]
POST system/notice                                       src/services/web/notify/announcement.ts:18
POST system/role                                         src/services/web/system/role/index.ts:16[crud.create]
POST system/role/{}/user                                 src/services/web/system/role/index.ts:83
POST system/sms/config                                   src/services/web/sms/config.ts:13
POST system/storage                                      src/services/web/storage/index.ts:13
POST system/user                                         src/services/web/system/user.ts:22[crud.create]
POST tenant/management                                   src/services/web/tenant/management.ts:13
POST tenant/package                                      src/services/web/tenant/package.ts:13
POST user/profile/social/{}                              src/services/web/user-profile/index.ts:43
PUT open/app/{}                                          src/services/web/open/app.ts:20
PUT schedule/job/{}                                      src/services/web/schedule/job.ts:20
PUT system/dept/{}                                       src/services/web/system/organization/index.ts:28[crud.update]
PUT system/dict/item/{}                                  src/services/web/system/dict/dictItem.ts:20
PUT system/dict/{}                                       src/services/web/system/dict/dict.ts:25[crud.update]
PUT system/file/recycle/restore/{}                       src/services/web/file/index.ts:47
PUT system/menu/{}                                       src/services/web/system/menu/menu.ts:19[crud.update]
PUT system/notice/{}                                     src/services/web/notify/announcement.ts:26, src/services/web/notify/announcement.ts:67
PUT system/option                                        src/pages/system/config/components/OptionForm.tsx:46, src/services/web/system/config/index.ts:14
PUT system/role/{}                                       src/services/web/system/role/index.ts:21[crud.update]
PUT system/role/{}/permission                            src/services/web/system/role/index.ts:56
PUT system/sms/config/{}                                 src/services/web/sms/config.ts:20
PUT system/sms/config/{}/default                         src/services/web/sms/config.ts:35
PUT system/storage/{}                                    src/services/web/storage/index.ts:20
PUT system/storage/{}/default                            src/services/web/storage/index.ts:35
PUT system/storage/{}/status                             src/services/web/storage/index.ts:41
PUT system/user/{}                                       src/services/web/system/user.ts:33[crud.update], src/services/web/system/user.ts:90[crud.update]
PUT tenant/management/{}                                 src/services/web/tenant/management.ts:20
PUT tenant/management/{}/admin/pwd                       src/services/web/tenant/management.ts:38
PUT tenant/package/{}                                    src/services/web/tenant/package.ts:20, src/services/web/tenant/package.ts:51

=== HTTP 方法不匹配 —— 后端仅有：见 actual (0) ===

=== 后端不存在该路径（需修正或下线） (0) ===

=== 已登记下线（scripts/offline-endpoints.json，不计入门禁） (4) ===
DELETE chat/sessions/{}                                  src/pages/chat/services/index.ts:18
GET chat/sessions                                        src/pages/chat/services/index.ts:6
GET chat/sessions/{}/messages                            src/pages/chat/services/index.ts:24
POST chat/sessions                                       src/pages/chat/services/index.ts:12

=== 权限码不在后端权威集合（后端共 126 个可校验码）—— 会导致按钮永久隐藏或 403 (0) ===

=== 汇总 ===
{
  "backendRoutes": 184,
  "frontendCallSites": 140,
  "matched": 136,
  "verbMismatch": 0,
  "noPath": 0,
  "registeredOffline": 4,
  "unusedBackendRoutes": 48,
  "unresolvableCrudTemplates": 6,
  "backendPermissionCodes": 126,
  "frontendPermissionCodes": 55,
  "unknownPermissions": 0
}

```
