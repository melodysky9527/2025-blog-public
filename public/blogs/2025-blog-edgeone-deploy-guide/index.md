想用 YYsuni 的 2025 Blog 搭建个人博客，又想把网站托管到腾讯云 EdgeOne，可以沿用“GitHub 保存内容、GitHub App 管理写入”的方案，把部署平台换成 EdgeOne。

本文参考[原作者的使用引导](https://www.yysuni.com/blog/readme)，按当前项目代码和 EdgeOne 官方文档重新整理，依次完成 Fork、创建 GitHub App、导入仓库、填写构建参数、绑定域名和在线写作。

> 名称说明：EdgeOne Pages 已升级为 **EdgeOne Makers**。控制台和旧教程中可能出现这两个名称，本文使用“Makers（原 Pages）”。[官方产品说明](https://pages.edgeone.ai/document/product-introduction)

## 1. 先准备好这些东西

你需要一个 GitHub 账号和一个可登录的腾讯云账号。建议另备一个自己管理的域名，方便博客长期访问；没有域名时，可以先用控制台提供的预览入口检查部署。

这套流程用到两份不同的 GitHub 授权：

- **EdgeOne 的仓库授权**：让托管平台拉取代码、构建网站。
- **你自己创建的 GitHub App**：让博客页面把文章、图片和配置写回你的仓库。

两者都需要完成。只把仓库导入 EdgeOne，并不等于已经配置好博客的在线写作权限。[Git 导入说明](https://pages.edgeone.ai/document/importing-a-git-repository) · [项目内容提交代码](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/src/lib/github-commit.ts)

## 2. Fork 博客项目

打开 [YYsuni/2025-blog-public](https://github.com/YYsuni/2025-blog-public)，点击 **Fork**，将项目复制到自己的 GitHub 账号下。新手可以保留公开仓库和原仓库名，减少需要修改的地方。

记下自己仓库的三个信息：仓库所有者、仓库名、准备部署的分支。本文后续以 `OWNER` / `REPO` / `BRANCH` 表示它们；如果保留原名称和分支，后两项通常是 `2025-blog-public`、`main`，仍需在自己的仓库中确认。

这一步可以直接在浏览器完成，暂时不需要克隆代码到本地。[原项目说明](https://github.com/YYsuni/2025-blog-public)

## 3. 创建用于在线写作的 GitHub App

先把写作权限配置好，后面创建 EdgeOne 项目时就能一次填齐变量。

### 注册 App

在 GitHub 个人设置中进入 **Settings → Developer settings → GitHub Apps → New GitHub App**，按下面填写：

| 项目 | 填写方式 |
| --- | --- |
| GitHub App name | 起一个未被占用的名称 |
| Homepage URL | 填自己 Fork 后的仓库完整地址，已有博客域名也可以使用 |
| Webhook → Active | 取消勾选，这个博客的内容写入方案不需要它接收 Webhook |
| Repository permissions → Contents | 设为 **Read and write** |
| 其他权限 | 不额外扩大权限；系统自动要求的只读权限保留 |
| 安装范围 | 自己使用时选择 **Only on this account** |

这里关闭的是你创建的**写作 App** 的 Webhook，EdgeOne 的 Git 集成仍需正常保留。字段含义及最小权限原则可参考 [GitHub App 注册文档](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)。博客所需的 Contents 写权限沿用原项目的配置。

### 生成密钥并安装到仓库

创建后记录 **App ID**。后面填的是这个值，注意区分 Client ID 和 Installation ID。

接着找到密钥管理区域，生成并下载 PEM 私钥。不同界面可能显示 **Private keys → Generate a private key**，或 **Credentials → Key pairs → New key**。然后进入 **Install App**，选择 **Only select repositories**，只勾选自己的博客仓库，再完成安装。

**只创建 App 还不够，必须安装到目标仓库。** PEM 文件请妥善保存在自己手中；后面通过自己博客的“导入密钥”入口使用，不要提交到 GitHub，也不要填进任何 `NEXT_PUBLIC_` 变量。[GitHub 私钥说明](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps) · [原项目引导](https://www.yysuni.com/blog/readme)

## 4. 在 EdgeOne 导入自己的仓库

可以从你给出的 [EdgeOne 控制台入口](https://console.tencentcloud.com/edgeone/zones) 登录。**如果当前看到的是站点列表，请进入 Makers（旧版名 Pages）产品的 Git 仓库导入流程。** 官方当前文档指向的项目控制台是 [EdgeOne Makers](https://console.tencentcloud.com/edgeone/makers)。[官方 Git 导入说明](https://pages.edgeone.ai/document/importing-a-git-repository)

选择 **GitHub / 导入 Git 仓库**，按页面提示授权 EdgeOne 访问仓库，再选择自己 Fork 的那一份。仓库列表里找不到项目时，先检查 EdgeOne 的 GitHub 授权范围是否包含它。

### 核对构建参数

本次核对的项目使用 Next.js 16，`build` 脚本为 `next build`，并带有 `pnpm-lock.yaml`；配置里没有启用 `output: 'export'`。因此，这里走 Next.js 框架部署流程。[package.json](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/package.json) · [next.config.ts](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/next.config.ts)

| 配置项 | 本文采用的设置 |
| --- | --- |
| 框架预设 | **Next.js** |
| 根目录 | 仓库根目录，通常为 `./` |
| 生产分支 | 自己准备用来发布的分支，例如 `main` |
| 安装命令 | `pnpm install`；自动识别正确时可保留平台配置 |
| 构建命令 | `pnpm build` |
| 输出目录 | `.next` |
| Node.js | 建议选择平台预装的 Node.js 22，例如控制台可选的 `22.21.1` |

官方目前支持 Next.js 16，普通 Next.js 构建的输出目录是 `.next`；只有明确改成静态导出时才使用 `out`。这篇教程不需要修改成静态导出，也不使用仓库里的 Cloudflare 专用 `build:cf` 或 `deploy` 脚本。[Next.js 框架指南](https://pages.edgeone.ai/document/framework-nextjs) · [构建指南](https://pages.edgeone.ai/document/build-guide)

### 填写环境变量

在创建页面或项目设置的环境变量区域，添加以下四项。下面是格式示例，提交前要把所有值核对成自己的信息：

```dotenv
NEXT_PUBLIC_GITHUB_OWNER=your-github-name
NEXT_PUBLIC_GITHUB_REPO=2025-blog-public
NEXT_PUBLIC_GITHUB_BRANCH=main
NEXT_PUBLIC_GITHUB_APP_ID=123456
```

- `OWNER` 是你自己的仓库所有者，不要保留原作者的账号。
- `REPO` 是实际仓库名，改过名字就填新名字。
- `BRANCH` 要与计划发布的分支一致。
- `APP_ID` 换成刚才记录的 App ID。

变量至少应用到**生产环境**；需要预览分支也能使用同样配置时，再将它们应用到预览环境。这四个变量名来自项目的 [src/consts.ts](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/src/consts.ts)。

`NEXT_PUBLIC_` 变量会进入浏览器代码，不适合保存秘密。它们在构建时确定：**以后修改变量，需要重新构建部署，刷新网页不会更新旧构建里的值。** [Next.js 环境变量说明](https://nextjs.org/docs/app/guides/environment-variables) · [EdgeOne 环境管理](https://pages.edgeone.ai/document/build-guide)

### 选择加速区域

按域名情况选择：尚未完成 ICP 备案的域名，可先采用 **Global availability zone (exclude Chinese mainland)**，即全球可用区（不含中国大陆）。如果需要使用包含中国大陆节点的加速区域，先确认域名满足控制台要求；官方文档要求这类区域的自定义域名完成 ICP 备案。具体可选区域以账号当前界面为准。[域名与加速区域说明](https://pages.edgeone.ai/document/domain-overview)

## 5. 处理原项目的重定向，再开始部署

当前项目在 `next.config.ts` 中把 `/zh` 和 `/en` 跳转到首页。EdgeOne 官方说明目前不支持直接沿用 Next.js 内置的 `redirects` / `rewrites` 配置，建议使用 `edgeone.json`。

如果希望这两个入口在 EdgeOne 上继续跳转，在仓库根目录新建 `edgeone.json`，加入下面的配置；已有这个文件时，只合并 `redirects`，保留其他设置：

```json
{
  "redirects": [
    { "source": "/zh", "destination": "/", "statusCode": 301 },
    { "source": "/en", "destination": "/", "statusCode": 301 }
  ]
}
```

不需要改动 `next.config.ts` 的其他内容。配置依据：[Next.js 支持范围](https://pages.edgeone.ai/document/framework-nextjs) · [edgeone.json 配置说明](https://pages.edgeone.ai/document/edgeone-json)。

核对仓库、分支、变量和构建设置后，点击 **开始部署 / Start Deployment**。如果配置页打开期间又提交了文件，确认这次构建使用的是最新提交；必要时从控制台重新部署。

等待构建与部署状态成功，再通过控制台提供的项目地址或预览入口检查首页和文章列表。构建失败时打开日志，先看最早出现的有效错误；部署时间取决于项目构建和平台状态，不用按照原 Vercel 教程里的耗时等待。[部署与重新部署说明](https://pages.edgeone.ai/document/create-deploys)

## 6. 绑定自己的域名

进入项目的**域名管理**，添加自己控制的域名，例如 `blog.example.com`。按控制台指引完成归属验证，再到域名 DNS 服务商处添加指定的 CNAME，随后完成 HTTPS 证书配置。记录值以当前项目显示为准，别照抄其他人的 CNAME。[自定义域名指南](https://pages.edgeone.ai/document/custom-domain)

把正式域名关联到生产环境，等待解析、验证和证书就绪，再检查 HTTPS 访问。

这里有个容易误判的问题：官方当前规则下，**从中国大陆网络访问平台分配的项目域名、部署域名，需要使用系统生成的预览链接，有效期为 3 小时；过期会返回 401**。可以回到控制台项目概览重新点“预览”。因此，预览链接适合调试，长期访问建议绑定自己的域名。[平台域名访问规则](https://pages.edgeone.ai/document/domain-overview)

## 7. 导入密钥，发布第一篇文章

打开部署好的自己的博客，从“写文章”进入编辑器，也可以访问自己域名下的 `/write`。点击**导入密钥**，选择前面下载的 PEM 文件。

建议先写一篇简短测试文章：填写标题、唯一 slug、摘要和正文；需要图片时先在图片管理区上传，再插入正文，最后打开预览检查。确认内容后再发布。这是原项目提供的在线写作方式。[原作者使用引导](https://www.yysuni.com/blog/readme)

点击“发布”后的过程，可以这样理解：

1. 博客通过你的 GitHub App，把内容提交到配置好的仓库分支。
2. 在 Git 集成正常、对应环境开启自动部署的情况下，仓库变更触发 EdgeOne 构建。
3. 新部署成功后，对应环境的网站才会提供更新后的内容。

第一步由[项目提交代码](https://github.com/YYsuni/2025-blog-public/blob/e72a6b8091e2f38848f3472cb9b9c215828168bd/src/lib/github-commit.ts)支持，后两步依据 [Git 导入与部署规则](https://pages.edgeone.ai/document/importing-a-git-repository)及[环境管理](https://pages.edgeone.ai/document/build-guide)整理。这说明“发布成功”的提示还需要配合 EdgeOne 的部署状态一起检查。

如果 GitHub 已经出现新提交，但 EdgeOne 没有开始部署，先检查分支、自动部署开关和仓库授权，也可以手动触发一次重新部署。确认第一次提交的结果后再操作，避免重复发布。

测试通过后，再通过站内管理入口清理不需要的原作者示例文章，修改站点名称、介绍等配置。部分首页内容仍需调整仓库中的 `src/app/(home)` 相关组件；按自己 Fork 版本的目录核对，修改后提交并等待重新部署。[原作者配置说明](https://www.yysuni.com/blog/readme)

## 8. 常见问题按这个顺序查

| 现象 | 优先检查 |
| --- | --- |
| 导入仓库时找不到项目 | EdgeOne 的 GitHub 授权范围是否包含自己的 Fork |
| 依赖安装失败 | 日志中的 Node、pnpm 版本及锁文件兼容性；不要先删锁文件或盲目升级依赖 |
| 提示找不到输出目录 | 是否误填 `out`；本文的普通 Next.js 构建输出应为 `.next` |
| 导入 PEM 后写入失败，出现 401/403 | App ID、密钥是否配套，App 是否安装到目标仓库，Contents 是否可写，分支规则是否允许此次写入 |
| 页面仍显示错误的仓库配置 | 四个变量是否正确、是否应用到当前环境、修改后是否重新构建 |
| GitHub 有文章，网站却没变化 | EdgeOne 是否构建了对应提交，生产分支和域名环境是否一致，部署是否成功；之后再检查缓存 |
| 平台预览地址返回 401 | 若从中国大陆网络访问，先重新生成预览链接，再按日志排查其他原因 |
| 首页能开，某篇文章打不开 | 直接访问并刷新该文章地址，核对对应部署是否包含正文文件；用访客窗口复查 |

上表是按配置依赖关系整理的排查建议，不代表每个状态码只有一种原因。尤其是写作 App、EdgeOne 仓库集成和网站访客访问，分别发生在不同环节，排查时先确认出错的位置。

## 来源与时效

本文为资料核验版部署教程，核对日为 **2026 年 11 月 27 日（Asia/Shanghai）**。项目代码核对到提交 `e72a6b8`；未在读者的腾讯云账号中创建项目或完成线上部署，不作“亲测成功”承诺。

- [YYsuni：2025 Blog 的使用引导](https://www.yysuni.com/blog/readme)：原项目流程与在线写作说明，本文据此重新组织 EdgeOne 版本，未沿用 Vercel 截图。
- [2025-blog-public 项目及本次代码快照](https://github.com/YYsuni/2025-blog-public/tree/e72a6b8091e2f38848f3472cb9b9c215828168bd)：框架、脚本、配置变量和提交逻辑。
- [EdgeOne 产品介绍](https://pages.edgeone.ai/document/product-introduction)、[Git 仓库导入](https://pages.edgeone.ai/document/importing-a-git-repository)：产品名称与导入流程。
- [Next.js 框架指南](https://pages.edgeone.ai/document/framework-nextjs)、[构建指南](https://pages.edgeone.ai/document/build-guide)、[edgeone.json](https://pages.edgeone.ai/document/edgeone-json)：构建、环境变量与重定向配置。
- [部署触发](https://pages.edgeone.ai/document/create-deploys)、[域名概览](https://pages.edgeone.ai/document/domain-overview)、[自定义域名](https://pages.edgeone.ai/document/custom-domain)：部署、预览域名和域名接入条件。
- [GitHub App 注册](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)、[私钥管理](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps)、[Next.js 环境变量](https://nextjs.org/docs/app/guides/environment-variables)：权限与变量使用说明。

以上资料均于核对日访问。控制台菜单、版本和域名规则可能调整；账号内可用配置、资源用量与计费，以实际使用时的官方控制台为准。
