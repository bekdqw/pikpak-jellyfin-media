# PikPak + Jellyfin 完整搭建指南：如何用 PikPak 云盘驱动 Jellyfin 私人影音库？注册教程、WebDAV 配置、STRM 直链播放方案全解析（含套餐价格对比）

如果你也在折腾 Jellyfin，大概率遇到过这个灵魂拷问：视频资源放哪？

本地硬盘当然好，但贵，占地方，还得想着备份。于是很多人开始把眼光投向云盘——而在支持 Jellyfin 这件事上，**PikPak** 是一个值得认真研究的选项。

这篇文章就来把 PikPak Jellyfin 这个组合讲清楚：PikPak 是什么、怎么接入 Jellyfin、几种方案各有什么坑、套餐怎么选。

---

## PikPak 是什么？跟普通云盘有什么区别

很多第一次听说 PikPak 的人，会把它理解成"又一个网盘"，这个定位其实差了一口气。

PikPak 的核心卖点是**云端离线下载 + 私密云存储**，两个功能捆在一起卖。你把磁力链接、BT 种子、甚至 Twitter 上的视频链接扔进去，PikPak 的服务器帮你下好，存到你的云端空间，你再随时流媒体播放或者下载到本地。

换句话说，整个下载过程不占你本地的带宽，也不需要你的设备一直开着。文件静静地躺在云端，等你想看了就看。

对搭 Jellyfin 的人来说，这个逻辑很顺：资源自动入云盘 → 通过某种方式让 Jellyfin 认识这些文件 → 刮削元数据，建立海报墙 → 随时随地播放。

目前 PikPak 支持 Android、iOS、Windows、macOS、Web 以及 Telegram Bot，多平台协同，整个工作流打通起来还是比较顺的。

👉 [用邀请码注册 PikPak，免费体验 5 天 Premium](https://mypikpak.com?invitation-code=74098243)（注册时填入邀请码 **74098243**）

---

## PikPak 接入 Jellyfin 的三种方案

说直接的：PikPak 和 Jellyfin 的整合，目前社区里走得比较多的路子有三条，各有利弊。

### 方案一：Jellyfin 专用插件（不推荐）

GitHub 上有一个开源项目 `jellyfin-plugin-pikpak`，可以直接以插件方式把 PikPak 挂入 Jellyfin。听起来很美，但作者本人在 README 里写了一句大实话："**可以成功播放视频，但是 CPU 和内存负载高，不建议使用。**"

实测确实如此——视频能播，但服务器压力显著上升，在低配 NAS 或者树莓派上基本跑不稳。这个路子目前不是主流，了解即可。

---

### 方案二：PikPak WebDAV + Alist + Jellyfin（推荐，稳定）

这是当前最主流的方案，链路是这样的：

**PikPak → Alist（挂载 PikPak）→ Jellyfin（挂载 Alist WebDAV）**

**第一步：在 PikPak 开启 WebDAV**

登录 PikPak 网页版，进入「设置 → 实验室功能」，启用 WebDAV 选项。页面会生成一个统一的 WebDAV 连接地址，以及你的用户名和密码，记下来备用。

**第二步：在 Alist 里挂载 PikPak**

Alist 是一个开源的文件列表工具，支持几十种网盘的挂载，并统一对外提供 WebDAV 接口。Docker 部署最方便：

bash
docker run -d \
  --name alist \
  --restart unless-stopped \
  -p 5244:5244 \
  -v /your/path/alist:/opt/alist/data \
  xhofe/alist:latest


启动后访问 `IP:5244`，进入「存储 → 添加」，驱动选择 PikPak，填入账号密码，WebDAV 策略选择「302 重定向」（这个很重要，这样播放时视频流直接从 PikPak 服务器走，不绕经你的 NAS）。

**第三步：在 Jellyfin 里添加媒体库**

Jellyfin 本身支持直接挂载 WebDAV 作为媒体源，把 Alist 的 WebDAV 地址（`http://你的IP:5244/dav`）配进去，Jellyfin 就能看到 PikPak 里的所有文件，然后正常扫库、刮削、建立海报墙。

这个方案的缺点是链路稍长，偶尔会出现因为 Alist 中转导致的速度或稳定性问题，但整体可用性很好。

---

### 方案三：STRM 文件直链播放（进阶，体验最佳）

STRM 方案是目前追求极致体验的玩家最喜欢的路子。

STRM 文件本质上是一个文本文件，里面只有一行内容——媒体文件的直链 URL。Jellyfin 读到 STRM 文件后，会直接去请求那个 URL 播放，不做本地转码，也不占 NAS 带宽，视频流从 PikPak 服务器直达播放器。

操作流程：

1. 通过 Alist 挂载好 PikPak
2. 使用工具（如 `alist-strm`、`AutoFilm`、`SmartStrm` 等开源项目）扫描 Alist 上的媒体文件，批量生成对应的 `.strm` 文件到本地目录
3. 在 Jellyfin 里把这个本地目录添加为媒体库，正常刮削
4. 播放时 Jellyfin 读 STRM 文件里的直链，流媒体直出

这个方案的好处是：不需要 rclone 长期挂载（资源占用低）、播放时不占 NAS 带宽、对 NAS 性能要求极低。代价是需要定期同步更新 STRM 文件，可以通过定时任务自动化解决。

> 小提示：PikPak 的 WebDAV 地址国内可以直连，不需要额外折腾代理。但播放 STRM 直链时，取决于你所在地区的网络情况，有时可能需要科学环境，这点要有预期。

---

## PikPak 免费版 vs Premium：搭 Jellyfin 用哪个？

免费版有 6GB 空间，够测试，但不够用。要真的把 PikPak 当作 Jellyfin 的媒体仓库，Premium 是必要的，10TB 空间大约能放 8000 个高清视频文件。

以下是 PikPak 目前的套餐对比：

| 套餐 | 存储空间 | 价格 | 核心权益 | 购买链接 |
|------|----------|------|---------|---------|
| 免费版 | 6GB | 免费 | 基础云存储，下载速度受限，并发数受限 |  [注册免费账号](https://mypikpak.com?invitation-code=74098243) |
| Premium 月付 | 10TB | $10.0 / 月 | 无限速下载，高优先级任务队列，4K 流媒体播放 |  [立即订阅月付](https://mypikpak.com/drive/payment?invitation-code=74098243) |
| Premium 年付 | 10TB | $100.99 / 年（约 $8.4 / 月） | 同月付全部权益，按年计费约省 16% |  [立即订阅年付](https://mypikpak.com/drive/payment?invitation-code=74098243) |

> 注册时填写邀请码 **74098243**，可免费体验 5 天 Premium 权限（含 10TB 空间）。

对长期用户来说，年付方案划算很多。有人算过：10TB 年付约合每 TB 每月 $0.83，放在同等容量的云存储产品里属于相当有竞争力的价位。

---

## 几个常见问题

**Q：PikPak 的 WebDAV 在国内能用吗？**

能。PikPak 虽然屏蔽了中国大陆 IP 的直接访问，但 WebDAV 接口是对会员开放且可以从国内直连的，这是不少人选择它配 Alist 使用的主要原因之一。

**Q：Jellyfin 刮削 PikPak 里的资源需要怎么命名文件？**

和本地文件一样，遵循 Jellyfin 的标准命名规范：电影用 `电影名 (年份).mkv`，剧集用 `剧集名/Season 01/剧集名 S01E01.mkv`。命名正确的话，刮削成功率很高。

**Q：STRM 方案和直接挂载 WebDAV 哪个更好？**

取决于你的 NAS 配置和使用习惯。NAS 性能弱、追求低资源占用，选 STRM；对实时同步要求高、不想额外维护 STRM 脚本，选 WebDAV 直挂。两个都试试，感受一下哪个更顺手。

**Q：rclone 挂载和 Alist 有什么区别？**

rclone 可以把 PikPak（通过 Alist 的 WebDAV 接口）挂载成本地磁盘，Jellyfin 当作本地目录来读取；Alist 本身则提供 Web 界面和 WebDAV 接口，让 Jellyfin 通过 WebDAV 协议访问。两种方式都有人用，rclone 挂载后 Jellyfin 无感知（就像本地盘），稳定性依赖挂载进程持续运行。

---

## 搭建 PikPak + Jellyfin 的建议路线

如果你是新手，从这条路走：

1. 👉 [注册 PikPak 账号](https://mypikpak.com?invitation-code=74098243)，注册时填入邀请码 **74098243** 获得 5 天免费 Premium 体验
2. 部署 Alist（Docker 最简单），在 Alist 里添加 PikPak 驱动，WebDAV 策略选 302 重定向
3. 部署 Jellyfin（官方 Docker 镜像），添加媒体库时指向 Alist 的 WebDAV 地址
4. 等 Jellyfin 扫库完成，海报墙就自动出来了
5. 如果体验稳定，可以考虑升级到 STRM 方案进一步优化

整套系统跑起来之后，你的 Jellyfin 就有了一个云端的、可以自动填充新内容的媒体仓库，体验确实比纯本地硬盘方案灵活不少。

👉 [开始注册 PikPak，用邀请码 74098243 解锁 5 天 Premium 试用](https://mypikpak.com?invitation-code=74098243)
