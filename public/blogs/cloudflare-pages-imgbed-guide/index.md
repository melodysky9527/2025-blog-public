> 适合第一次使用 Cloudflare Pages 的读者。本文按“GitHub 保存代码 → Pages 运行网站 → KV 保存目录和设置 → R2 保存图片文件”的路线一步步部署，并做一次实际上传验收。
>
> 核验日期：2026-11-27。Cloudflare 控制台会持续调整按钮名称和菜单位置；如果界面不同，请以字段名称和项目官方文档为准。

## 先看懂：这几个服务分别做什么？

CloudFlare ImgBed 不是“部署成功就
封面
+
元信息
从 Fork 项目开始，按 11 个步骤完成 Cloudflare Pages、KV 和 R2 配置，设置管理员账号并启用上传渠道，最后用小图片验证上传与访问。
#Cloudflare×
#图床×
#部署教程×
添加标签（按回车）

未分类

2026/11/28 16:48

图片管理
能上传”的单一网页。它需要把程序、数据库和文件存储连起来：

封面
+
元信息
从 Fork 项目开始，按 11 个步骤完成 Cloudflare Pages、KV 和 R2 配置，设置管理员账号并启用上传渠道，最后用小图片验证上传与访问。
#Cloudflare×
#图床×
#部署教程×
添加标签（按回车）

未分类

2026/11/28 16:48

图片管理

| 服务 | 在本教程中的职责 | 关键名称 |
|---|---|---|
| GitHub | 保存 Fork 后的项目代码；提交更新后触发 Pages 自动部署。 | 自己的 `CloudFlare-ImgBed` 仓库 |
| Cloudflare Pages | 运行 ImgBed 网站和后端，处理页面访问、登录、上传及文件读取。 | Pages 项目及其生产网址 |
| Workers KV | 保存文件索引、设置和会话等元数据。 | Pages 绑定变量 `img_url` |
| R2 | 保存实际图片和文件内容。 | Pages 绑定变量 `img_r2` |

**记住这个区别：KV 保存“文件信息和设置”；R2 保存“实际文件内容”。** 两者用途不同，不能把 R2 当成 KV，也不能只创建 KV 就指望它保存图片。

本教程选择 **KV + R2**：都在 Cloudflare 内配置，不用额外建 Telegram 机器人。KV 和 D1 是二选一的元数据数据库；如果你已有 Telegram、S3、WebDAV 等存储，也可以在管理后台配置相应上传渠道。

## 开始前准备

准备以下内容：

- 一个 GitHub 账号，用来 Fork 项目并授权 Cloudflare 读取代码。
- 一个 Cloudflare 账号，用来创建 Pages、KV 和 R2 资源。
- 一个 R2 存储桶。稍后会在步骤中创建。
- 一个可用浏览器。自定义域名是可选项，可以先用 Cloudflare 提供的 `pages.dev` 地址。

Cloudflare 的额度、R2 价格和账号要求会变化。部署前请查看账户当前的用量与计费说明；“有免费额度”不等于任何用法都永远免费。第一次部署建议只上传一张小图片测试。

## 步骤总览

1. Fork `CloudFlare-ImgBed` 项目到自己的 GitHub。
2. 在 Cloudflare Pages 连接自己的 GitHub 仓库。
3. 填写构建命令 `npm install`、输出目录 `/frontend-dist`，完成首次生产部署。
4. 创建 KV 命名空间，本教程命名为 `img_url`。
5. 将 KV 以变量名 `img_url` 绑定到 Pages 生产环境。
6. 重新部署，等待 KV 绑定生效。
7. 首次进入 ImgBed 管理后台，设置管理员凭据并验证登录。
8. 在同一 Cloudflare 账号创建 R2 存储桶。
9. 将 R2 以变量名 `img_r2` 绑定到 Pages 生产环境，并再次重新部署。
10. 在 ImgBed 管理后台添加、启用并保存 R2 上传渠道。
11. 上传一张小图片，验收返回链接、后台文件记录和 R2 对象。

---

## 第 1 步：把项目 Fork 到自己的 GitHub

Fork 就是把作者的公开仓库复制到你自己的 GitHub 账号下。Cloudflare 后面连接的是你自己的副本。

1. 登录 [GitHub](https://github.com/)。
2. 打开项目仓库：[MarSeventh/CloudFlare-ImgBed](https://github.com/MarSeventh/CloudFlare-ImgBed)。
3. 点击页面右上方的 **Fork**。
4. 在 Owner 处选择自己的 GitHub 账号；仓库名可保留 `CloudFlare-ImgBed`。
5. 点击 **Create fork**，等待 GitHub 创建完成。
6. 检查地址栏应当以 `github.com/你的用户名/CloudFlare-ImgBed` 开头。

如果你以前已经 Fork 过，先打开自己的仓库，点 **Sync fork → Update branch**，让代码跟上游版本同步后再部署。旧副本可能缺少新版文件夹或设置。项目版本更新有时会要求手动同步 Fork，部署前也可以查看仓库公告。

**检查点：** 仓库页面的所有者是你自己的 GitHub 用户名，而不是 `MarSeventh`。

## 第 2 步：从 Cloudflare 控制台创建 Pages 项目

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)。
2. 如果你的账号加入了多个 Cloudflare Account，先选一个账号。后面的 Pages、KV 和 R2 都要在这个账号下操作，方便互相绑定。
3. 打开 **Workers & Pages**。
4. 点击 **Create application / 创建应用程序**。
5. 选择 Pages 的 Git 部署入口：新版界面通常是 **Pages → Connect to Git**；有些账号会先显示 **Looking to deploy Pages? → Get started**，再进入 **Import an existing Git repository**。
6. 选择 **GitHub**。第一次连接时，按照提示登录 GitHub 并授权 Cloudflare。
7. 在仓库列表中选中第 1 步自己的 `CloudFlare-ImgBed`，点击 **Begin setup / 开始设置**。

![Cloudflare Pages 创建项目页面，参考项目官方部署文档](/blogs/cloudflare-pages-imgbed-guide/25a98e51b162299d.png)

> 图片来自 [CloudFlare ImgBed 官方 Pages 部署说明](https://cfbed.sanyue.de/deployment/pages.html)。控制台改版后按钮位置可能变化，找 “Pages” 和 “Connect to Git” 即可。

**找不到自己的仓库？** 回到 GitHub 确认 Fork 已完成；再检查 Cloudflare 的 GitHub App 是否被授权访问这个仓库。组织仓库还要确认组织允许该 App 访问。调整授权后重新载入仓库列表。

> **重要：** Cloudflare 官方说明，选择 Git 集成后，不能再把同一个 Pages 项目切换成 Direct Upload。这里我们明确使用 GitHub 自动部署路线。

## 第 3 步：填写构建参数并首次部署

在 **Set up builds and deployments / 设置构建和部署** 页面，按表填写。构建目录特别容易填错，建议复制下面的值。

| 字段 | 填写值 | 说明 |
|---|---|---|
| Project name / 项目名称 | 自选，例如 `my-imgbed` | 生成 `https://my-imgbed.pages.dev`。名称被占用就换一个。 |
| Production branch / 生产分支 | `main` | 如果你的 Fork 默认分支不是 `main`，就选仓库实际存在的默认分支。 |
| Framework preset / 框架预设 | `None` / 无 | 这是项目文档给出的设置；不要误选其他框架预设。 |
| Build command / 构建命令 | `npm install` | 拼写和空格都按这里填写。 |
| Build output directory / 构建输出目录 | `/frontend-dist` | 新版项目的前端文件目录。不要填成 `/`。 |
| Root directory / 根目录 | 保持默认（仓库根目录） | 除非你把整个项目放进了子目录，否则不要修改。 |

![CloudFlare ImgBed 官方文档中的 Pages 构建设置示意](/blogs/cloudflare-pages-imgbed-guide/c37447eb0a6f2e42.png)

填写完成后，再核对一次：

```text
Production branch: main
Build command: npm install
Build output directory: /frontend-dist
Root directory: 仓库根目录（默认）
```

然后点击 **Save and Deploy / 保存并部署**。Cloudflare 会显示构建日志；等待状态变为成功，再点击项目提供的 `*.pages.dev` 网址打开网站。

如果出现构建失败，不要只看红色状态就重复点部署。展开日志，先看第一条实际错误，再核对分支、命令、输出目录和 GitHub 仓库权限。

**检查点：** 首页能打开，Pages 的最新生产部署显示成功。此时数据库和文件上传渠道还没配置，上传不成功是正常的，继续下一步。

## 第 4 步：创建 KV 命名空间

KV 是本教程使用的元数据数据库，用来保存 ImgBed 的文件索引和设置。

1. 回到 Cloudflare Dashboard，打开 **Storage & databases / 存储和数据库 → Workers KV**。
2. 点击 **Create namespace / 创建命名空间**（有些界面会显示“创建实例”）。
3. 命名空间名称填：

   ```text
   img_url
   ```

4. 点击创建，等列表中出现 `img_url`。

![Workers KV 创建命名空间的示意图](/blogs/cloudflare-pages-imgbed-guide/2b6d27536fb967c3.png)

![填写 img_url 并创建 KV 命名空间](/blogs/cloudflare-pages-imgbed-guide/fe99f9f2368a10c6.png)

命名空间的资源名称可以自己起；为了跟教程一致，这里使用 `img_url`。下一步的 **变量名称** 必须按程序要求填写 `img_url`，这两处看起来相似，但不是同一个字段。

## 第 5 步：把 KV 绑定到 Pages

绑定的意思是：把刚才创建的 KV 资源交给 Pages 项目使用。

1. 打开 **Workers & Pages → 你的 Pages 项目**。
2. 进入 **Settings / 设置 → Bindings / 绑定**。
3. 选择 **Add / 添加 → KV namespace / KV 命名空间**。
4. 在 **Variable name / 变量名称** 中填写：

   ```text
   img_url
   ```

5. 在 KV namespace 下拉框中，选择第 4 步创建的命名空间。
6. 确认添加到 **Production / 生产环境**，然后保存。

### 这里最容易填错

- 变量名要完全等于 `img_url`，全小写，中间是英文下划线。
- 这是 **KV namespace binding / 资源绑定**，不是新建一个普通的文字环境变量。
- Pages 项目与 KV 命名空间要在同一个 Cloudflare 账号下。
- 如果你要测试 Preview 预览环境，也要给 Preview 单独绑定；本教程先保证 Production 生产环境正确。

## 第 6 步：重新部署，让 KV 绑定生效

保存绑定后，要有一次新的生产部署，应用才能使用新资源。

1. 在 Pages 项目中打开 **Deployments / 部署**。
2. 找到最新的生产部署记录，点击右侧 **…** 菜单。
3. 选择 **Retry deployment / 重试部署**。如果你的界面提供 **Redeploy / 重新部署**，选择重新部署最新生产版本。
4. 等待新部署成功，再打开生产网址。

![重试部署的菜单位置示意](/blogs/cloudflare-pages-imgbed-guide/9da3602e49a3db9d.png)

不要只在浏览器里刷新页面；浏览器刷新不会让新的资源绑定进入部署。

## 第 7 步：首次进入后台，马上设置管理员密码

新建实例的认证还没有按你的需要配置。不要假设有通用的 `admin/admin` 初始密码；完成首次进入后，请立即设置属于自己的凭据。

1. 用 Pages 的生产网址打开项目，例如：

   ```text
   https://你的项目名.pages.dev/dashboard
   ```

   把 `你的项目名` 换成 Cloudflare 实际分配的项目名。也可以先打开首页，再从页面菜单进入管理后台。

2. 在管理界面进入 **系统设置 → 安全设置 → 认证管理**（具体分组名称可能随版本略有调整）。
3. 设置一个管理员用户名和强密码，然后保存。
4. 如果准备允许其他人使用上传页面，再设置用户端认证码。用户端认证码和管理员密码用途不同，建议设置为不同内容。
5. 保存后重新登录一次，确认管理员密码可用。再用浏览器无痕窗口访问，确认没有旧登录会话时也会要求认证。

不要把密码、认证码、Token 放在公开仓库、截图或教程示例里。完成后应妥善保管，忘记密码时查看项目官方 FAQ 的恢复流程。

## 第 8 步：在 R2 创建一个私有存储桶

R2 是实际保存图片和文件内容的地方。本教程让程序通过 Cloudflare 资源绑定访问桶；一般无需为了完成部署而把整个桶公开。

1. 在同一个 Cloudflare 账号打开 **R2 Object Storage / R2 对象存储**。
2. 点击 **Create bucket / 创建存储桶**。
3. 给桶取一个名称，例如 `my-imgbed-files-2026`。Cloudflare 如果提示名字已占用，就换一个。
4. 区域按控制台提供的选项选择；不确定时使用账户默认或推荐选项。
5. 点击创建，确认存储桶出现在列表中。

![Cloudflare R2 创建存储桶的项目文档示意](/blogs/cloudflare-pages-imgbed-guide/6530670cb5f3d97b.png)

![R2 存储桶创建表单](/blogs/cloudflare-pages-imgbed-guide/57858a6f42686a2b.png)

### 是否要开启公开访问？

先保持默认私有。ImgBed 通过 R2 绑定存取文件；只有在你要使用需要公网读取的特殊功能（例如项目文档提到的图片审查场景）或配置自定义公开域名时，再按需求设置公开访问。公开访问会让知道链接的人读取对象，不要为了“试试看”就开放整个桶。

R2 可能按 Cloudflare 当前规则产生费用。请在 Cloudflare 当前的 R2 计费说明中确认存储、操作次数和公网流量等项目，再决定是否用于长期存放大量文件。

## 第 9 步：把 R2 桶绑定到 Pages

1. 回到 **Workers & Pages → 你的 Pages 项目 → Settings / 设置 → Bindings / 绑定**。
2. 点击 **Add / 添加 → R2 bucket / R2 存储桶**。
3. **Variable name / 变量名称** 填写：

   ```text
   img_r2
   ```

4. 在存储桶列表中选中第 8 步创建的桶。
5. 保存到 Production / 生产环境。
6. 按第 6 步的方法再部署一次，让新的 R2 绑定生效。

![CloudFlare ImgBed 官方 R2 渠道配置示意](/blogs/cloudflare-pages-imgbed-guide/72adfbe97ac90f70.png)

变量名 `img_r2` 需要拼写准确。绑定保存后必须等新部署成功，之后再去管理后台测试。

## 第 10 步：在 ImgBed 管理后台添加 R2 上传渠道

完成 R2 绑定并重新部署后，还需要在 ImgBed 里面启用一个“上传渠道”。**绑定是让程序能访问 R2；添加渠道是告诉 ImgBed 上传时要用它。** 这两步都要做。

1. 登录 `/dashboard` 管理后台。
2. 打开 **系统设置 → 上传设置**。
3. 找到 **Cloudflare R2 / R2 渠道配置**，点击 **添加渠道**。
4. 给渠道起一个容易识别的名称，例如 `我的 R2`。
5. 在页面给出的 R2 配置中选择已通过 `img_r2` 绑定的 Cloudflare 桶，并启用该渠道。不同项目版本的表单字段可能略有差别，按页面提示选择绑定的桶；不要将 Access Key、Secret 之类的机密写入公开位置。
6. 如果表单有 **公开访问链接 / Public URL** 字段，普通绑定上传先留空；只有确实配置了 R2 公网域名并且功能需要时再填写。
7. 点击 **保存设置**，刷新页面，确认渠道仍存在并且已启用。

[查看项目官方 R2 渠道设置说明](https://cfbed.sanyue.de/deployment/configuration.html)

**如果 R2 渠道列表里没有桶：** 先检查 Pages 的 `img_r2` 绑定是否保存到了 Production，再确认绑定后做过新部署。资源绑定没生效时，管理后台无法从程序运行环境中使用对应存储桶。

## 第 11 步：上传一张小图片，做完整验收

先用一张小 JPG 或 PNG 测试。这样容易区分基础链路问题和大文件限制。

1. 打开图床首页，进入上传页面；如启用了用户端认证，输入刚设置的用户端认证码。
2. 在上传设置或渠道选择中选择刚创建的 **R2 渠道**。
3. 选择一张小图片，点击上传并等待成功提示。
4. 复制页面返回的图片 URL，在新浏览器标签页打开，确认显示的是图片。
5. 打开管理后台的文件管理，确认能看到文件记录和预览。
6. 刷新文件管理页面，再确认记录仍然存在。
7. 如需确认文件确实写入 R2，可以打开 Cloudflare R2 存储桶，检查是否出现新对象。

**部署完成标准：** Pages 生产部署成功；管理后台能登录；KV 记录可读写；至少一个 R2 上传渠道处于启用状态；测试图片上传成功且返回链接可以打开。

---

## 可选：绑定自己的域名

你可以先使用 `*.pages.dev` 地址，不需要域名也能完成教程。以后要绑定域名时：

1. 在 Cloudflare Dashboard 打开 **Workers & Pages → 你的 Pages 项目 → Custom domains / 自定义域名**。
2. 点击 **Set up a domain / 设置域名**，填写想使用的完整域名，例如 `img.example.com`。
3. 按 Cloudflare 提示确认 DNS。若使用子域名，按要求添加 CNAME；若使用根域名（例如 `example.com`），域名需要作为 Cloudflare Zone 并使用 Cloudflare Nameservers。
4. 等页面显示域名已生效后，再通过这个域名访问网站并测试登录、上传、图片链接。

先在 Pages 项目里添加域名，再按提示处理 DNS。只在 DNS 面板手工添加 CNAME、但没把域名加入 Pages 项目，可能无法正确路由。具体条件请看 [Cloudflare Pages 自定义域名文档](https://developers.cloudflare.com/pages/configuration/custom-domains/)。

## 常见问题：先按现象检查

| 现象 | 先检查什么 |
|---|---|
| Cloudflare 仓库列表里找不到项目 | Fork 是否完成；Cloudflare 的 GitHub 授权是否包含这个仓库；两边登录的是不是同一 GitHub 账号。 |
| 构建失败 | 打开构建日志，核对生产分支、`npm install`、`/frontend-dist` 和根目录；不要照搬旧教程的输出目录 `/`。 |
| 网站打开是 404 或空白 | 检查是否部署了正确分支、输出目录是否写成 `/frontend-dist`、最新部署是否成功。 |
| 后台提示未配置数据库或配置读取失败 | 检查是否建立了 KV 资源绑定、变量名是否为 `img_url`、是否绑定到生产环境、保存后是否重新部署。 |
| 绑定下拉框里找不到 KV/R2 | Pages 与 KV/R2 是否属于同一个 Cloudflare 账号；资源是否创建完成。 |
| 页面可以打开，但没有可用上传渠道 | 在 **系统设置 → 上传设置** 添加并启用 R2 渠道；只有绑定 R2 不等于已经添加上传渠道。 |
| 上传报 R2 相关错误 | 核对变量名 `img_r2`、桶是否选对、绑定后是否完成重新部署、后台渠道是否启用。 |
| 图片记录存在，图片链接打不开 | 检查上传渠道状态、返回 URL 是否完整、网络与域名过滤设置；不要立刻把整个桶公开。 |
| 刚保存的设置还没同步 | 稍等片刻再刷新。KV 在不同地区的读取更新可能不是瞬时的。 |

## 数据库替代方案：D1

本教程使用 KV。如果你决定使用 D1，请把它当作**另一条从头选择的数据库路线**：创建 D1、按项目当前文档执行完整 SQL、用变量名 `img_d1` 绑定，并重新部署。不要同时绑定 KV 和 D1，也不要指望它们自动同步。

D1 的初始化文件和索引要求可能随版本变化。请以 [CloudFlare ImgBed 当前 Pages 部署说明](https://cfbed.sanyue.de/deployment/pages.html) 和 [项目仓库的 `database` 目录](https://github.com/MarSeventh/CloudFlare-ImgBed/tree/main/database) 为准，不要从旧视频复制数据库 SQL。

## 参考资料

- [CloudFlare ImgBed：Cloudflare Pages 部署](https://cfbed.sanyue.de/deployment/pages.html)
- [CloudFlare ImgBed：前期准备](https://cfbed.sanyue.de/deployment/prerequisites.html)
- [CloudFlare ImgBed：配置说明（R2 渠道）](https://cfbed.sanyue.de/deployment/configuration.html)
- [CloudFlare ImgBed：常见问题](https://cfbed.sanyue.de/qa/)
- [Cloudflare 官方：Git 集成部署 Pages](https://developers.cloudflare.com/pages/get-started/git-integration/)
- [Cloudflare 官方：Pages 自定义域名](https://developers.cloudflare.com/pages/configuration/custom-domains/)
- [Cloudflare 官方：Pages Functions 绑定](https://developers.cloudflare.com/pages/functions/bindings/)

截图来源：CloudFlare ImgBed 项目文档，版权与项目文档页面说明一致。界面如有更新，请以当前控制台和上面的官方说明为准。
