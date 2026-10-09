# 纯照片相册方案：独立核查与评估

> 核查日期：2026-10-09。本轮只做了研究和评估，没有安装、购买、部署、修改云权限，也没有联系买家。
> 状态标记：✅ 现成支持 ｜ ⚙️ 需要配置 ｜ 🛠 需要二次开发 ｜ ❓ 尚未验证 ｜ ❌ 不满足

## 0. 结论摘要

1. **没有一个现成开源系统能完整覆盖全部需求。** 最接近的是 Lychee 7.10（MIT 许可，完整源码）。它缺两项：**每册独立背景音乐**完全没有；**手动拖拽排序**也没有，只能按标题、拍摄时间等字段排序。另外，**微信内原图保存**需要专门适配。
2. **“支持 S3”不等于“腾讯云 COS 已验证”。** Lychee 访问私有桶时，用的是 AWS SDK 生成的 SigV4 预签名 URL。腾讯云官方的“用 AWS S3 SDK 访问 COS”示例（Python）配置的却是 `signature_version = s3`。我没有找到官方材料确认 SigV4 预签名在 COS 中国香港地域可用。而且上游源码把 S3 分支标成了 `@codeCoverageIgnore`，也就是上游 CI 不测这条路径。**这是整个 Lychee 路线的最大风险，动工前必须用买家账号实测。**
3. **原方案是否过度设计：无法逐条判断。** 仓库是空的，对话里只有“4800 元”这个数字，没有 Codex 方案原文。第 5 节先按需求列出“必要”和“可删”的部分，拿到原文后可以逐项对照。
4. **更简单但可靠的做法：**做一个单进程的小应用，用 SQLite，上传时在服务器端生成预览图，通过**腾讯云官方 COS SDK** 签发短时效 URL，用 Caddy 自动配置 HTTPS，整体只有一个 docker compose。它的工作量和“Lychee + 二次开发”差不多（约 50–80 小时）。好处是 COS 走官方 SDK，兼容风险更低；而且不用长期维护一个 Lychee 分支。
5. **600 元做不全。** 按估算工时 50–80 小时折算，相当于每小时 7.5–12 元。600 元最多能覆盖“Lychee 部署 + 配置 + COS 兼容性实测”（前提是实测通过）。背景音乐、拖拽排序、单张授权、微信原图页这几块都不在范围内，缺口见第 7 节。安全要求不随预算降低。

## 1. 核查范围与方法说明

- 需求清单如下（视频、文字编辑、二维码生成、OSS 兼容明确不做）：电脑管理后台、按册密码、腾讯云香港 COS 私有存储、自动生成压缩预览、单张原图授权下载、固定相册链接、排序、轮播、每册独立背景音乐。云资源用买家账号，费用由买家承担。
- **Lychee** 的结论都来自直接阅读源码。读的是正式发布版 `v7.10.0`（2026-09-28），同时对照了 master（`35a330c`，2026-10-09；master 里已经混入 v8 的开发代码）。源码只读，没有运行。
- **腾讯云、微信、Chevereto、Ente** 的结论来自官方文档的搜索摘要。本环境的网页抓取工具没法直接打开 cloud.tencent.com 等站点，所以价格类数字一律标注“需在官网定价页复核”。
- 文中没有引用任何“成交行情”，因为我没有可靠的行情数据。

## 2. 候选方案筛选

| 候选 | 许可证 | 结论 |
|---|---|---|
| **A. Lychee 7.10** | MIT（含 SE 代码） | 最接近，入选 |
| **B. 自研轻量系统**（官方 COS SDK） | 自有代码 + MIT/Apache 依赖 | 入选，作为“更简单”的对照 |
| **C. Ente（自托管）** | AGPL-3.0 | 安全性最强但最重，入选作对照 |
| Chevereto | Free 版是 AGPL；Lite/Pro 是专有许可 | **排除。**官方外部存储文档要求存储端提供 *public-read* URL，与“私有 COS”冲突。社区给出的私有桶方案是在前面加一层能签名的 CDN，不是程序自带功能。2026 年价格未能确认 |
| Piwigo + Protected Albums 插件 | GPL-2.0 | **排除。**插件页标注“与最新版 Piwigo 不兼容”；插件说明自己承认，受保护相册的照片仍能通过搜索、标签、日历访问到；论坛也有绕过的报告。没找到 S3 插件 |
| Immich / PhotoPrism / WordPress / Halo 等 | — | 本轮未深入核查，不作结论 |

## 3. 逐项对比

| 需求 | A. Lychee 7.10 | B. 自研轻量 | C. Ente 自托管 |
|---|---|---|---|
| 电脑管理后台 | ✅ Web 后台，自带 zh_CN 语言包 | 🛠 | ✅ 桌面端/Web 端（以“个人相册账号”形态出现） |
| 按册密码 | ✅ 公开相册可设密码。解锁接口限流 10 次/分钟（`routes/api_v2.php:68`）。“同密码连带解锁其他相册”的选项 `enable_propagate_unlock_option` 默认关闭，**必须保持关闭** | 🛠 | ✅ 公开链接可加密码、有效期、设备数上限 |
| 香港 COS 私有存储 | ⚙️ + ❓ 需配置 `S3_ENABLED=true`、`AWS_IMAGE_VISIBILITY=private`、`AWS_ENDPOINT`，以及 `AWS_USE_PATH_STYLE_ENDPOINT=false`（2024 年起新建的 COS 桶禁用 path-style）。**SigV4 预签名在 COS 上是否可用尚未验证** | ⚙️ 用官方 `cos-nodejs-sdk-v5` 签名，属官方文档路径；仍需实测 ❓ | ❓ museum.yaml 支持 S3 兼容存储，客户端用预签名 URL 直传桶，需要配置 CORS。COS 未验证 |
| 自动生成压缩预览 | ✅ 服务器端生成 thumb/small/medium，再上传到 S3 | 🛠 用 sharp 生成 | ✅ 客户端生成，加密存储 |
| 单张原图授权下载 | ✅ 按**相册**授权：`grants_full_photo_access` 关闭时，接口不下发原图 URL（`SizeVariantsResouce.php:33,50`）。注意：没有 medium 尺寸的小图不会降级。🛠 若要“逐张单独授权”需开发 | 🛠 | ✅ 按链接设置“允许下载”；逐张授权 🛠 |
| 固定相册链接 | ✅ 相册 ID 固定，另有 slug（2026-02 迁移加入） | 🛠 | ⚙️ 链接里带密钥片段，URL 很长；用自定义域名需另起一个 albums Web 实例 |
| 排序 | ✅ 可按标题、拍摄时间、上传时间、评分等字段排序；🛠 **没有手动拖拽排序**（`ColumnSortingPhotoType` 枚举里没有 manual）。权宜办法是用标题前缀如“001_” | 🛠 用 SortableJS | ❓ |
| 轮播 | ✅ 自带 slideshow | 🛠 用 PhotoSwipe | ❓ |
| 每册背景音乐 | 🛠 **没有。**源码里的 Album “Track” 是 GPX 轨迹文件（`GpxFileRule`），不是音频 | 🛠 | 🛠（AGPL 大型代码库，改动成本高） |
| 微信内原图保存 | 🛠 + ❓ 下载按钮在微信内置浏览器里通常会失效，需要改成“原图页 + 长按保存”的引导 | 🛠 同样做原图页 + 长按引导 | ❌/❓ 原图在浏览器里解密成 blob，微信长按保存 blob 图片能否成功未验证；JS 触发的下载会被微信拦截 |
| 许可证与付费功能 | ✅ MIT。SE 独占功能（去品牌、水印、统计、AES 链接、用户组等）需要通过 GitHub Sponsors 取得密钥；本需求**用不到**，不建议改代码绕过 | ✅ 依赖都是宽松许可：PhotoSwipe MIT、SortableJS MIT、sharp Apache-2.0（libvips 为 LGPL）、cos-nodejs-sdk-v5（npm 元数据标为 ISC） | ⚠️ AGPL-3.0：可以商用，但修改后对外提供服务必须公开源码；交付完整源码本身没问题 |
| 部署与迁移 | ⚙️ 需要 PHP 8.4 + Imagick，官方提供 Docker 镜像。迁移内容 = 数据库 + `.env`，COS 桶保持不变。另有 `lychee:s3_migrate` 命令。上游发版频繁，**二次开发等于长期维护一个分支** | ⚙️ docker compose + SQLite 文件 + `.env`，最轻 | ⚙️ Go 服务端 + Postgres + 两个 Web 实例 + 桶 CORS，最重 |

**小结：**

- **A（Lychee）**：现成功能最多，但卡在三处：COS 兼容性未验证、背景音乐要开发、拖拽排序要开发。
- **B（自研）**：每项都要写代码，但依赖最少，COS 走官方路径，可控性最高。
- **C（Ente）**：安全模型最强，可是对本需求属于过度设计，微信原图保存风险最高，不推荐。

## 4. 重点核对项

### 4.1 私有素材鉴权（不随预算降低）

> **补充（审阅 Codex V3/V4 后）：**按本单规模，**由应用鉴权后转发 COS 素材**比直接下发 COS 签名 URL 更简单，也更稳。这样改密码后可以立即撤销访问，完全避开 4.2 所述的默认域名和备案问题，也不需要 CORS。代价是访客流量从主机出网。下面关于签名 URL 的几条，只在采用直签方式时适用。详见 `v4-review.md`。

- 桶设为**私有读写**，不开公有读。所有素材都通过服务器签发的短时效 URL 访问：预览约 1 小时，原图约 10–30 分钟。原图有效期要覆盖用户在页面停留到长按保存的这段时间。
- 预览和原图**存成不同的对象**。如果改用数据万象在下载时实时缩放，腾讯云官方建议**把处理参数一起纳入签名**；否则别人去掉参数就能用同一签名取到原图。
- 服务器使用 CAM 子账号密钥，只授权本桶需要的 Get/Put/Delete 操作。密钥不能进前端。本轮**没有**改动任何权限。
- 相册密码用 bcrypt/argon2 哈希存储；接口限流；Cookie 设为 HttpOnly + Secure + SameSite；解锁状态按相册隔离。管理员登录也要限流，全站只走 HTTPS。
- 固定链接里的相册标识用不可猜的随机串（≥10 位），不要用自增 ID 或拼音。

### 4.2 COS 默认域名限制（官方公告，2024-01-01 起执行）

- 2024 年起新建的桶，通过默认域名访问时，任何文件都会被加上 `Content-Disposition: attachment` 强制下载，而且禁用 path-style 访问。
- 这对 `<img>` 标签内嵌显示有没有影响，官方没有专门说明。❓ 需要实测。稳妥做法是绑定**自定义源站域名**。
- 自定义域名的备案要求：官方表述是“用于中国大陆地域存储桶时必须备案”；香港桶是否免备案，以控制台提示为准 ❓。
- 自定义域名加 SigV4 预签名时，签名里的 Host 必须和实际访问的域名一致。Lychee 目前的配置不能直接解决这一点 ❓🛠。

### 4.3 微信原图保存

- 微信内置浏览器一般会拦截 `a[download]`、blob 和 JS 触发的下载（来源是开发者社区文章，非官方文档）。可行做法：用 `<img>` 显示原图，提示用户“长按保存”；识别到普通浏览器时再给附件下载链接。
- `wx.downloadImage` 只接受微信服务器的 serverId，不适用于这个场景。JS-SDK 还需要公众号和安全域名，社区资料普遍说要求已备案域名，这一点在微信官方文档中未能核实 ❓。
- 以下几点需要在 iOS 和 Android 真机上实测 ❓：长按保存的是否原始字节（尺寸和 EXIF 是否保留）；20MB 以上大图的表现；未备案的香港域名在微信里会不会被拦截或弹出提示。

### 4.4 背景音乐

- iOS 微信对自动播放有限制。常见做法是在 `WeixinJSBridgeReady` 事件里调用 `play()`，同时**必须保留“点击播放”按钮作为兜底**。资料比较旧，需要真机实测 ❓。

## 5. 原方案是否过度设计

因为没有原文，下面先按需求划出“必要”和“可删”，拿到 Codex 方案后可以逐项对照：

- **必要：**私有桶 + 短时效签名、预览与原图分开存储、按册密码 + 限流、子账号最小权限、HTTPS、数据库备份、真机微信测试。
- **通常可删或推迟：**
  - 前后端分离 SPA + Redis/消息队列/微服务（单管理员、低并发用不上）；
  - 浏览器 STS 直传（单管理员可以由服务器中转上传，省掉 CORS 和 STS 配置）；
  - 首期就上 CDN 或 WAF；
  - 多用户和角色权限；
  - 任何 OSS、视频、二维码、文字编辑相关的抽象层（需求已明确排除）；
  - Kubernetes。

## 6. 工作量与费用估算依据

### 6.1 开发与测试工时（以 B 方案为基准；A 方案见备注）

| 模块 | 小时 |
|---|---|
| 项目骨架、配置、部署脚本（docker compose + Caddy） | 5 |
| 管理员登录、相册增删改、按册密码、固定链接 | 8 |
| 上传 → 生成预览 → 写入 COS（官方 SDK）、删除、签名 URL | 8 |
| 拖拽排序 | 3 |
| 前台相册页 + PhotoSwipe 轮播 + 自适应布局 | 6 |
| 单张原图授权 + 微信原图页 | 4 |
| 每册背景音乐（上传、签名播放、微信兜底） | 3 |
| 安全加固（限流、Cookie、CAM 策略文档、桶策略检查） | 4 |
| 自动化测试（鉴权、签名、越权） | 4 |
| 真机测试：iOS/Android 微信 + 桌面浏览器 | 6 |
| 修 bug、回归测试 | 6 |
| 交付文档、迁移说明、备份恢复演练 | 4 |
| **合计** | **约 61 小时（区间 50–80）** |

**A 方案（Lychee）备注：**

- 部署与配置约 4–6 小时；COS 兼容性实测 4–8 小时，如果失败，这条路线就走不通。
- 背景音乐（前后端都要改，新建音频模型和签名）12–20 小时；拖拽排序 6–10 小时；微信原图页 4–8 小时；逐张授权（如需要）6–10 小时；测试 10 小时以上。
- 合计约 45–70 小时，另加长期跟进上游版本的维护成本。

### 6.2 一次性费用（买家承担）

- 软件许可：A/B 都是 0 元（MIT 或自有代码），不需要 Lychee SE。
- 域名注册费：以注册商页面为准（未查价）。HTTPS 证书用 Let's Encrypt，免费。

### 6.3 后续费用（买家承担，按量）

- **COS 中国香港标准存储：0.136 元/GB/月**（来自 2024 年降价通知，需复核）。例如 50GB 约 6.8 元/月。新用户有 50GB × 6 个月的标准存储免费包，但**不抵扣流量**。
- **COS 外网下行流量：未查到香港地域的官方单价**，需在定价页确认。用量估算公式：月流量 ≈ 访客数 × 浏览张数 × 预览大小 + 原图下载次数 × 原图大小。例如 100 访客 × 30 张 × 0.3MB + 20 次 × 8MB ≈ 1GB/月。
- 数据万象基础图片处理：每月免费 10TB，超出约 0.025 元/GB。B 方案在服务器端生成预览，可以完全不用。
- 香港服务器：公开资料互相矛盾，既有 24–32 元/月的旧促销价，也有文章称起步价 335 元/月。本报告**不采用任何数字**，需要买家在购买页确认。服务器配置建议 1 核 2G 起（Lychee 建议 2G 以上内存）。

## 7. 600 元预算的缺口

- 按 50–80 小时折算，600 元相当于每小时 7.5–12 元。能否接受由卖方自己判断，本报告不评价市场价。
- **600 元能做到的现实范围（A 方案）：**Lychee 部署、私有 COS 配置与实测、安全配置、基础交付文档。实现的是：后台、按册密码、预览、按相册的原图授权、固定链接、按字段排序、轮播。
- **明确缺口：**
  1. 每册背景音乐；
  2. 手动拖拽排序；
  3. 逐张原图授权（如果需求指的是逐张单独授权）；
  4. 微信内原图保存的适配页面；
  5. 完整的真机回归测试。
  6. 如果 COS 的 SigV4 实测失败，600 元也不够切换到 B 方案。
- 不能为了凑预算而降低以下要求：公有读桶、长期有效 URL、明文密码、前端暴露密钥、跳过限流。

## 8. 动工前必须用买家账号做的验证（本轮未执行）

1. 在香港地域新建一个测试桶，分别用 AWS SDK（SigV4）和官方 COS SDK 签发 GET/PUT 预签名 URL，看能否正常访问。
2. 分别用默认域名和自定义源站域名测试：`<img>` 能否内嵌显示、响应头是什么、HTTPS 证书是否正常、是否要求备案。
3. iOS/Android 微信真机测试：长按保存原图（核对尺寸和 EXIF）、背景音乐自动播放、未备案香港域名能否打开。
4. 确认香港地域的外网下行流量单价和服务器实际价格。

## 9. 来源

**源码与项目文档（直接阅读）**
- Lychee 源码（v7.10.0 / master）：https://github.com/LycheeOrg/Lychee
  - `config/filesystems.php`（S3 磁盘配置）、`app/Services/UrlGenerator.php`（私有桶 `temporaryUrl`）、`app/Actions/Album/Unlock.php`、`routes/api_v2.php`、`app/Http/Resources/Models/SizeVariantsResouce.php`、`app/Enum/ColumnSortingPhotoType.php`、`app/Http/Requests/Album/SetAlbumTrackRequest.php`（GpxFileRule）、`database/migrations/2026_01_22_000000_propagate_unlock_option.php`、`database/migrations/2026_02_27_000001_add_slug_to_base_albums.php`
- Lychee 文档仓库：https://github.com/LycheeOrg/LycheeOrg.github.io （configuration.md 中的 S3/AWS 变量；se/overview.md 中的 SE 功能与 MIT 许可）
- Lychee 配置文档：https://lycheeorg.dev/docs/getting-started/configuration/
- Lychee SE 说明：https://lycheeorg.dev/docs/se/overview/
- Ente 许可证（AGPL-3.0）与服务端配置：https://github.com/ente-io/ente
- Ente 对象存储：https://ente.io/help/self-hosting/administration/object-storage
- Ente 公开链接：https://help.ente.io/photos/features/sharing-and-collaboration/public-links
- npm 依赖许可：https://www.npmjs.com/package/cos-nodejs-sdk-v5 、https://www.npmjs.com/package/photoswipe 、https://www.npmjs.com/package/sortablejs 、https://www.npmjs.com/package/sharp

**腾讯云**
- COS 存储桶域名安全管理通知：https://cloud.tencent.com/document/product/436/96243
- 域名合规 FAQ（path-style）：https://cloud.tencent.com/document/faq/436/102489
- 生成预签名 URL：https://cloud.tencent.com/document/product/436/35560
- 使用 AWS S3 SDK 访问 COS：https://www.tencentcloud.com/document/product/436/32537
- 自定义源站域名：https://cloud.tencent.com/document/product/436/36638
- 存储容量降价通知（香港 0.136 元/GB/月）：https://cloud.tencent.com/document/product/436/102430
- 存储容量费用：https://cloud.tencent.com/document/product/436/53482
- 流量费用：https://cloud.tencent.com/document/product/436/53863
- COS 免费额度：https://cloud.tencent.com/document/product/436/6240
- 数据万象基础图片处理（私有读签名、参数纳入签名）：https://cloud.tencent.com/document/product/460/72972
- 数据万象免费额度：https://cloud.tencent.com/document/product/460/36381
- 数据万象费用示例：https://cloud.tencent.com/document/product/460/36380

**排除项与微信相关**
- Chevereto 外部存储（public-read 要求）：https://v3-docs.chevereto.com/features/integrations/external-storage.html
- Chevereto 私有桶讨论：https://chevereto.com/community/threads/unable-to-load-pictures-uploaded-to-s3-compatible-external-storage.15005
- Chevereto 许可变更：https://chevereto.com/community/threads/changes-in-license-terms.16264
- Piwigo Protected Albums 插件：https://piwigo.org/ext/index.php?eid=597
- 微信 H5 下载限制（开发者社区，非官方）：https://developer.aliyun.com/article/906395
- 微信音频自动播放（开发者社区，非官方）：https://developer.aliyun.com/article/637205
- 香港轻量服务器价格（第三方，互相矛盾，未采用）：https://cloud.tencent.com/developer/article/2657551 、https://oldtang.com/9956.html
