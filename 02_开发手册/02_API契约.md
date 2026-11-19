# API 契约

基线：/api/v1；这是文字契约，不是接口实现或 OpenAPI 代码。内部端点用内部会话，review 端点用客户会话。所有写操作校验 CSRF 与 Origin，登录和验证端点也适用。除公开验证入口外均要求有效身份。

## 通用响应与字段

成功响应含 data、request_id；分页另含 next_cursor。失败响应含 error.code、error.message、error.fields（可选）、request_id。ID 为字符串，时间为含 Z 的 ISO 8601，枚举大写，字节数为非负整数。列表默认20、最大100，游标不允许跨空间使用。

Project 返回 id/title/status/current_version_id/revision；Version 返回 id/project_id/number/status/title/description/assets/revision/published_at；Asset 返回 id/display_name/kind/bytes/status/failure_code，不返回私有对象 key；Decision 返回 id/version_id/outcome/message/decided_at/receipt_id。外部响应不含内部成员名单、配额、对象 key 或未授权版本 ID。

## 身份与空间

| 方法与路径 | 输入 | 输出／限制 |
|---|---|---|
| GET /auth/csrf | 无 | 提供同源 CSRF 引导；不创建已认证身份 |
| POST /auth/challenges | email | 202；统一提示及 challenge_id；无资格也不泄露存在性 |
| POST /auth/verify | challenge_id、code | 200；设置内部会话 Cookie，返回用户和空间摘要 |
| POST /auth/logout | 无 | 204；撤销本会话 |
| GET /me | 无 | 当前身份、角色、空间状态、剩余额度 |
| GET /members | cursor | 成员与邀请列表；仅内部可见 |
| POST /invitations | email | Owner；201 邀请；事务内检查席位 |
| POST /invitations/accept | invite_token、已验证登录身份 | 接受邮箱必须匹配；200 成员关系 |
| DELETE /members/{id} | 无 | Owner；204；禁止删除 Owner，立即失效目标成员权限 |

邀请接受前允许已被邀请但未入空间的用户完成身份验证，/me 返回待接受状态；不能据此访问业务对象。Owner 初始空间由受控运营后台开通，不提供自助建空间端点。

接受邀请会新增成员关系，只读/已经到期/冻结时拒绝；身份验证或邀请邮件成功不能豁免该检查。/me 对待接受及冻结身份只返回最小身份和状态，不包含配额、成员列表或项目摘要。所有现有端点的状态归类见 [权限动作对照](03_权限与安全.md)。

## 项目与交付

| 方法与路径 | 输入 | 输出／约束 |
|---|---|---|
| GET /projects | q、status、cursor | 分页项目，仅所属空间；q 最多120字符 |
| POST /projects | title | 201 Project；占活跃名额 |
| GET/PATCH /projects/{id} | PATCH: title、expected_revision | 200 Project；改项目名不改历史快照 |
| POST /projects/{id}/archive | expected_revision | 200；不得有审阅中版本 |
| POST /projects/{id}/restore | expected_revision | 200；恢复归档，检查配额 |
| DELETE /projects/{id} | expected_revision、confirmation_title | 202；Owner 发起删除并返回可撤回期限 |
| POST /projects/{id}/cancel-deletion | expected_revision | 200；Owner、宽限期内 |
| GET /projects/{id}/versions | cursor | 内部历史列表；客户不能调用 |
| POST /projects/{id}/versions | title、description | 201 DRAFT；已有草稿返回409 |
| GET/PATCH /versions/{id} | PATCH: title、description、asset_order、expected_revision | 200；仅草稿可改；文件顺序应含全部现有附件一次 |
| DELETE /versions/{id} | expected_revision | 202；仅草稿；异步清文件 |
| POST /versions/{id}/publish | reviewer_email、expires_in_days、expected_project_revision、expected_version_revision | 201；版本、share_id、一次性显示的 share_url |
| POST /versions/{id}/withdraw | reason、expected_revision | 200；只允许 IN_REVIEW |
| POST /versions/{id}/shares | expires_in_days | 201；只限当前已发布且未撤回版本；绑定原邮箱，替换旧分享 |
| DELETE /shares/{id} | 无 | 204；撤销授权和会话，不改变版本状态 |

发布前前端刷新项目与版本 revision。409 时必须重新读取状态供用户确认，不能自动带新 revision 重发。

## 文件

| 方法与路径 | 输入 | 输出／限制 |
|---|---|---|
| POST /versions/{id}/uploads | display_name、declared_bytes、kind | 201；upload_id、asset_id、expires_at；预留配额，15分钟有效 |
| PUT /uploads/{id}/content | 二进制流；Content-Type；声明长度 | 202；Asset PENDING_SCAN；唯一内容写入，流式检查实际大小 |
| GET /uploads/{id} | 无 | 200；reservation状态、asset_id、实际接收是否完成及expires_at；仅原内部空间可见 |
| GET /assets/{id} | 无 | 200；状态、错误码、大小和文件信息 |
| POST /assets/{id}/retry | 无 | 202；仅完整原件且可重试的 FAILED |
| DELETE /assets/{id} | expected_version_revision | 202；仅草稿、进入清理状态 |
| POST /assets/{id}/download | variant 为 original/preview | 200；url、expires_at；READY且有权限；preview仅图片 |

上传中断后客户端重新读取 upload 状态；完整已收取则不能再次覆盖，未完成的reservation释放后重新申请。上传API的expires_at是receive_expires_at的展示名，只约束接收凭据；PROCESSING预算不随15分钟过期。不可把重复PUT当作更换附件的入口。

## 客户审阅

| 方法与路径 | 输入 | 输出／限制 |
|---|---|---|
| POST /review/challenges | share_token、email | 202；统一提示、challenge_id；仅匹配邮箱实际发码 |
| POST /review/verify | share_token、challenge_id、code | 200；设置客户会话 Cookie；返回授权版本摘要 |
| GET /review/version | 客户会话 | 当前授权版本、文件清单和最终决定摘要 |
| GET /review/comments | cursor | 授权版本评论；不返回内部其他项目数据 |
| POST /review/comments | version_id、asset_id 可空、body | 201；version_id须匹配当前会话，本轮审阅中；纯文本 |
| POST /review/assets/{id}/download | variant | 200 短时 URL；附件必须在授权版本内 |
| POST /review/decisions | version_id、outcome、message、acknowledgement_version、expected_version_revision | 201 Decision；版本须与会话一致；确认需显式勾选说明 |
| GET /review/receipt | 无 | 200 固定快照；有决定且客户授权仍有效 |
| POST /review/logout | 无 | 204；撤销客户会话 |

## 内部反馈、记录、通知

GET /versions/{id}/comments 与 POST /versions/{id}/comments 分别读取和追加内部成员的公开文字回复；写入也仅限 IN_REVIEW。GET /versions/{id}/receipt 返回当前版本的固定记录；GET /versions/{id}/receipt/export 返回可下载 JSON。前端根据相同快照提供打印页，首版不生成服务端 PDF。GET /notifications 返回分页通知；POST /notifications/{id}/read 只能标记本人通知。

只读空间保留有效成员的附件下载与本人通知已读；用户主动重试失败附件仍是业务写入。内部/客户注销在各服务状态均可执行，会话已失效时幂等清除 Cookie，不以分享有效为前提，也不返回业务内容。

## 错误与幂等

| HTTP | code 示例 | 前端处理 |
|---|---|---|
| 400/422 | VALIDATION_ERROR、UNSUPPORTED_FILE | 定位字段；保留输入 |
| 401 | AUTH_REQUIRED、SESSION_EXPIRED | 重新验证，不丢失未提交文字 |
| 403 | FORBIDDEN、CSRF_FAILED、WORKSPACE_READ_ONLY | 解释权限或服务状态；不自动重试 |
| 404 | RESOURCE_NOT_FOUND | 不存在及跨空间访问统一表现 |
| 409 | REVISION_CONFLICT、ALREADY_DECIDED、DRAFT_EXISTS、STATE_CONFLICT、IDEMPOTENCY_CONFLICT | 刷新实际状态，让用户决定下一步 |
| 410 | SHARE_UNAVAILABLE | 链接失效页，不披露项目名称 |
| 413 | FILE_TOO_LARGE | 显示上限并重新选文件 |
| 429 | RATE_LIMITED、QUOTA_EXCEEDED | 返回 retry_after 或额度说明 |
| 503 | SERVICE_UNAVAILABLE | 显示可重试，决定请求先查询真实结果 |

创建项目/版本/上传预留、发布、重发分享、评论、决定要求 Idempotency-Key。幂等范围是身份+路由+键；同键同内容返回原业务结果，同键不同内容返回409。重放发布或分享返回200及 share_id、secret_replay_available=false，原密钥只首次返回；用户确需链接时显式重发并撤销旧授权。服务不能以重试请求静默产生新版本、新评论或第二个决定。

幂等占位、业务数据和无密钥结果引用同事务提交。并发同键等待原事务（受请求超时约束）；若等待超时返回503并要求同键重试，不再执行一份业务写入。重放也校验当前权限，删除或撤销后的缓存不可返回受限内容。READ_ONLY按业务动作判断，不按POST/GET区分；验证、下载签发、注销及保护性撤销遵循权限文档动作表。

契约验收：T01 补齐请求/响应示例和正式模式文件是后续开发任务；字段、错误与用例先以本文为基线。任何新增端点都需注明鉴权、作用域、事务和审计策略。
