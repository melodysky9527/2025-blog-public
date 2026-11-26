想把密码、自动填充和多设备同步放到自己管理的服务里，又不想维护一台 VPS？NodeWarden 提供了一条基于 Cloudflare Workers 的部署路线：用 D1 保存结构化数据，用 R2 或 KV 保存附件，再连接 Bitwarden 官方客户端使用。

本文参考[周润发博客的 NodeWarden 教程](https://blog.zrf.me/p/NodeWarden-Bitwarden/)，结合项目当前文档重新整理，覆盖部署、域名、账号初始化、客户端连接、密码迁移、TOTP、通行密钥和远程备份。步骤依据公开资料核对，不代表本站已对所有客户端和恢复场景完成实测。

## 先了解：它能做什么，费用怎么算

NodeWarden 是兼容 Bitwarden 协议的第三方服务端，与 Bitwarden 官方没有隶属关系。按照[项目 README](https://github.com/shuaiplus/nodewarden#readme)，它提供网页密码库、密码同步、附件、Send、导入导出和 WebDAV/S3 备份，也支持多用户邀请码注册。组织、集合、成员权限以及 SSO 等企业功能未实现。

客户端兼容性也要看具体平台和版本。项目列出 Windows、Linux、移动端和浏览器扩展的测试情况，同时注明 macOS 桌面端尚未完整验证。正式迁移前，建议先用少量测试条目跑通自己的设备。

原教程所说的“零成本”，可以理解为利用 Cloudflare 免费额度运行，不需要租用 VPS。额度有容量、请求数和资源限制，域名及外部备份服务也可能收费，不能把它理解成无限使用、保证不扣费。

| 附件存储方式 | 部署命令 | 开通条件与限制 |
| --- | --- | --- |
| R2，默认方案 | `npm run deploy` | 项目文档注明需要绑卡；附件实际可上传大小同时受项目应用限制与 [Worker 请求体上限](https://developers.cloudflare.com/workers/platform/limits/) 约束。R2 Standard 有免费额度，超额按量计费。 |
| KV，可选方案 | `npm run deploy:kv` | 项目文档注明无需为此开通 R2；单个值受 Cloudflare 25 MiB 上限约束，另有容量和读写配额。 |

Cloudflare 各服务分别设有免费额度，其中部分账户级上限可能由同一账户多个项目共用，不是所有服务共享同一个额度池。以 Cloudflare 2026-11-26 当前价目为例，Workers Free 每日请求数有限，D1 对行读写和单库容量设限，KV 有每日读写与单值大小限制，R2 Standard 仅在免费存储量与操作配额内免额外费用。达到免费上限可能使请求、读取或写入失败，超过可计费项的配额也可能按账户方案收费。上线前查看 [Workers 定价](https://developers.cloudflare.com/workers/platform/pricing/)与[运行限制](https://developers.cloudflare.com/workers/platform/limits/)、[D1 定价](https://developers.cloudflare.com/d1/platform/pricing/)与[运行限制](https://developers.cloudflare.com/d1/platform/limits/) 和 [KV 限制](https://developers.cloudflare.com/kv/platform/limits/)，结合账户中其他项目的用量判断；不要按某次注册时的绑卡提示推断将来绝无费用。

[Cloudflare R2 定价页](https://developers.cloudflare.com/r2/pricing/)列出的 Standard 免费额度为每月 10 GB-month 存储、100 万次 A 类操作、1000 万次 B 类操作；不适用于 Infrequent Access。选择前同时查看自己的 Workers、D1、KV 和 Durable Objects 用量与套餐。

## 第一步：准备账号，Fork 项目

需要一个 GitHub 账号和一个 Cloudflare 账号。建议提前准备托管在 Cloudflare 的域名，后面给密码库绑定独立子域名，例如 `vault.example.com`。默认 `workers.dev` 地址在部分网络环境中可能无法直接访问，自定义域名的实际可用性也应现场验证。

打开 [NodeWarden 仓库](https://github.com/shuaiplus/nodewarden)，点击右上角 **Fork**，把项目复制到自己的 GitHub 账号。后续 Cloudflare 连接这个仓库构建部署。

如果 Cloudflare 找不到 Fork，先检查 GitHub 应用是否获准访问该仓库。项目 FAQ 还建议遇到识别或 404 问题时，尝试修改 Fork 的仓库名和描述，再重新连接。

## 第二步：在 Cloudflare 部署 Worker

进入 [Cloudflare Workers & Pages](https://dash.cloudflare.com/?to=/:account/workers-and-pages/create)，创建应用，选择 **Continue with GitHub**，授权并选中刚才的 Fork。

按项目[快速部署文档](https://nodewarden.app/zh/guide/quick-start)填写：

```text
构建命令：npm run build
部署命令：npm run deploy
```

默认使用 R2。如果选择 KV，将部署命令改为：

```text
npm run deploy:kv
```

这条命令会运行 [`scripts/ensure-kv.cjs`](https://github.com/shuaiplus/nodewarden/blob/main/scripts/ensure-kv.cjs)。仅当配置里尚无 KV 命名空间 ID 时，脚本才会使用 Cloudflare 授权查询现有命名空间，必要时创建一个，并把 ID 写入本次构建工作副本中的 `wrangler.kv.toml`；已有 ID 时会跳过这一步。确认 GitHub/Cloudflare 构建连接具备检查或创建 KV 命名空间及部署 Worker 所需的授权。

确认设置后部署，查看构建日志，等待完成。构建环境的 Node.js 版本应符合仓库当前的 `package.json` 和 `.node-version`；如果曾手动设置 `NODE_VERSION`，还要检查旧覆盖值是否影响构建。

这里有一个容易混淆的地方：**数据库资源绑定和数据库表初始化是两回事。** 项目会在首次后端请求时初始化 D1 schema，但这不代表任何绑定错误都能自动修好。部署后检查 Worker 的资源绑定：

| 绑定名 | 用途 |
| --- | --- |
| `DB` | D1 数据库，保存用户、密码库元数据和配置 |
| `ATTACHMENTS` | R2 附件与 Send 文件正文，R2 模式使用 |
| `ATTACHMENTS_KV` | KV 附件与 Send 文件正文，KV 模式使用 |
| `NOTIFICATIONS_HUB` | 实时通知所用的 Durable Object |
| `BACKUP_TRANSFER_RUNNER` | 备份与还原任务所用的 Durable Object |

这些名称由仓库配置决定，保留相应绑定和迁移配置。普通部署不需要手工上传 SQL；如果要复用旧实例，应确认仍连接原来的 D1 与附件存储。详细说明见[Cloudflare 参数文档](https://nodewarden.app/zh/guide/deployment/cloudflare)。

## 第三步：绑定域名，配置 JWT_SECRET

在 Worker 的 **设置 → 域和路由 → 自定义域** 中添加准备好的子域名，例如 `vault.example.com`，等待 DNS 和证书配置完成，再通过 HTTPS 打开。

如果页面提示缺少 `JWT_SECRET`，返回 **设置 → 变量和机密** 添加配置：

```text
类型：Secret／机密
名称：JWT_SECRET
值：安全生成的至少 32 个字符的随机字符串
```

注意添加的是 **Worker 运行时 Secret**，仅设置“构建变量”不能满足应用运行时读取要求。这个值应通过可信的密码生成器或本地安全随机工具生成，不要使用域名、生日、示例文字，也不要提交到 GitHub。`JWT_SECRET` 用于签发和验证 JWT 等令牌，也参与备份目标凭据的运行时加密；**不是解密密码库的主密码**。升级时保持这个值稳定，并单独安全保存；任意更换会影响会话及已有备份配置。参见[配置与密钥文档](https://nodewarden.app/zh/guide/deployment/configuration)。

## 第四步：创建管理员，开启两步验证

配置就绪后，打开网页密码库并注册账号。当前项目的**第一个注册用户会成为管理员**，后续注册默认需要邀请码。因此，部署完成后应及时完成首次注册。

设置一条独立、足够强且自己能保存好的主密码。根据[账号与主密码文档](https://nodewarden.app/zh/guide/security/accounts)，修改数据库中的登录校验值无法替你解密已有密码库，所以不要把“我能管理 Cloudflare”当作忘记主密码后的恢复手段。

登录后进入账户设置，按界面启用 TOTP 两步验证，用独立验证器扫码或手动录入设置密钥，输入动态验证码完成确认，再保存恢复代码。

这里要分清两类用途：**密码库账号的 TOTP 是进入密码库的门锁；条目里的 TOTP 是登录其他网站的验证码。** 不要只把密码库自身的验证码和恢复代码放在它自己里面，否则设备全部退出时可能无法取出。建议保留独立验证器和离线恢复资料。

## 第五步：连接 Bitwarden 官方客户端

从 [Bitwarden 官方下载页](https://bitwarden.com/download/)安装浏览器扩展或手机、桌面客户端。登录前找到服务器或地区选择入口，将默认服务器改为 **Self-hosted／自托管**。

服务器地址填写完整的 HTTPS 根地址，例如：

```text
https://vault.example.com
```

通常不需要自行添加 `/api` 或登录页面路径。保存后，用刚才注册的邮箱、主密码登录；启用了 TOTP 时再输入验证码。不同客户端的入口位置可能变化，可对照 [NodeWarden 客户端文档](https://nodewarden.app/zh/guide/core/clients)和 [Bitwarden 服务器设置说明](https://bitwarden.com/help/change-client-environment/)。

先创建一条测试登录记录，在另一台设备同步并检查。随后试一次自动填充、编辑与附件下载，确认自己的组合可用，再迁移重要数据。

## 第六步：迁移浏览器密码，核对导入结果

在 Chrome 或 Edge 的密码管理设置里找到导出功能，按系统提示验证身份并导出 CSV。浏览器版本不同，菜单位置可能不同，以当前密码管理页面为准。

将文件导入密码库时，选择与来源相符的格式。若当前 NodeWarden 页面没有对应浏览器格式，可先使用 Bitwarden 官方客户端支持的导入流程；不要把浏览器 CSV 直接当作 Bitwarden CSV。NodeWarden 文档列出 Bitwarden JSON、CSV、ZIP 等迁移格式，具体支持范围见[导入与导出文档](https://nodewarden.app/zh/guide/core/import-export)。

导入后先核对条目数量、网址、用户名和备注，抽查几个重要账号，再同步到第二台设备。重复条目应逐项比较后处理，不要在第一次导入时就批量删除旧数据。

**浏览器导出的密码 CSV 通常是明文敏感文件。** 迁移时避免放进公共共享目录或同步盘；确认导入无误后，清理临时文件及其回收站或云端副本。带附件迁移还要检查附件正文，不能只看密码条目已出现。

## 第七步：使用网站 TOTP 和通行密钥

对于支持验证器的第三方网站，在其安全设置中开启 TOTP，取得二维码或设置密钥，然后编辑 Bitwarden 中对应登录条目的“验证器密钥／TOTP”字段，保存并完成一次验证码验证。第三方网站提供的恢复代码也要另行保存。

Bitwarden 官方客户端自身有保存网站 Passkey 的使用说明（见[官方指南](https://bitwarden.com/help/storing-passkeys/)）；但 NodeWarden 文档说明的是服务端账号 Passkey 登录和第二因素，不能据此证明网站 Passkey 能存进 NodeWarden 密码库并在其上同步或自动填充。本文没有实测这一完整流程；在目标网站、客户端版本与设备上验证成功之前，不要把它当作已确认可用的功能，也不要仅靠此方式保护重要账号。

**给网站保存 Passkey，与用 Passkey 登录 NodeWarden 自身是两件事。** 后者另有配置和客户端条件，见[Passkey 登录文档](https://nodewarden.app/zh/guide/security/passkey-login)。不要仅因网页能打开，就认定所有通行密钥场景都兼容。

把网站密码和验证码放在同一密码库里很方便。对于邮箱、Cloudflare、GitHub 等能控制账号恢复或基础设施的重要账号，建议仍保留独立的验证器、备用安全密钥或离线恢复代码。

## 第八步：配置远程备份，再做恢复检查

NodeWarden 的备份中心支持 **WebDAV 和 S3 兼容存储**。参考教程使用 WebDAV，你也可以使用已有服务；服务地址、容量和费用应以自己的服务商后台为准。

在备份中心新增 WebDAV 目标，按界面填写：

```text
服务地址：服务商实际提供的 WebDAV HTTPS 地址
用户名：该服务的 WebDAV 用户名
密码：应用专用密码或服务商要求的凭据
远程目录：例如 nodewarden-backup
```

使用 S3 时则需要对应 Endpoint、Region、Bucket 和访问凭据。不要把别人的 WebDAV 主机名、账号或推荐码当作自己的配置。连接字段可对照[备份服务商文档](https://nodewarden.app/zh/guide/backup/providers)。

新建并配置备份目标后先保存。默认设置不包含附件（当前代码默认 `includeAttachments: false`，见[项目默认配置](https://github.com/shuaiplus/nodewarden/blob/main/shared/backup-schema.ts)）。如需备份附件，先在目标设置中主动启用“包含附件”并再次保存。随后手动运行一次备份，确认远程目标里确实出现备份，再设置自动运行时间和保留策略；下载备份并在独立测试实例中验证密码项与附件都能恢复，避免首次恢复就在正式密码库上覆盖数据。

还要区分**当前用户的密码库导出**与**管理员的整站备份**：前者用于用户迁移，后者涉及整站用户、配置及可选附件。恢复时如果从 R2 改到 KV，超过 KV 上限的附件可能被跳过，应检查结果列表。完整说明见[还原与校验文档](https://nodewarden.app/zh/guide/backup/restore)。

## 更新与常见问题

日常更新可在自己的 Fork 点击 **Sync fork → Update branch**。更新前先做备份，阅读上游变更；更新后重新验证登录、同步、附件和备份，不建议把密码库长期交给无人检查的自动更新。

| 遇到的问题 | 优先检查 |
| --- | --- |
| 构建失败 | Node.js 版本覆盖值、依赖安装和构建日志 |
| 一直提示缺少 JWT_SECRET | 是否加在正确 Worker 的运行时“变量和机密”中，是否保存部署，长度是否符合要求 |
| 默认域名打不开 | 当前网络环境、自定义域名、DNS 与证书状态 |
| 客户端登录失败 | 自托管 HTTPS 地址、账号信息、两步验证及该版本兼容性 |
| 重部署后仍要求邀请码 | 是否仍绑定旧数据库；删除 Worker 不会自动清空持久数据 |
| 备份有记录却无法完整还原 | 远程文件、附件是否纳入备份、目标存储限制和恢复结果 |

部署结束后，至少完成一次跨设备同步和一次备份恢复检查，再逐步迁入重要账号。后续维护重点是保管主密码与恢复资料、检查资源用量、验证更新，以及确认备份能够恢复。

## 来源与时效

核对日期：**2026 年 11 月 26 日（Asia/Shanghai）**。本文是同主题的重新编写与资料更新，部署界面、功能和免费额度可能继续变化，实际操作以当前项目及服务商说明为准。

- [周润发（周润发博客），《NodeWarden 部署教程：零成本搭建私有 Bitwarden 密码管理器，无需服务器，使用 Cloudflare》](https://blog.zrf.me/p/NodeWarden-Bitwarden/)：参考作品标注 CC BY-NC-SA 4.0；本文结合当前项目资料重新编写并更新，保留作者、标题、来源和许可证信息；未采用原作者的个人实测经历作为本站经历。
- [NodeWarden 项目 README](https://github.com/shuaiplus/nodewarden#readme)：项目定位、功能、兼容性及第三方身份说明。
- [快速部署](https://nodewarden.app/zh/guide/quick-start)与 [Cloudflare 参数](https://nodewarden.app/zh/guide/deployment/cloudflare)：构建命令、资源绑定、注册和运行时配置。
- [配置与密钥](https://nodewarden.app/zh/guide/deployment/configuration)、[账号与主密码](https://nodewarden.app/zh/guide/security/accounts)、[客户端连接](https://nodewarden.app/zh/guide/core/clients)、[Passkey 登录](https://nodewarden.app/zh/guide/security/passkey-login)与 [Bitwarden 网站 Passkey 说明](https://bitwarden.com/help/storing-passkeys/)：认证与功能边界。
- [导入与导出](https://nodewarden.app/zh/guide/core/import-export)、[备份服务商](https://nodewarden.app/zh/guide/backup/providers)、[还原与校验](https://nodewarden.app/zh/guide/backup/restore)：迁移与备份边界。
- [Workers 定价](https://developers.cloudflare.com/workers/platform/pricing/)与 [Workers 限制](https://developers.cloudflare.com/workers/platform/limits/)、[D1 定价](https://developers.cloudflare.com/d1/platform/pricing/)与 [D1 限制](https://developers.cloudflare.com/d1/platform/limits/)、[KV 限制](https://developers.cloudflare.com/kv/platform/limits/)和 [Cloudflare R2 定价](https://developers.cloudflare.com/r2/pricing/)：免费额度、平台限额与计费条件；核对时 R2 页面标注更新于 2026 年 10 月 30 日，该日期不代表产品首发日。

参考原教程标注的许可，本文按 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/deed.zh-hans) 共享，转载请注明本文与参考来源并遵循相同许可。所链接软件与第三方素材各自适用其原有许可。
