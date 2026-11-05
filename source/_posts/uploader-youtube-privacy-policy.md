title: Uploader Youtube 隐私政策 / Uploader Youtube Privacy Policy
date: 2026-11-05 19:55:00
tags:
- privacy policy
---

**最后更新日期 / Last Updated:** 2026-11-05

Uploader Youtube（以下简称“本应用”）是 zvz.im 作者自用的工具。它在作者自己的电脑上运行，通过 YouTube Data API 把作者制作的视频上传到作者自己的 YouTube 频道，不向公众提供服务。本隐私政策说明本应用如何处理经 Google 账号授权获得的信息。

Uploader Youtube ("the App") is a personal tool used by the author of zvz.im. It runs on the author's own computer and uploads the author's own videos to the author's own YouTube channel through the YouTube Data API. It is not offered as a service to the public. This Privacy Policy explains how the App handles information obtained through Google account authorization.

<!--more-->

------

## 一、YouTube API 服务 / YouTube API Services

本应用使用 YouTube API 服务。使用本应用即表示您同意受 [YouTube 服务条款](https://www.youtube.com/t/terms) 的约束。Google 如何处理数据，请参阅 [Google 隐私权政策](https://www.google.com/policies/privacy)。

The App uses YouTube API Services. By using the App, you agree to be bound by the [YouTube Terms of Service](https://www.youtube.com/t/terms). For how Google handles data, see the [Google Privacy Policy](https://www.google.com/policies/privacy).

------

## 二、本应用访问的信息 / Information the App Accesses

经频道所有者授权（权限范围：管理 YouTube 账号），本应用只做以下事情：

- 读取所连接频道的 ID、名称和“上传的视频”播放列表 ID
- 上传视频，并设置标题、简介、标签、分类、语言、公开状态和定时公开时间
- 读取本应用上传的视频的处理状态和公开状态，并修改它们的公开状态和定时公开时间
- 读取频道的播放列表名称以找到同名播放列表，找不到时新建一个，并把上传的视频加入其中
- 上传意外中断时，查看频道最近上传的视频，确认那次上传是否已经完成

本应用不读取评论、订阅者、观看记录、数据分析或账号的其他任何信息。

With the channel owner's permission (scope: manage your YouTube account), the App only:

- Reads the connected channel's ID, title, and the ID of its uploads playlist
- Uploads videos and sets their title, description, tags, category, language, privacy status, and scheduled publish time
- Reads the processing and privacy status of the videos it uploaded, and changes their privacy status and scheduled publish time
- Reads the channel's playlist titles to find a playlist by name, creates one if none matches, and adds uploaded videos to it
- When an upload is interrupted, looks at the channel's recent uploads to confirm whether that upload completed

The App does not read comments, subscribers, watch history, analytics, or any other account information.

------

## 三、信息的存储 / How Information Is Stored

- OAuth 客户端凭据和刷新令牌只保存在作者电脑的 macOS 钥匙串中；访问令牌只保存在内存中。
- 每个上传片段的记录（文件哈希、YouTube 视频 ID、标题、上传和公开状态、定时公开时间、播放列表 ID）保存在作者电脑本地，与对应的视频放在同一个任务文件夹中。
- 本应用没有服务器或云端数据库。数据只在作者的电脑与 Google 的接口之间通过 HTTPS 传输。

- The OAuth client credentials and the refresh token are stored only in the macOS Keychain on the author's computer. Access tokens are kept in memory only.
- A record of each uploaded clip (file hash, YouTube video ID, title, upload and privacy status, scheduled publish time, playlist ID) is stored locally on the author's computer, in the same task folder as the video it describes.
- The App has no server or cloud database. Data travels only between the author's computer and Google's APIs, over HTTPS.

------

## 四、信息的使用与共享 / How Information Is Used and Shared

上述信息只用于把视频上传和发布到所连接的频道。本应用不会出售、出租或与任何第三方共享这些信息，不将其用于广告或用户画像；本应用不包含第三方内容或广告，也不让第三方收集信息或设置 Cookie。

本应用对从 Google API 获得的信息的使用和转移，遵守 [Google API 服务用户数据政策](https://developers.google.com/terms/api-services-user-data-policy)，包括其中的有限使用（Limited Use）要求。

This information is used only to upload and publish videos to the connected channel. The App does not sell, rent, or share it with any third party, and does not use it for advertising or profiling. The App shows no third-party content or advertisements, and lets no third party collect information or set cookies.

The App's use and transfer of information received from Google APIs adheres to the [Google API Services User Data Policy](https://developers.google.com/terms/api-services-user-data-policy), including the Limited Use requirements.

------

## 五、保留与删除 / Retention and Deletion

- 刷新令牌一直保存到频道所有者在本应用中断开连接为止。断开时，本应用会向 Google 撤销该令牌，并将其从钥匙串中删除。
- 视频的状态记录在每次同步时从 YouTube 刷新，作者可以随时删除这些本地记录。
- 您也可以随时在 [Google 账号安全设置](https://security.google.com/settings/security/permissions) 中撤销本应用的访问权限。

- The refresh token is kept until the channel owner disconnects in the App. On disconnecting, the App revokes the token with Google and deletes it from the Keychain.
- Video status records are refreshed from YouTube on every sync, and the author can delete these local records at any time.
- You can also revoke the App's access at any time on the [Google security settings page](https://security.google.com/settings/security/permissions).

------

## 六、隐私政策更新 / Changes to This Policy

我们可能会不定期更新本隐私政策。更新内容将发布在本页面。

We may update this Privacy Policy from time to time. Updates will be posted on this page.

------

## 七、联系我们 / Contact Us

如果您对本隐私政策有任何疑问，请联系：

If you have any questions regarding this Privacy Policy, please contact:

📧 Email: [i@zvz.im](mailto:i@zvz.im)

------

© 2026 赵文周. All rights reserved.
