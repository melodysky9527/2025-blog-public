第一次搭博客，可以按“看图 → 操作 → 核对结果”的顺序来。这篇保留 YYsuni 的 2025 Blog 项目，把部署平台改为腾讯云 EdgeOne，并补上 **16 张配图**：从 Fork 仓库到写下第一篇文章，不要求你先在电脑上安装开发环境。

先记住整条路线：**复制仓库 → 创建写作 App → 导入 EdgeOne → 填参数 → 部署 → 绑定域名 → 在线写文章。**

> **配图怎么用**：GitHub 账号设置采用原作者教程截图，EdgeOne 采用官方文档截图，写作编辑器为本站实际演示截图。图片上的编号和外侧说明是本次补充；账号、仓库、App ID、域名和 DNS 值都要换成自己的。参数对照图已单独标为“非控制台截图”。界面名称可能随版本、语言不同而变化，以当前页面和文中说明为准。
>
> **产品名称**：EdgeOne Pages 已升级为 **EdgeOne Makers**。旧教程还可能写 Pages；本文走 Makers 的 Next.js 框架部署流程。[官方产品说明](https://pages.edgeone.ai/document/product-introduction)

## 第 1 步：把博客项目复制到自己的 GitHub

准备一个 GitHub 账号、一个可登录的腾讯云账号。域名可以稍后再配；长期使用时建议准备一个自己管理的域名。

打开 [YYsuni/2025-blog-public](https://github.com/YYsuni/2025-blog-public)，找到 **Fork**。它的意思是“把这个项目复制一份到我的账号下”。

![图 01：把项目复制到自己的 GitHub](/blogs/2025-blog-edgeone-deploy-guide/da25188e5d7a833f.webp)

*图 01｜原作者截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://www.yysuni.com/blog/readme)。*

选择自己的账号，创建副本。新手可以保留仓库名 `2025-blog-public`，避免多改一处。

**完成标志**：浏览器里的仓库地址变成 `github.com/你的账号/你的仓库`。记下所有者、仓库名和要发布的分支；后文用 `OWNER`、`REPO`、`BRANCH` 表示。分支若保留原设置通常是 `main`，仍以自己的仓库为准。[原项目引导](https://www.yysuni.com/blog/readme)

## 第 2 步：创建一个能写入仓库的 GitHub App

先分清两份授权：**自己的 GitHub App 负责把文章写回仓库；EdgeOne 的仓库授权负责拉代码和部署。** 两者都要做。下面先配写作 App。[项目提交逻辑](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/src/lib/github-commit.ts) · [EdgeOne 导入说明](https://pages.edgeone.ai/document/importing-a-git-repository)

### 2.1 找到创建 App 的入口

在 GitHub 个人设置中，依次进入 **Settings → Developer settings → GitHub Apps → New GitHub App**。

![图 02：进入 GitHub 的开发者设置](/blogs/2025-blog-edgeone-deploy-guide/5dbe2e3c5135ad9f.webp)

*图 02｜原作者截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://www.yysuni.com/blog/readme)。*

按下面的图找到表单，再逐项填：

![图 03：填写写作 App 的基本信息](/blogs/2025-blog-edgeone-deploy-guide/3de4045fc5d47a77.webp)

*图 03｜原作者截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://www.yysuni.com/blog/readme)。*

- **GitHub App name**：起一个未被占用的名称。
- **Homepage URL**：填自己 Fork 后的仓库完整地址；已经有博客地址时也可以填博客地址。
- **Webhook → Active**：取消勾选。这里只关闭你的写作 App 接收 Webhook 的功能，不要因此关闭 EdgeOne 的 Git 集成。
- **安装范围**：自己使用时选 **Only on this account**。

这些字段和最小权限原则可对照 [GitHub 官方注册文档](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)。

### 2.2 给 Contents 写权限

在 **Repository permissions** 中找到 **Contents**，设为 **Read and write**。其他权限不额外扩大，系统要求保留的只读权限照常保留。

![图 04：只配置写文章需要的仓库权限](/blogs/2025-blog-edgeone-deploy-guide/fb60866aa7a77b90.webp)

*图 04｜原作者截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://www.yysuni.com/blog/readme)。*

然后创建 App。这一权限配置沿用[原作者的项目引导](https://www.yysuni.com/blog/readme)，用于项目创建文件、提交和更新分支。

### 2.3 保存 App ID 和 PEM，再安装到仓库

App 创建后，请完成三件事：

1. 记录页面里的 **App ID**，不要拿 Client ID 或 Installation ID 代替。
2. 生成并下载 PEM 私钥。旧界面可能是 **Private keys → Generate a private key**；新界面可能是 **Credentials → Key pairs → New key**。
3. 进入 **Install App**，继续安装到自己的博客仓库。

![图 05：创建 App 后，还要安装到仓库](/blogs/2025-blog-edgeone-deploy-guide/a246892e599c61c6.webp)

*图 05｜原作者截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://www.yysuni.com/blog/readme)。*

仓库范围选 **Only select repositories**，只勾选自己的博客仓库，再完成安装。

![图 06：只授权自己的博客仓库](/blogs/2025-blog-edgeone-deploy-guide/06e132690185ea43.webp)

*图 06｜原作者截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://www.yysuni.com/blog/readme)。*

**完成标志**：你手上有自己的 App ID 和一个下载好的 `.pem` 文件，而且 App 已安装到目标仓库。只创建 App、不安装，后面仍然不能写入。

**PEM 是私钥文件**，请保存在自己手中，后面只通过自己博客的“导入密钥”入口使用。不要提交到仓库、贴进文章，也不要填到任何 `NEXT_PUBLIC_` 变量里。[GitHub 私钥说明](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps)

## 第 3 步：让 EdgeOne 导入自己的仓库

从 [EdgeOne 控制台](https://console.tencentcloud.com/edgeone/zones) 登录后，如果看到的是站点列表，请进入 **Makers（原 Pages）** 的 Git 仓库导入流程。官方当前项目入口指向 [EdgeOne Makers](https://console.tencentcloud.com/edgeone/makers)。[官方导入指南](https://pages.edgeone.ai/document/importing-a-git-repository)

选择 **GitHub / 导入 Git 仓库**：

![图 07：进入 Makers，选择从 GitHub 导入](/blogs/2025-blog-edgeone-deploy-guide/c0e54496123cd348.webp)

*图 07｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/importing-a-git-repository)。*

跟着 GitHub 授权页面，让 EdgeOne 能访问自己的 Fork，再回到 EdgeOne 选择这份仓库。

![图 08：让 EdgeOne 能读取你的仓库](/blogs/2025-blog-edgeone-deploy-guide/f41f1f5ad7570ab4.webp)

*图 08｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/importing-a-git-repository)。*

**完成标志**：EdgeOne 里选中的仓库所有者是你自己，仓库名也与第 1 步一致。找不到仓库时，先检查 EdgeOne 的 GitHub 授权范围是否包含它。

## 第 4 步：按本项目填写构建参数

在创建项目时核对构建设置；创建后也可以到项目设置的构建与部署区域调整。下图用于辨认设置位置，**官方演示值不一定属于这个博客项目**。

![图 09：找到构建设置，数值按本文填写](/blogs/2025-blog-edgeone-deploy-guide/a4393788d111e947.webp)

*图 09｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/build-guide)。*

真正要照着核对的是这一张：

![图 10：本项目这 7 项，照着核对](/blogs/2025-blog-edgeone-deploy-guide/cb2518edf65f3aee.webp)

*图 10｜填写对照图（非控制台截图）。本图是填写对照，不是控制台界面。 [参数依据](https://pages.edgeone.ai/document/framework-nextjs)。*

为方便复制和核对，文字版也放在这里：

| 项目 | 本文采用的值 |
| --- | --- |
| 框架预设 | Next.js |
| 根目录 | `./`，也就是仓库根目录 |
| 生产分支 | 自己准备发布的分支，例如 `main` |
| 安装命令 | `pnpm install`；自动识别正确时可保留平台配置 |
| 构建命令 | `pnpm build` |
| 输出目录 | `.next` |
| Node.js | 平台可选的 Node.js 22，例如 `22.21.1` |

本次核对的项目是 Next.js 16，`build` 脚本就是 `next build`，仓库带 `pnpm-lock.yaml`，且未启用 `output: 'export'`。官方支持该版本的普通框架构建；它的输出是 **`.next`**，不是静态导出使用的 `out`，也不需要运行 Cloudflare 专用脚本。

依据：[项目 package.json](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/package.json) · [Next.js 框架指南](https://pages.edgeone.ai/document/framework-nextjs) · [构建指南](https://pages.edgeone.ai/document/build-guide)。

## 第 5 步：添加四个环境变量，并选择区域

环境变量可以理解为“告诉这个网站要管理哪一个仓库”。在创建页或项目设置中找到环境变量区域：

![图 11：添加四个 GitHub 环境变量](/blogs/2025-blog-edgeone-deploy-guide/08bb3c5fb05ce93c.webp)

*图 11｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/build-guide)。*

**变量名按下面复制，等号右边换成自己的信息**：

```dotenv
NEXT_PUBLIC_GITHUB_OWNER=your-github-name
NEXT_PUBLIC_GITHUB_REPO=2025-blog-public
NEXT_PUBLIC_GITHUB_BRANCH=main
NEXT_PUBLIC_GITHUB_APP_ID=123456
```

- `OWNER`：自己的仓库所有者，不是原作者的账号。
- `REPO`：自己的实际仓库名，改过名就填新名字。
- `BRANCH`：要写入并部署的分支。
- `APP_ID`：第 2 步记录的 App ID，示例里的 `123456` 不能照抄。

至少应用到**生产环境**；需要预览分支也能使用同样配置时，再应用到预览环境。变量名来自项目 [src/consts.ts](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/src/consts.ts)。

`NEXT_PUBLIC_` 的值会进入浏览器代码，不适合保存秘密；它在构建时确定，**以后修改变量要重新构建，刷新网页不会修改旧构建里的值**。[Next.js 环境变量说明](https://nextjs.org/docs/app/guides/environment-variables) · [EdgeOne 构建指南](https://pages.edgeone.ai/document/build-guide)

接着按自己的域名情况选择加速区域：

![图 12：按自己的域名条件选择加速区域](/blogs/2025-blog-edgeone-deploy-guide/6c8730b24badc07d.webp)

*图 12｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/importing-a-git-repository)。*

尚未备案的域名，可先采用 **Global availability zone (exclude Chinese mainland)**，即全球可用区（不含中国大陆）。包含中国大陆节点的区域，其自定义域名需满足官方的 ICP 备案要求；可选项以账号当前控制台为准。[域名与加速区域说明](https://pages.edgeone.ai/document/domain-overview)

**完成标志**：仓库、分支、App ID 都是你自己的信息，变量已应用到将要部署的环境，区域也已核对。

## 第 6 步：补上两个跳转规则，再开始部署

原项目把 `/zh`、`/en` 跳转到首页。EdgeOne 官方当前说明不支持直接沿用 Next.js 内置的 `redirects` / `rewrites`，建议用 `edgeone.json`。[支持范围](https://pages.edgeone.ai/document/framework-nextjs) · [配置说明](https://pages.edgeone.ai/document/edgeone-json)

如果希望保留这两个入口，在仓库根目录新建 `edgeone.json`，内容如下。已有这个文件时只合并 `redirects`，保留其他配置：

```json
{
  "redirects": [
    { "source": "/zh", "destination": "/", "statusCode": 301 },
    { "source": "/en", "destination": "/", "statusCode": 301 }
  ]
}
```

完成文件提交后，核对 EdgeOne 将构建的提交版本，再点击 **开始部署 / Start Deployment**。如果创建页面打开期间又修改了仓库，必要时重新部署最新提交。

![图 13：等到部署成功，再打开网站检查](/blogs/2025-blog-edgeone-deploy-guide/941c6622d7bff210.webp)

*图 13｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/importing-a-git-repository)。*

**完成标志**：本次部署显示成功，并且能从控制台预览入口打开首页和文章列表。构建失败时先查看日志里最早的有效错误；不要因为原 Vercel 教程写了某个耗时，就认定超过它一定失败。[部署与重新部署说明](https://pages.edgeone.ai/document/create-deploys)

## 第 7 步：绑定自己的域名

进入项目的**域名管理**，添加自己控制的域名，例如 `blog.example.com`。

![图 14：在项目的域名管理中添加域名](/blogs/2025-blog-edgeone-deploy-guide/f7875511fef30f46.webp)

*图 14｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/custom-domain)。*

然后按控制台提供的信息，到域名的 DNS 服务商处设置记录。需要先做归属验证时，按页面给出的记录类型和记录值填写；业务 CNAME 也使用当前项目给出的值。

![图 15：到 DNS 服务商处填写项目给出的记录](/blogs/2025-blog-edgeone-deploy-guide/13b4cf1c6524dd63.webp)

*图 15｜EdgeOne 官方截图。已增加外侧步骤说明，截图里的示例账号、仓库和记录值请勿照抄。 [图片出处](https://pages.edgeone.ai/document/custom-domain)。*

不要照抄截图里的 DNS 值。把正式域名关联到生产环境，完成 HTTPS 证书配置，等待解析、验证和证书就绪后再访问。[自定义域名指南](https://pages.edgeone.ai/document/custom-domain)

**先没有域名怎么办？** 可以先用控制台生成的预览入口调试。官方当前规则下，从**中国大陆网络**访问平台分配的项目域名、部署域名，需要使用系统生成的预览链接；有效期 **3 小时**，过期会返回 **401**，回项目概览重新点“预览”即可。这个限制不应泛化到所有自定义域名。[平台域名访问规则](https://pages.edgeone.ai/document/domain-overview)

## 第 8 步：照着编号，写第一篇文章

打开自己博客的“写文章”，或访问自己域名下的 `/write`。这张图是本次在本站实际编辑器中拍摄的演示，**没有提交示例文章**：

![图 16：写文章：按蓝色编号找到入口](/blogs/2025-blog-edgeone-deploy-guide/aedd1dbe626ae000.webp)

*图 16｜本站实际编辑器演示截图（未提交）。本次在本站编辑器实际拍摄，演示文章未提交。 [写作入口](https://blog.1381886.xyz/write)。*

| 图中编号 | 找到哪里 | 怎么操作 |
| --- | --- | --- |
| 1 | 标题 | 填文章名字，例如“我的第一篇文章” |
| 2 | Slug | 填唯一地址名，例如 `hello-edgeone`；建议用小写英文、数字和短横线 |
| 3 | 正文 | 写文章内容；不熟 Markdown 时先写简单段落 |
| 4 | 摘要 | 用一两句话说明文章讲什么；标签和分类也在右侧填写 |
| 5 | 图片管理的“＋” | 先添加图片，再拖入正文；建议先压缩图片 |
| 6 | 预览 | 检查标题、正文、图片和代码是否正常 |
| 7 | 导入密钥 | 选择第 2 步下载的 PEM；认证后，新文章按钮变成“发布” |

已有文章的编辑页面在认证后会显示“更新”。使用的是 **GitHub App 下载的 PEM 文件**，不是腾讯云 API 密钥。图片与写作流程可参考[原作者引导](https://www.yysuni.com/blog/readme)；按钮状态也与[本站采用的编辑器代码](https://github.com/melodysky9527/2025-blog-public/blob/main/src/app/write/components/actions.tsx)一致。

先写一篇短测试文章，预览无误后再发布。之后按这个顺序检查：

1. **仓库有新提交**：文章已写回你配置的仓库分支。
2. **EdgeOne 新部署成功**：Git 集成正常，且相应环境开启自动部署时，符合条件的分支提交会触发构建。
3. **访客页面显示新内容**：用无登录窗口打开文章地址，再检查列表和首页。

“发布成功”的提示和“网站已经更新”是两个检查点。仓库已有新提交但网站没变时，先检查部署分支、Auto Deploy 开关和构建记录，必要时手动重新部署；确认结果后再操作，避免重复提交。[Git 部署规则](https://pages.edgeone.ai/document/importing-a-git-repository) · [环境管理](https://pages.edgeone.ai/document/build-guide)

测试通过后，再清理不需要的原作者示例文章，修改站点名称、介绍等配置。部分首页内容仍需改 `src/app/(home)` 相关组件，按自己的 Fork 版本核对。[原作者配置说明](https://www.yysuni.com/blog/readme)

## 卡住时，先对照这一张表

| 现象 | 先检查什么 |
| --- | --- |
| EdgeOne 找不到仓库 | EdgeOne 的 GitHub 授权是否包含自己的 Fork |
| 提示找不到输出目录 | 本文普通 Next.js 构建应填 `.next`，是否误填 `out` |
| 导入 PEM 后写入失败 | App ID 和密钥是否配套，App 是否已安装，Contents 是否可写，分支规则是否允许写入 |
| 网站仍使用错误仓库信息 | 四项变量是否正确，是否应用到当前环境，修改后是否重新构建 |
| GitHub 已更新，网站没变化 | 是否部署了对应提交、构建是否成功、域名环境是否正确；之后再排查缓存 |
| 平台预览链接返回 401 | 从中国大陆网络访问时，先重新生成有效预览链接，再排查其他原因 |

这是排查建议，不代表每种现象只有一种原因。先确定出错的是“App 写仓库”“EdgeOne 构建”还是“访客访问”，再检查对应环节。

## 来源与时效

文字与配图资料核对日为 **2026 年 11 月 27 日（Asia/Shanghai）**。上游框架与配置核对到提交 `e72a6b8`；配图没有把示例账号、域名和旧界面当成读者实际设置。

- 图 01—06 来自 [YYsuni 原项目使用引导](https://www.yysuni.com/blog/readme)，增加了本次步骤说明；不使用其中的 Vercel 部署截图。
- 图 07—09、11—15 来自 EdgeOne 官方的 [Git 仓库导入](https://pages.edgeone.ai/document/importing-a-git-repository)、[构建指南](https://pages.edgeone.ai/document/build-guide)和[自定义域名](https://pages.edgeone.ai/document/custom-domain)页面，已在各图标注出处。
- 图 10 为按[项目配置](https://github.com/YYsuni/2025-blog-public/tree/e72a6b8091e2f38848f3472cb9b9c215828168bd)和 [Next.js 框架指南](https://pages.edgeone.ai/document/framework-nextjs)整理的参数对照图，明确不是控制台截图。
- 图 16 为本站实际编辑器演示截图，蓝色编号按真实控件位置添加，示例文章未提交。
- 其他依据：[GitHub App 注册](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)、[私钥管理](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps)、[Next.js 环境变量](https://nextjs.org/docs/app/guides/environment-variables)、[EdgeOne 域名规则](https://pages.edgeone.ai/document/domain-overview)。

本教程不是在读者账号内创建云项目的实测记录。界面、版本、区域和域名规则可能调整；可用配置、资源用量与计费以实际使用时的官方控制台为准。
