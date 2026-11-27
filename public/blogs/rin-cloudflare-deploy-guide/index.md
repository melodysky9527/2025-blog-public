导语：如果你想把 Rin 部署成自己的博客，但不熟悉 Cloudflare、GitHub Actions 和环境变量，这篇文章只讲官方部署指南中的首次部署流程：准备账号 → 复制仓库 → 填写 5 个 Secrets → 手动运行 Build → 等待 Deploy 完成 → 登录并发布第一篇文章。官方页面估计准备约 10–15 分钟、部署约 3–5 分钟；实际时间可能更长。

> 本文依据 [Rin 官方部署指南](https://docs.openrin.org/guide/deploy.html) 整理。文中出现的域名是官方示例，实际地址以你自己的部署结果为准。

![Rin 官方标识](https://github.com/OXeu/Rin/assets/36541432/df3607ca-a8a9-49b8-92ce-6b348afeb13f)

## 一、开始前准备

你需要准备：

- 一个 GitHub 账号；
- 一个 Cloudflare 账号，官方文档说明免费版即可；
- 能够保存好 API 令牌、账户 ID 和登录密码的地方；
- 约 10–15 分钟。

这套部署不要求你先在本地安装 Node.js、Bun 或数据库。首次部署的操作都在 GitHub 和 Cloudflare 网页中完成。

## 二、先把 Rin 复制到自己的 GitHub

1. 打开 [Rin GitHub 仓库](https://github.com/openRin/Rin)。
2. 点击右上角 **Fork**。
3. 选择你自己的 GitHub 账号，保持默认设置。
4. 点击 **Create fork**。

完成后，你会得到一个属于自己的 Rin 仓库。后面的 Secrets 和部署操作，都在这个 fork 仓库里完成。

## 三、准备 Cloudflare 的两个信息

### 1. 找到账户 ID

官方指南的做法是：在 Cloudflare 中打开一个已经绑定的域名，在右侧信息区域向下查找 **账户 ID**。如果你还没有绑定域名，也可以打开 Cloudflare Dashboard → **Workers & Pages**，在 **Account Details** 中复制 **Account ID**（账户 ID）。不要把 **Zone ID** 或 GitHub 用户 ID 当成账户 ID。

### 2. 创建 API 令牌

在 Cloudflare 控制台中按下面的路径操作：

**右上角头像 → 我的个人资料 → API 令牌 → 创建令牌**

创建令牌时，模板选择 **编辑 Cloudflare Workers**。继续确认权限针对你要部署的 Cloudflare 账户，并检查是否包含 Rin 部署 D1 数据库所需的写入权限；如果模板未提供 D1 权限，请在令牌设置中补齐相应权限。不要使用 Global API Key。创建完成后立即保存令牌，因为后面要把它作为 GitHub Secret 使用。

![Cloudflare API 令牌入口示意图](https://docs.openrin.org/cloudflare-api-key-cn.png)

*图片来源：Rin 官方部署指南。Cloudflare 控制台的文字或位置可能随界面更新而变化。*

> API 令牌只需要填入 GitHub Secrets，不要写进文章、公开评论或代码仓库文件中。

## 四、在 GitHub 中填写 Secrets

打开你刚才 fork 的 Rin 仓库，依次进入：

**Settings → Secrets and variables → Actions → New repository secret**

每次添加一个 Secret，填入名称和值，然后保存。按照官方文档的基础部署流程，加上推荐的账号密码登录，需要配置下面 5 项：

| Secret 名称 | 填什么 | 用途 |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | 刚刚创建的 Cloudflare API 令牌 | 让 GitHub Actions 可以部署到 Cloudflare |
| `CLOUDFLARE_ACCOUNT_ID` | Cloudflare 的账户 ID | 指定部署到哪个 Cloudflare 账户 |
| `JWT_SECRET` | 你自己生成的一段随机字符串 | 用于 JWT 签名 |
| `ADMIN_USERNAME` | 你想使用的登录用户名 | Rin 管理员登录用户名 |
| `ADMIN_PASSWORD` | 你想使用的登录密码 | Rin 管理员登录密码 |

`JWT_SECRET` 请自行生成一段难以猜中的随机字符串，不要照抄文档中的示例值；`ADMIN_PASSWORD` 也请设置为你自己的密码。

### 逐项检查

保存前可以按下面的清单检查：

- 名称拼写与上表完全一致，尤其注意下划线；
- API 令牌和账户 ID 没有前后多余空格；
- `JWT_SECRET`、`ADMIN_PASSWORD` 不是公开示例值；
- 5 项都已经出现在 **Actions** 的 Secrets 列表中。

GitHub Secrets 保存后不会显示完整内容，这是正常现象。

## 五、运行部署工作流

1. 在你的 Rin fork 仓库中打开 **Actions**。
2. 在左侧选择 **Build** 工作流。

如果页面提示启用此工作流，先按页面提示启用 Build。

3. 点击 **Run workflow**。
4. 保持默认分支为 `main`。
5. 再点击 **Run workflow** 开始运行。

部署开始后，可以点进这次运行记录查看进度。官方指南估计部署过程约需 3–5 分钟；当前仓库的 GitHub Actions 会在 Build 成功后自动触发 **Deploy** 工作流，所以要等 Build 和 Deploy 都结束，并确认 Deploy 成功，才算部署完成。实际时间可能更长；若还在运行，请先等待，不要重复点击。

> 首次部署不需要另外填写前端环境变量；当前客户端会在运行时从服务端获取所需配置。

## 六、部署完成后访问博客

部署完成后，请打开仓库 **Actions → Deploy → 最近一次成功的运行记录**，在 **Deploy to Cloudflare** 的日志中找到 `Resolved deployment URL` 后面的实际 `https://` 地址，并用它访问博客。也可使用部署完成后 Wrangler 输出的真实地址。不要直接照抄文档里的 `rin-client.pages.dev` 或 `your-account-id` 示例地址。

> **为什么以运行结果为准？** 文档示例展示的是 Pages 地址；截至 2026-11-27，当前仓库的部署工作流会在 Build 成功后继续部署到 Cloudflare，并从部署输出读取实际应用地址。你的项目名和实际结果可能不同。

## 七、第一次登录并发布文章

### 登录

1. 打开部署完成的前端页面。
2. 点击右上角的登录图标。
3. 输入你在 Secrets 中设置的 `ADMIN_USERNAME` 和 `ADMIN_PASSWORD`。
4. 登录成功后，点击头像进入个人资料页面；如果需要，可以在那里修改昵称和上传头像。

### 发布第一篇文章

1. 登录后点击右上角 **写作**。
2. 填写文章标题和正文内容。
3. 点击 **发布**。

发布后回到文章列表，打开刚发布的文章，确认正文可以正常读取。到这里，Rin 博客的首次部署和第一篇文章发布才算完成。

## 八、如果没有成功，先检查这三处

### 1. 部署失败

回到 GitHub Actions，打开失败的运行记录查看日志；优先检查 5 个 Secrets 的名称和值是否正确。

### 2. 无法登录

确认 `ADMIN_USERNAME` 和 `ADMIN_PASSWORD` 与 GitHub Secrets 中保存的内容一致，然后再查看 Worker 日志中的认证错误。

### 3. 数据库迁移失败

Rin 的部署脚本会自动运行数据库迁移。如果迁移失败，先查看 Deploy 日志和数据库迁移状态；官方指南建议进入 **Cloudflare Dashboard → D1**，找到对应数据库后打开 **Console**。当前版本迁移文件位于项目 `server/sql/` 目录（官方部署指南此处仍写旧目录）；不要不看日志和状态就把所有 SQL 文件重复执行。

## 九、先不要配置的项目

为了先把博客跑起来，第一次部署可以暂时不配置以下内容：

- 自定义域名；
- Cloudflare R2 或其他 S3 兼容图片存储；
- GitHub OAuth；
- Webhook 通知；
- 网站名称、描述等可选 GitHub Variables。

这些都属于官方指南的高级配置，不是本文首次部署所需的步骤。等博客可以正常登录和发布文章后，再根据需要回到官方文档配置即可。

## 来源与时效

- [Rin 部署指南](https://docs.openrin.org/guide/deploy.html)：账号准备、Fork、Cloudflare 账户 ID 与 API 令牌、GitHub Secrets、工作流、首次登录和发文流程的主要来源；核对日：2026-11-27（Asia/Shanghai）。
- [Rin 仓库 Build 工作流](https://github.com/openRin/Rin/blob/6a7ce76a220f4ebd8920325366c00e6d7f7d3569/.github/workflows/build.yml)、[Deploy 工作流](https://github.com/openRin/Rin/blob/6a7ce76a220f4ebd8920325366c00e6d7f7d3569/.github/workflows/deploy.yml) 与 [Cloudflare 部署脚本](https://github.com/openRin/Rin/blob/6a7ce76a220f4ebd8920325366c00e6d7f7d3569/cli/src/tasks/deploy-cf.ts)：核对默认部署方式、完成判断和实际 URL；核对日：2026-11-27。
- [Cloudflare 账户 ID 查找说明](https://developers.cloudflare.com/fundamentals/account/find-account-and-zone-ids/) 与 [API Token 权限模板](https://developers.cloudflare.com/fundamentals/api/reference/template/)：账户 ID 的备用查找入口和模板可自定义权限的说明；核对日：2026-11-27。
- [Cloudflare Wrangler CI/CD 文档](https://developers.cloudflare.com/workers/wrangler/ci-cd/)：Rin 官方指南提供的补充链接；控制台界面可能更新，核对日：2026-11-27。
