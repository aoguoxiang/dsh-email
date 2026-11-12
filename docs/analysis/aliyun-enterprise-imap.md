# 阿里企业邮箱接入分析与截图功能可行性评估

- 分析对象：`dsh-email@0.25.0`（当前仓库 `main`，提交 `2fe2dd6`）
- 分析日期：2026-11-11
- 范围：**只分析、给方案，不修改源代码**
- 截图目标：通过 IMAP 只读接入阿里企业邮箱，实现每日简报、重要邮件分类、外语翻译与回复建议、历史邮件智能检索、超期未回复提醒、日程提取

> 前提假设：企业邮箱为标准版/尊享版（非私有化定制），管理员可调整三方客户端与 IMAP 策略；这里的“只读”指客户端行为只读，不是阿里侧提供的只读权限。若企业已强制 SSO 并禁用标准协议，结论需要按实际策略重新评估。

---

## 0. 结论摘要

### 0.1 接入结论

**能接，但不是当前设置页“选择阿里邮箱”就能接。**

当前项目把 `aliyun` 预设指向阿里云**免费个人邮箱**的 `imap.aliyun.com` / `smtp.aliyun.com`；阿里企业邮箱官方要求的是 `imap.qiye.aliyun.com` / `smtp.qiye.aliyun.com`（不同地域/版本还有各自的域名）。同时：

- 设置页的内置服务商只有 `aliyun`，没有“阿里企业邮箱”；
- IMAP 账号卡片只保存 `provider`，**没有直接的 IMAP/SMTP 主机输入框**，要改端点必须走“服务器预设”或手写 YAML；
- 阿里企业邮箱新购默认**禁止第三方客户端**，管理员未放行时 IMAP 登录会被拒绝；
- 如果企业启用了“三方客户端安全密码”，必须用 16 位安全密码，不能用邮箱登录密码；
- 项目没有账号级“只读”能力，`email_send`、`email_mark`、文件夹增删改等写工具仍然注册；
- 项目要求 IMAP 账号必须配置 SMTP 主机，即使只想读信；
- 登录失败时只提示“检查邮箱地址与授权码”，无法区分租户策略、IP 白名单、安全密码等阿里侧原因。

所以“接不上”通常不是 IMAP 协议不兼容，而是**端点选错 + 企业侧策略未放行 + 设置页没有企业版入口**三件事叠加。

### 0.2 截图功能结论

**接入只解决“能读信”，不等于自动拥有截图里的智能能力。**

| 截图功能 | 现状判断 | 说明 |
|---|---|---|
| IMAP 只读接入 | 🟡 部分具备 | 读路径用 EXAMINE，但账号仍可发信/改标记/建删文件夹，无账号级只读开关 |
| 每日邮件简报 | 🟡 可用 DSH 定时任务拼出 MVP | 插件本身没有调度器；`email_watch` 文档明确建议“由定时任务周期性调用” |
| 重要邮件分类 | 🟡 只能做一次性模型分类 | 无持久标签、规则库、反馈与评估；`mail-task-rules` 是任务分发规则，不是邮件分类 |
| 外语翻译及回复建议 | ✅ 组合能力最成熟 | `email_read` 取正文，模型翻译/起草，`email_draft` 落本地草稿；发送仍需确认 |
| 历史邮件智能检索 | ❌ 不具备“智能/全量”检索 | 只有服务端 subject/from/to/cc 搜索 + 有界本地扫描，无全文索引、无语义检索、无跨文件夹检索 |
| 超期未回复提醒 | ❌ 缺少线程与回复状态 | `email_read` 不返回 Message-ID/References，列表也没有线程信息；无法可靠判断“已回复/未回复” |
| 日程提取 | ❌ 没有日历通道 | 可从正文提取文字建议，但没有 .ics 解析、CalDAV/日历 API 写入 |

一句话：**接入后可以做出“演示版”，但只有“翻译 + 回复建议”是现有工具的直接组合；“每日简报”可借 DSH 调度做 MVP；其余四项需要新增模块。**

---

## 1. 项目现状（读代码）

### 1.1 连接与账号模型

项目是一个 DSH 插件，核心链路是：

- `imapflow` 连 IMAP，`nodemailer` 发 SMTP；
- `src/config.ts` 解析账号、服务商预设、端点；
- `src/mail-client.ts` 负责 IMAP 连接池、列表、读取、搜索、标记、移动、文件夹；
- `src/tools.ts` 注册 `email_*` 工具；
- `src/web.ts` + `templates/client.js` 提供设置页和工作台；
- `src/draft-web.ts` / `src/draft-store.ts` 提供本地草稿与确认发送；
- `src/mail-tasks.ts` 提供“邮件触发 DSH 任务”，不是邮件智能。

账号配置里没有“只读”字段，也没有“能力集”字段：

- [`src/config.ts`](../../src/config.ts#L95-L146)：`AccountConfig` 只有 provider/user/password/authUser/authKind/imap/smtp/proxy 等；
- [`src/config.ts`](../../src/config.ts#L148-L176)：`EmailConfig` 的全局策略也只有 `sendApproval`、`allowTrustedReply` 等，没有 `readOnly`；
- [`src/settings.ts`](../../src/settings.ts#L13-L42)：设置 schema 同样没有只读开关。

### 1.2 服务商预设：`aliyun` 是个人版

[`src/config.ts`](../../src/config.ts#L202-L212) 的内置预设：

```ts
aliyun: {
  imap: { host: 'imap.aliyun.com', port: 993, secure: true },
  smtp: { host: 'smtp.aliyun.com', port: 465, secure: true },
}
```

对照官方文档：

- 阿里云**免费个人邮箱**：IMAP `imap.aliyun.com:993`、SMTP `smtp.aliyun.com:465`；
- 阿里**企业邮箱**：IMAP `imap.qiye.aliyun.com:993`、SMTP `smtp.qiye.aliyun.com:465`。

所以 `provider: aliyun` 在企业账号上属于**服务商预设不匹配**。即使某些网络环境下个人版入口也能路由到同一套后端，也不应该依赖这种未文档化行为。

项目文档也只列了个人版这一行：

- [`docs/USAGE.md`](../USAGE.md#L150)：`aliyun | imap.aliyun.com:993 | smtp.aliyun.com:465`
- [`README.md`](../../README.md)：只写了“腾讯企业邮预设”，没有阿里企业邮箱预设。

### 1.3 设置页：没有企业版入口，IMAP 卡片没有主机覆盖

内置服务商列表在 [`templates/client.js`](../../templates/client.js#L163-L173)：

```js
const PROVIDERS = [
  ["qq", ...], ["exmail", ...], ["163", ...], ["126", ...],
  ["sina", ...], ["aliyun", ...], ["gmail", ...],
  ["outlook", ...], ["icloud", ...],
]
```

没有“阿里企业邮箱”。账号卡片的服务商下拉就是这张表 + `serverPresets` 自定义预设 + AgentMail（[`templates/client.js`](../../templates/client.js#L3178-L3206)）。

更关键的是：**IMAP 账号卡片不显示主机输入框**。卡片只显示服务商、地址、密码、认证方式、代理等（[`templates/client.js`](../../templates/client.js#L4158-L4300)）；只有 POP3 账号才显示 POP3/SMTP 主机输入。账号保存时也只写 `provider`，端点由后端从预设表展开（[`src/web.ts`](../../src/web.ts#L546-L570)）。

因此，企业用户在当前 UI 里只有三条路：

1. 选“阿里邮箱”，但端点被解析成个人版 `imap.aliyun.com`；
2. 先在“服务器预设（高级）”里新建一个 `aliyun-enterprise` 预设，再在账号卡片里选它；
3. 直接手写 YAML（`accounts` + `imap`/`smtp`，或 `serverPresets`）。

第 1 条是默认路径，也是失败路径；第 2/3 条对普通用户门槛偏高。

### 1.4 认证：只支持密码，或 Microsoft/Google OAuth

[`src/config.ts`](../../src/config.ts#L117-L133) 和 [`src/oauth2.ts`](../../src/oauth2.ts#L1-L40) 显示：

- `authKind` 只有 `password` / `oauth2`；
- `oauthProvider` 只有 `google` / `microsoft`；
- OAuth2 是 Outlook/Microsoft 设备码流程，以及 Google 授权码 + PKCE；
- 没有阿里邮箱的 OAuth2 分支。

阿里企业邮箱的标准接法是“邮箱地址 + 三方客户端安全密码”（或未启用安全密码时的登录密码），所以**密码模式本身是兼容的**。不支持阿里 OAuth2 不是当前接不上的首要原因，但会影响“完全无密码/SSO 强制”的租户。

补充：阿里 IMAP 服务端确实宣告了 `AUTH=XOAUTH2`（见附录 A），但阿里公开文档里的 OAuth2 是**网页 SSO 身份源**和 **OpenAPI 的 client_credentials**，不是面向第三方 IMAP 客户端的公开应用注册流程。不要把“服务端支持 XOAUTH2”误判成“我们可以直接申请一个 IMAP OAuth 应用”。

### 1.5 读路径是只读的，但账号不是只读的

读操作默认用只读方式打开邮箱：

- [`src/mail-client.ts`](../../src/mail-client.ts#L514-L521)：`withImap(..., readOnly = true)`
- [`src/mail-client.ts`](../../src/mail-client.ts#L628-L636)：`mailboxOpen(folder, { readOnly })`

所以 `email_list`、`email_read`、`email_search`、`email_folders`、`email_attachment` 等读取不会改 `\Seen`，这是对的。

但同一个账号仍然注册了：

- `email_send` / `email_reply` / `email_schedule`（SMTP 发信）
- `email_mark`（已读/未读/星标/移动）
- `email_folder_create` / `rename` / `delete` / `purge`
- `email_draft`（本地草稿，但草稿页可确认发送）

见 [`src/tools.ts`](../../src/tools.ts#L74-L363)。`sendApproval` 只是“发信前确认”，不是权限开关，而且可以被关闭；`allowTrustedReply` 还能跳过部分确认。因此**当前无法承诺“只读接入”**。

另外，[`src/config.ts`](../../src/config.ts#L663-L664) 对 IMAP 账号强制要求 `smtp.host`：

```ts
} else if (imap.host === undefined || imap.host === '') problems.push(...)
if (receiveProtocol !== 'pop3' && (smtp.host === undefined || smtp.host === '')) problems.push(...)
```

这意味着即使只想读信，也必须填一个 SMTP 主机；对“管理员只开 IMAP、不开 SMTP”的企业账号，配置层会显得别扭（填了也不代表 SMTP 可用）。

### 1.6 搜索：关键词 + 有界扫描，不是“智能检索”

[`src/mail-client.ts`](../../src/mail-client.ts#L940-L995) 的搜索逻辑：

- 服务端只搜 `subject` / `from` / `to` / `cc` 四个字段（[`src/mail-client.ts`](../../src/mail-client.ts#L955-L960)）；
- 不搜正文 `TEXT`，注释明确说部分服务商会返回错误结果；
- 服务端结果还要用信封再确认；
- 失败后扫本地 header，默认最近 5000 封、最大 20000 封（[`src/mail-client.ts`](../../src/mail-client.ts#L998-L1025)）；
- 再失败才扫正文，默认只扫最近 30 封、最大 200 封（[`src/config.ts`](../../src/config.ts#L519)）；
- 每次只搜一个文件夹（`folderName = folder || cfg.inboxFolder`）；
- 没有全文索引、没有语义向量、没有跨文件夹聚合、没有相关性排序、没有线程聚合。

这套设计对“最近邮件关键词查找”是务实的，但离“历史邮件智能检索”差一个索引层。

### 1.7 数据模型：没有线程/回复状态

列表行只有 uid/date/from/subject/seen/flagged/size/hasAttachments：

- [`src/mail-client.ts`](../../src/mail-client.ts#L373-L384) 的 `listedFrom`

读取结果只有正文、地址、主题、附件等：

- [`src/mail-client.ts`](../../src/mail-client.ts#L1150-L1164)
- [`src/types.ts`](../../src/types.ts#L40-L64)

**没有返回 Message-ID / In-Reply-To / References。** 项目在“回复时”才会临时抓取这些 header（[`src/mail-client.ts`](../../src/mail-client.ts#L1392-L1408)），用于构造线程头，但没有把它们暴露给检索/跟进逻辑。

这直接导致“超期未回复提醒”无法可靠实现：既没有线程图，也没有“已回复”状态。

### 1.8 调度与智能能力

- 插件没有内置 cron/定时摘要；`email_watch` 的说明是“Call this periodically (e.g. via a scheduled task)”，即把调度交给宿主（[`src/tool-contract.ts`](../../src/tool-contract.ts#L385)）。
- `email_health` 只汇总配置，不测网络（[`src/tool-contract.ts`](../../src/tool-contract.ts#L383)）；设置页“测试连接”只测 IMAP，不测 SMTP（[`src/web.ts`](../../src/web.ts#L1192-L1196)）。
- 没有分类、摘要、翻译、日程、提醒等专用工具；这些只能由模型在会话里组合现有工具完成。

---

## 2. 为什么接不上企业版阿里邮箱

按“可复现的排查顺序”分五层。真正的原因通常在其中一层，而不是“IMAP 不支持”。

### 2.1 第一层：端点选错（项目侧，最常见）

| 产品 | IMAP | SMTP |
|---|---|---|
| 阿里云免费个人邮箱 | `imap.aliyun.com:993` | `smtp.aliyun.com:465` |
| 阿里企业邮箱（标准版/中国大陆） | `imap.qiye.aliyun.com:993` | `smtp.qiye.aliyun.com:465` |
| 阿里企业邮箱（中国香港） | `imaphk.qiye.aliyun.com:993` | `smtphk.qiye.aliyun.com:465` |
| 阿里企业邮箱（新加坡） | `imap.sg.aliyun.com:993` | `smtp.sg.aliyun.com:465` |
| 阿里企业邮箱（美国） | `imap.us.alibabacloud.com:993` | `smtp.us.alibabacloud.com:465` |
| 阿里企业邮箱（德国） | `imap.de.alibabacloud.com:993` | `smtp.de.alibabacloud.com:465` |
| 信创版 | `imap.xc.aliyun.com:993` | `smtp.xc.aliyun.com:465` |
| 旧版地址 | `imap.mxhichina.com:993` | `smtp.mxhichina.com:465` |

来源：阿里云官方《IMAP/POP/SMTP服务器地址及端口配置》与《阿里云免费个人邮箱IMAP、POP、SMTP地址和端口信息》。

**项目的行为**：选 `provider: aliyun` → `imap.aliyun.com`。企业账号被送到个人版入口，登录失败是可预期的。设置页又没有企业版选项，普通用户不会想到去建自定义预设。

### 2.2 第二层：企业侧默认禁止第三方客户端（阿里侧，最常见）

阿里企业邮箱官方文档明确：

> 新购买的阿里企业邮箱默认有一条开启的“三方客户端访问策略”黑名单策略，不允许登录第三方客户端。管理员需到「安全管理 → 账号安全 → 访问策略」把账号加入例外或关闭策略，**5 分钟后生效**。

在策略生效时，客户端看到的就是“认证失败/无法连接服务器/用户名密码错误”。项目的测试连接只会把它归为“邮箱登录失败：请检查邮箱地址与授权码”，于是用户会一直怀疑密码，实际是租户策略。

### 2.3 第三层：IMAP 未开 / 安全密码未用 / IP 白名单

三个独立的前置条件：

1. **IMAP 服务协议**：管理员需在「组织与用户 → 员工账号 → 功能权限 → 客户端」为账号开启 IMAP（以及 SMTP）。
2. **三方客户端安全密码**：如果管理员启用了“强制启用安全密码”，客户端不能用邮箱登录密码，必须用网页端生成的 **16 位三方客户端安全密码**；开启策略的瞬间，旧密码立即失效。密码只显示一次，丢了只能重新生成。
3. **IP 白名单**：访问策略可以限制 IP 范围；DSH 跑在办公室外、家里或云主机上时，出口 IP 不在白名单就会失败。项目支持账号级 `proxy`，但代理的出口 IP 也必须被阿里放行。

这三项都不会被项目的错误提示区分出来。

### 2.4 第四层：区域 / 版本 / 域名

- 企业邮箱可能在中国香港、新加坡、美国、德国等站点，必须用对应区域的服务器；用中国大陆的 `qiye.aliyun.com` 去连海外站点，可能超时或认证失败。
- 信创版是 `*.xc.aliyun.com`，不是 `qiye.aliyun.com`。
- 如果企业使用自定义域名作为服务器地址（如 `imap.company.com`），需要在阿里后台配置 SSL 证书；否则 Node 的 TLS 校验会失败。项目没有“跳过证书校验”的选项（这是对的），只能改用官方域名或修好证书。
- 旧企业域名可能仍在 `mxhichina.com` 后缀上。

### 2.5 第五层：协议本身不是主要矛盾

我们对两个入口做了**无凭据**探测（见附录 A）：

- `imap.aliyun.com:993` 与 `imap.qiye.aliyun.com:993` 都能建立 TLS 1.3，证书有效；
- 问候语都是 `* OK AliYun IMAP Server Ready`；
- `CAPABILITY` 都是 `IMAP4rev1 IDLE XLIST UIDPLUS ID SASL-IR AUTH=XOAUTH AUTH=XOAUTH2 AUTH=EXTERNAL`；
- 官方 IMAP 示例使用标准 `LOGIN`，不要求先发 `ID` 命令；
- 服务端支持 `UIDPLUS`，项目依赖的 UID 语义可用。

所以“Node/imapflow 连不上阿里 IMAP”这个假设基本可以排除。**如果错误是 `LOGIN failed`/`Command failed`，优先查端点、三方客户端策略、安全密码、IP 白名单，而不是换库。**

### 2.6 第六层：只读没有工程化保证

即使登录成功，“只读接入”在当前项目里也只是**调用习惯**：

- 读取用 EXAMINE，不会改标记；
- 但同一个密码可以 SMTP 发信，也可以 STORE/COPY/EXPUNGE；
- 插件注册了发信、标记、移动、文件夹增删改工具；
- `sendApproval` 可关闭，`allowTrustedReply` 可跳过确认；
- 没有账号级 `readOnly`，也没有 UI 隐藏写操作。

如果“只读”是合规要求，仅靠提示词/审批不够；需要服务端禁 SMTP、客户端能力开关，甚至一个只放行读命令的代理。

### 2.7 诊断树（建议按此顺序排查）

```text
测试连接失败
├─ ENOTFOUND / ETIMEDOUT / ECONNREFUSED
│  ├─ 主机名拼错 / 用了个人版入口
│  ├─ 区域选错（大陆 vs HK/SG/US/DE）
│  └─ 防火墙/代理未放行 993
├─ TLS 证书错误
│  ├─ 自定义域名未上传证书
│  └─ 中间代理做了 TLS 拦截
├─ LOGIN failed / Command failed
│  ├─ 企业策略禁止三方客户端
│  ├─ IMAP 协议未开
│  ├─ 用了登录密码而不是三方客户端安全密码
│  ├─ IP 不在白名单
│  └─ 账号被禁用/密码过期
├─ 登录成功但文件夹报 NONEXISTENT
│  ├─ 文件夹路径写错（中文名/层级）
│  └─ 已发送/废纸篓的 special-use 未被识别
├─ 登录成功但搜索为空
│  ├─ 服务端 SEARCH 不支持该关键词/编码
│  └─ 本地扫描窗口没覆盖到
└─ 测试连接成功但发信失败
   ├─ SMTP 主机/端口错（阿里 587 未开通，用 465）
   ├─ 管理员关闭了 SMTP
   └─ 只读场景本来就不需要 SMTP
```

---

## 3. 截图功能可行性逐项分析

### 3.1 IMAP 只读接入

**现状**：读路径是只读的，但账号不是只读的。

- ✅ `email_list` / `email_read` / `email_search` / `email_folders` / `email_attachment` 用 EXAMINE；
- ❌ 没有 `readOnly` 配置；发信/标记/移动/文件夹工具仍然可用；
- ❌ IMAP 账号强制要求 SMTP 主机；
- ❌ 错误提示不区分租户策略。

**判断**：可以“以只读方式使用”，但不能“保证只读”。要做成产品能力，需要新增账号级能力开关，并在服务层而不是提示词层拒绝写操作。

### 3.2 每日邮件简报

**现状**：插件没有调度器，但工具足够拼出 MVP。

可行路径：

1. 用 DSH 的定时任务（`schedule_create`）每天固定时间给会话发一条提示；
2. 模型调用 `email_list`（since/until 取过去 24 小时）+ `email_read`（读重点邮件）；
3. 模型按“重要/待办/资讯/垃圾”分组，输出简报；
4. 只读场景下不调用 `email_send`，简报直接发在会话/通知里。

限制：

- DSH 必须在定时点运行；错过时间不会自动补；
- 没有内置去重、历史归档、投递回执；
- `email_watch` 只适合“新未读”游标，不适合每日全量简报（第一次调用只建基线，且不会覆盖已读但当天新到的邮件）；
- 要发“简报邮件”给用户，就需要另一个发信账号/通道，不能算纯只读。

**判断**：MVP 可做，产品化需要调度 + 索引 + 去重。

### 3.3 重要邮件分类

**现状**：没有分类器，也没有标签存储。

- `mail-task-rules` 支持 subject/body/from/to/cc 的 contains/equals/regex，最多 8 条规则；但它的用途是“匹配后创建 DSH 任务”，不是给邮件打标签/分级（[`src/mail-task-rules.ts`](../../src/mail-task-rules.ts#L1-L40)）。
- 模型可以在每次简报里临时判断“重要/不重要”，但没有持久化、没有反馈、没有评估集，无法保证一致性。
- 阿里 IMAP 的 `PERMANENTFLAGS` 示例只有 `\Answered \Seen \Deleted \Draft \Flagged`，没有自定义 keyword；如果要在服务端做分类，只能建文件夹/移动邮件——这与“只读”冲突。
- 因此分类更适合做**本地标签**（数据库），而不是改服务器状态。

**判断**：可以做“一次性模型分类/简报分组”，但要做成稳定功能，需要规则库 + 本地标签 + 反馈闭环。

### 3.4 外语翻译及回复建议

**现状**：这是现有工具最接近截图的一项。

- `email_read` 取纯文本正文（HTML 会转换），默认上限 20000 字符（可配到 200000）；
- 模型翻译/起草回复；
- `email_draft` 保存本地草稿，不发送；
- `email_reply` 可在确认后发送，但只读场景应禁用；
- `replyLocale` / `replyTimeZone` 控制引用语言和日期时区；
- 附件只给元数据，PDF/图片/Office 文档内容不会被解析——如果外语内容在附件里，需要额外的文本提取。

**判断**：可直接做“翻译 + 回复建议（本地草稿）”；发送必须作为另一个显式动作，不应混在只读模式里。

### 3.5 历史邮件智能检索

**现状**：关键词 + 有界扫描，不是“智能检索”。

- 服务端只搜 subject/from/to/cc；
- 正文只在最近 30 封（默认）里做 fallback；
- header 扫描默认 5000 封，最大 20000 封，可用 `headerScanOffset` 往更早翻；
- 单文件夹、无跨文件夹聚合、无排序/相关性、无线程聚合；
- 阿里 IMAP 的 `CAPABILITY` 没有 `UTF8=ACCEPT`/`SORT`/`THREAD`/`ESEARCH`，中文搜索和复杂查询更依赖本地 fallback。

**判断**：对“最近几个月、记得关键词”的检索够用；对“多年历史、语义找信、跨文件夹”不够。要做截图里的“智能检索”，需要本地索引（FTS/向量）或阿里 OpenAPI（付费版）。

### 3.6 超期未回复提醒

**现状**：缺少线程与回复状态，做不准。

- 列表/读取都不返回 Message-ID / References；
- 没有“已回复”状态；
- 没有自动同步已发送文件夹；
- 没有 SLA/催办规则；
- 只能做“主题相似 + 发件人 + 时间”的启发式匹配，容易把 mailing list、自动回复、别人已回复的线程算错。

**判断**：可以做“粗略提醒”的 demo，但不建议直接承诺准确率。产品化需要：

1. 同步收件箱 + 已发送；
2. 抓取 Message-ID / In-Reply-To / References；
3. 建线程图；
4. 标记出站邮件；
5. 计算“最后一条入站之后是否有出站”；
6. 排除 noreply/newsletter/自动回复；
7. 定时扫描 + 人工确认。

### 3.7 日程提取

**现状**：没有日历通道。

- 没有 .ics 解析器；
- 没有 CalDAV/日历 API 客户端；
- `email_attachment` 可以把 .ics 下载到本地，但不会解析；
- 模型可以从正文里提取“时间/地点/参与人”，但只能输出文字建议，不能创建日程/提醒。

阿里企业邮箱本身提供 CalDAV 日历同步（官方文档给出新加坡/香港/德国/美国区域地址），也提供 OpenAPI 的日历能力（付费版，需管理员创建应用）。这些都需要在插件里新增连接器，不能靠 IMAP 完成。

**判断**：如果“日程提取”只是从正文抽取候选事件 → 可以做；如果要“写入日历/生成提醒” → 需要新模块。

---

## 4. 解决方案建议

### 4.1 第一步：先打通（0 代码，配置层）

#### 方案 A：自定义服务器预设（推荐，设置页可操作）

在「设置 → 邮件 → 服务器预设」新增：

```yaml
aliyun-enterprise:
  label: 阿里企业邮箱（标准版）
  imap: { host: imap.qiye.aliyun.com, port: 993, secure: true }
  smtp: { host: smtp.qiye.aliyun.com, port: 465, secure: true }
```

然后在账号卡片的服务商下拉里选“阿里企业邮箱（标准版）”，填写：

- 邮箱地址：完整企业邮箱地址；
- 密码：**三方客户端安全密码**（不是网页登录密码）；
- 收信协议：IMAP；
- 代理：如果出口 IP 不在阿里白名单，填能出白名单的代理。

#### 方案 B：YAML 显式端点（适合 headless/多账号）

```yaml
- id: tool-email
  config:
    accounts:
      work:
        user: me@corp.example
        password: '三方客户端安全密码'
        imap: { host: imap.qiye.aliyun.com, port: 993, secure: true }
        smtp: { host: smtp.qiye.aliyun.com, port: 465, secure: true }
    defaultAccount: work
```

注意：不写 `provider` 时，账号卡片在 UI 里可能显示为“自定义/未指定”，但 YAML 解析是支持的（[`src/config.ts`](../../src/config.ts#L621-L632) 优先用账号自己的 host）。

#### 方案 C：区域/版本覆盖

按企业实际站点替换：

- 香港：`imaphk.qiye.aliyun.com` / `smtphk.qiye.aliyun.com`
- 新加坡：`imap.sg.aliyun.com` / `smtp.sg.aliyun.com`
- 美国：`imap.us.alibabacloud.com` / `smtp.us.alibabacloud.com`
- 德国：`imap.de.alibabacloud.com` / `smtp.de.alibabacloud.com`
- 信创：`imap.xc.aliyun.com` / `smtp.xc.aliyun.com`
- 旧版：`imap.mxhichina.com` / `smtp.mxhichina.com`

#### 管理员侧检查清单（必须逐项确认）

- [ ] 已在「安全管理 → 账号安全 → 访问策略」把该账号加入三方客户端例外，或关闭默认黑名单策略；
- [ ] 已为该账号开启 IMAP（如要发信再开 SMTP）；
- [ ] 如启用“强制安全密码”，已生成 16 位三方客户端安全密码；
- [ ] 出口 IP 在白名单内（或已通过代理走白名单出口）；
- [ ] 确认企业站点区域，使用对应服务器；
- [ ] 如用自定义域名，已上传有效 SSL 证书；
- [ ] 确认 SMTP 用 465（阿里未开通 587/80）。

#### 验证顺序

1. `openssl s_client -connect imap.qiye.aliyun.com:993` 看 TLS/证书；
2. 设置页“测试连接”（只测 IMAP）；
3. `email_folders` 看 `INBOX`、`已发送`、`已删除` 等路径；
4. `email_list` 读收件箱；
5. 只读场景不要测 `email_send`；
6. 如果要验证 SMTP，单独用 `email_send` 给自己的地址发一封，并确认管理员已开 SMTP。

### 4.2 第二步：产品化接入（小改，建议排期）

#### 4.2.1 增加阿里企业邮箱预设

- 新增 `aliyun-enterprise`（中国大陆标准版）到 `PROVIDER_PRESETS`；
- 在 `ProviderName`、`PROVIDERS`、文档、测试中同步；
- 区域版本不建议全部做成内置（会膨胀），用“一个内置 + 文档化自定义预设”即可；
- 预设 label 用“阿里企业邮箱（标准版）”，不要和现有“阿里邮箱”混淆。

#### 4.2.2 让 IMAP 账号可以手填主机

这是比阿里适配更通用的能力：

- 账号卡片增加“高级：手动指定 IMAP/SMTP 主机/端口/TLS”；
- 或者在服务商下拉增加“自定义服务器”（`provider: ''` + host 输入）；
- 保存时不要把预设端点写死进 YAML；继续遵循“账号只存 provider，端点从预设展开”的现有设计，手填时再显式写 imap/smtp；
- 这样所有自建/企业邮箱都能用，不用为每个服务商加预设。

#### 4.2.3 增加“阿里企业邮箱”诊断提示

登录失败时，除了服务端原始错误，追加一段阿里专用检查清单：

> 阿里企业邮箱常见原因：①管理员未放行三方客户端；②未开启 IMAP；③使用了登录密码而非三方客户端安全密码；④出口 IP 不在白名单；⑤使用了个人版入口 `imap.aliyun.com` 而非 `imap.qiye.aliyun.com`。

不能从 `LOGIN failed` 精确区分具体原因，但可以把排查顺序固定下来，减少“反复试密码”。

#### 4.2.4 增加账号级只读能力

建议设计（仅方案，不实现）：

```ts
interface AccountConfig {
  /** 只读账号：禁止 SMTP 发信、标记、移动、文件夹增删改。 */
  readOnly?: boolean
}
```

落地点：

- `resolveAccount`：只读账号允许 `smtp.host` 为空；`smtp` 解析为空或仅占位；
- `EmailService` / `EmailPool`：增加 `assertWritable(account)`；`send`、`mark`、`createFolder`、`renameFolder`、`deleteFolder`、`purgeFolder` 在只读账号上直接拒绝；
- `email_send` / `email_reply` / `email_schedule` / `email_mark` / `email_folder_*`：运行时检查，不只改描述；
- `email_draft`：允许（本地草稿，不发信），但草稿页对只读账号隐藏“确认发送”；
- `email_attachment`：允许（只写本地文件）；
- `email_health`：显示“只读，SMTP 未配置/不启用”；
- 设置页：账号卡片加“只读”开关和徽标；
- 测试：增加“只读账号拒绝写操作”的回归用例。

#### 4.2.5 让 SMTP 对只读账号可选

当前 [`src/config.ts`](../../src/config.ts#L664) 强制 IMAP 账号填 SMTP，建议改成：

- `readOnly: true` 时允许 `smtp.host` 为空；
- `email_send` 在缺少 SMTP 时给明确错误：“该账号为只读/未配置 SMTP”；
- 不要用“SMTP host 必填”来伪装“可发信”。

#### 4.2.6 认证方式：先把密码模式做稳，OAuth2 不要过度承诺

- 阿里企业邮箱当前最稳妥的接法是“邮箱地址 + 三方客户端安全密码”；
- 阿里 IMAP 虽宣告 `AUTH=XOAUTH2`，但公开文档没有面向第三方 IMAP 客户端的 OAuth 应用注册流程；
- 如果企业强制 SSO/禁用密码，先确认管理员是否仍允许“安全密码 + 标准协议”；
- 真要做 OAuth2，优先评估阿里 OpenAPI（付费版，client_credentials + Mail/Calendar scope），而不是猜 IMAP XOAUTH2。

#### 4.2.7 代码落点（建议，不实施）

| 文件 | 改动 |
|---|---|
| [`src/config.ts`](../../src/config.ts) | 新增 `aliyun-enterprise` 预设；`AccountConfig.readOnly`；只读时允许 SMTP 为空 |
| [`templates/client.js`](../../templates/client.js) | 服务商下拉增加企业版；IMAP 卡片增加“高级主机”或“自定义服务器”；只读徽标/隐藏写操作 |
| [`src/web.ts`](../../src/web.ts) | 卡片序列化支持 `readOnly` 与显式 host；测试连接返回分阶段结果与阿里检查清单 |
| [`src/mail-client.ts`](../../src/mail-client.ts) | 写操作入口统一 `assertWritable`；列表/读取补 Message-ID/References（为线程/超期做准备） |
| [`src/tools.ts`](../../src/tools.ts) | 只读账号的写工具在运行时拒绝；新增摘要/分类/跟进/日程工具时按能力注册 |
| `src/mail-index.ts`（新） | 本地索引、增量同步、FTS/线程图、标签与回复状态 |
| `src/mail-digest.ts`（新） | 每日简报生成、去重、分组 |
| `src/calendar.ts`（新） | .ics 解析、CalDAV/OpenAPI 日历连接器 |

### 4.3 第三步：智能层（新建，决定截图功能能否落地）

建议把“邮件智能”做成独立于 IMAP 的本地层，IMAP 只负责同步：

```text
IMAP 只读同步 ──> 本地索引（消息 / 线程 / 标签 / 回复状态）
                     ├─ 检索（FTS + 可选向量）
                     ├─ 分类（规则 + 模型）
                     ├─ 超期（线程图 + 出入站）
                     └─ 日程（正文 + .ics + CalDAV/API）
                              │
                     DSH 定时任务 / 会话
                              │
                     简报 / 提醒 / 回复建议草稿
```

核心模块：

#### 4.3.1 本地邮件索引（历史检索 + 超期跟踪的地基）

最小数据模型：

```text
messages(
  account, folder, uid, uidvalidity,
  message_id, in_reply_to, references,
  date, from, to, cc, subject,
  size, has_attachments, flags,
  text_snippet,
  labels, importance, reply_state,
  first_seen_at, last_synced_at
)
```

实现建议：

- 增量同步：按 `UIDVALIDITY` + `UIDNEXT` 拉新；首次回填分批；
- 存储：SQLite FTS5（trigram，适合中文）或纯 JS 倒排索引；避免强制原生依赖；
- 检索：FTS + 查询扩展（模型生成同义词/英文缩写）+ 可选向量检索；
- 线程：用 Message-ID / References / In-Reply-To 建图；
- 隐私：本地存储、可配置保留期、可选加密；不要默认把全文上传到模型。

#### 4.3.2 每日简报

- 数据源：索引里“过去 24 小时入站 + 未读/重要”；
- 分类：规则（VIP/关键词/直接收件人/截止日期）+ 模型；
- 输出：重要/待办/资讯/垃圾四组 + 行动建议；
- 调度：优先用 DSH 定时任务；插件内不做常驻 cron，避免宿主休眠/重启问题；
- 去重：按 message-id 记录已汇报；
- 投递：会话通知；如果要发邮件，用独立通道，不要破坏只读。

#### 4.3.3 重要邮件分类

- 第一阶段：规则（发件人/域名/关键词/是否直接 To/是否含截止日期/是否在等回复）；
- 第二阶段：模型打分 + 结构化输出（importance 1-5、category、reason、confidence）；
- 持久化：本地 labels，不改服务器 flags；
- 反馈：用户“标为重要/不重要”写回本地，作为后续规则/提示；
- 只读：不移动邮件、不加服务器标签。

#### 4.3.4 超期未回复提醒

- 同步收件箱 + 已发送；
- 建线程图；
- 对每个线程计算：
  - 最后一条入站时间；
  - 最后一条出站时间；
  - 最后一条是否是入站；
  - 是否命中“无需回复”规则（noreply、newsletter、自动回复、已归档）；
- 超过 SLA（如 2 个工作日）→ 生成提醒；
- 只读场景：只提醒，不自动回复、不移动；
- 输出：发件人/主题/等待天数/线程链接/建议动作。

#### 4.3.5 日程提取

- 从正文/HTML 提取日期、时间、时区、地点、参与人；
- 解析 .ics 附件（`email_attachment` 下载后解析）；
- 输出结构化候选事件 + 置信度；
- 写入：
  - 首选 CalDAV（阿里企业邮箱支持，凭据与邮箱安全密码一致）；
  - 或阿里 OpenAPI 日历（付费版，管理员创建应用）；
  - 写入前必须人工确认，避免把邮件里的“会议纪要时间”误建成日程；
- 只读场景：只输出候选，不写日历。

#### 4.3.6 只读的“硬保证”

如果合规要求“绝不能写”，仅靠插件开关不够，因为 IMAP 密码本身有写权限。建议叠加：

1. **服务端**：管理员只开 IMAP、不开 SMTP；能关 SMTP 就关；
2. **客户端**：`readOnly` 能力 + 工具层拒绝 + UI 隐藏；
3. **网络层**：IMAP 代理只放行 `CAPABILITY/LOGIN/SELECT/EXAMINE/LIST/LSUB/STATUS/FETCH/SEARCH/UID/LOGOUT`，阻断 `STORE/COPY/MOVE/APPEND/EXPUNGE`；SMTP 直接阻断；
4. **凭据**：专用安全密码、定期轮换、离职/换机可单独删除；
5. **审计**：记录每次读操作；对写操作告警。

第 3 条是唯一能对“客户端之外”也成立的硬保证；如果企业允许，优先做网络层。

### 4.4 分期路线图（建议）

| 阶段 | 目标 | 交付 |
|---|---|---|
| P0（1-2 天） | 让企业账号能连上 | 自定义预设/YAML + 管理员清单 + 诊断树 |
| P1（1-2 周） | 产品化接入 | `aliyun-enterprise` 预设、IMAP 主机手填、阿里诊断提示、`readOnly` + SMTP 可选 |
| P2（2-4 周） | 截图 MVP | 本地索引、每日简报、规则分类、翻译/草稿建议 |
| P3（4-8 周） | 截图完整版 | 线程/超期提醒、语义检索、CalDAV/日历、反馈闭环 |

原则：**先规则后模型，先本地后服务端，先建议后写入。**

---

## 5. 风险与边界

1. **租户策略不可控**：即使代码全对，管理员不放行三方客户端/不开 IMAP/强制 SSO，仍然连不上；这不是插件能修的。
2. **IMAP 不是稳定 API**：不同区域/版本的能力、文件夹命名、搜索行为可能不同；不能假设 `SPECIAL-USE`/`SORT`/`UTF8=ACCEPT` 一定存在。
3. **只读不是服务端权限**：IMAP 没有“只读密码”；客户端只能约束自己，不能约束密码的其他用途。
4. **隐私与合规**：本地索引会存储邮件正文/元数据；模型调用会把内容送出本机。需要脱敏、保留期、访问控制、企业审批。
5. **“智能”准确率**：分类、超期判断、日程提取都是概率性的；必须给置信度、人工确认和纠错入口。
6. **成本与限流**：全量回填/每日摘要会消耗模型 token；IMAP 连接数/搜索频率也可能被服务端限制。
7. **付费版差异**：阿里 OpenAPI/日历能力在免费版不可用；CalDAV 区域地址也不同。方案要按企业版本降级。
8. **不要用“测试连接成功”代表“能发信”**：设置页只测 IMAP；SMTP 是独立配置。

---

## 6. 验收清单（建议直接用于 PoC）

### 6.1 连接

- [ ] 使用企业实际区域的官方主机；
- [ ] TLS 证书校验通过；
- [ ] IMAP 登录成功；
- [ ] `email_folders` 能看到 INBOX/已发送/已删除；
- [ ] `email_list` 能读最近邮件；
- [ ] `email_read` 打开一封 HTML 邮件，正文/附件元数据正确；
- [ ] 读信后网页端未读状态不变（验证 EXAMINE）。

### 6.2 只读

- [ ] 账号级只读开关生效；
- [ ] `email_send` / `email_reply` / `email_schedule` 被拒绝；
- [ ] `email_mark` / 文件夹增删改被拒绝；
- [ ] `email_draft` 可保存，但草稿页不能发送；
- [ ] 服务端 SMTP 已关闭（如可行）；
- [ ] 代理/防火墙阻断 SMTP 与 IMAP 写命令。

### 6.3 功能

- [ ] 每日简报：过去 24 小时、去重、分组、无发送；
- [ ] 分类：VIP/关键词/截止日期/等待回复四类，误报可反馈；
- [ ] 翻译：中英互译、保留原邮件引用、生成草稿；
- [ ] 历史检索：中文/英文关键词、跨文件夹、一年以上邮件、显示覆盖范围；
- [ ] 超期提醒：有回复/无回复/自动回复三种样本，准确率可评估；
- [ ] 日程：正文时间 + .ics 附件，时区正确，写入前确认。

---

## 附录 A：协议探测记录

2026-11-11，仅做无凭据探测，未登录任何账号：

```text
$ openssl s_client -connect imap.aliyun.com:993 -servername imap.aliyun.com
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Peer certificate: CN = mail.aliyun.com
Verification: OK
* OK AliYun IMAP Server Ready
* CAPABILITY IMAP4rev1 IDLE XLIST UIDPLUS ID SASL-IR AUTH=XOAUTH AUTH=XOAUTH2 AUTH=EXTERNAL

$ openssl s_client -connect imap.qiye.aliyun.com:993 -servername imap.qiye.aliyun.com
CONNECTION ESTABLISHED
Protocol version: TLSv1.3
Peer certificate: CN = mail.aliyun.com
Verification: OK
* OK AliYun IMAP Server Ready
* CAPABILITY IMAP4rev1 IDLE XLIST UIDPLUS ID SASL-IR AUTH=XOAUTH AUTH=XOAUTH2 AUTH=EXTERNAL
```

结论：

- 两个入口都可达，TLS 正常；
- 支持标准 LOGIN、UIDPLUS、IDLE、XLIST、XOAUTH2；
- 未宣告 `SORT` / `THREAD` / `ESEARCH` / `UTF8=ACCEPT` / `SPECIAL-USE` / `MOVE`；
- 官方示例不要求 `ID` 命令；
- `LOGIN` 失败的返回是笼统的 `LOGIN failed`，无法从协议层区分“密码错/策略禁止/IP 不允许”。

## 附录 B：参考资料

- 阿里云：IMAP/POP/SMTP 服务器地址及端口配置（企业邮箱）  
  https://help.aliyun.com/zh/document_detail/36576.html
- 阿里云：免费个人邮箱 IMAP/POP/SMTP 地址和端口  
  https://help.aliyun.com/zh/document_detail/465307.html
- 阿里云：配置第三方客户端访问控制与安全策略（默认禁止三方客户端）  
  https://help.aliyun.com/zh/document_detail/606337.html
- 阿里云：开启并设置三方客户端安全密码  
  https://help.aliyun.com/zh/document_detail/444269.html
- 阿里云：IMAP 命令集及 Python 示例  
  https://help.aliyun.com/zh/document_detail/2857156.html
- 阿里云：IMAP 命令行获取邮件信息（标准 LOGIN 示例）  
  https://help.aliyun.com/zh/document_detail/2857126.html
- 阿里云：手机自带日历及三方日历程序配置（CalDAV）  
  https://www.alibabacloud.com/help/zh/alibaba-mail/latest/iphone-calendar-configuration-method
- 阿里邮箱 API 开放平台（邮件/日历 API，免费版不支持）  
  https://mailhelp.aliyun.com/docs/api/alibaba-mail-api
- 阿里邮箱 OAuth 2.0 身份认证（网页 SSO，AI 尊享版）  
  https://help.aliyun.com/en/document_detail/2937171.html
