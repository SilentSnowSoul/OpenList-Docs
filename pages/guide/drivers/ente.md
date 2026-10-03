---
title:
  en: Ente
  zh-CN: Ente
icon: iconfont icon-state
# This control sidebar order
top: 702
# A page can have multiple categories
categories:
  - guide
  - drivers
# A page can have multiple tags
tag:
  - Storage
  - Guide
  - 'Proxy'
---

<!--@include: @/snippets/reverse-tip.md-->

::: en
Read-only mounts for Ente, the end-to-end encrypted photo service. Two drivers are provided:

- **EnteShare**: public share URL mount; the root directory lists every file of that album.
- **Ente**: account mount; each album of the account is a folder under the root directory.

Both drivers are strictly read-only — upload, delete and rename are always rejected. Files and thumbnails are streamed as decrypted plaintext through the OpenList proxy; no direct link to the client is ever exposed, so the account credentials stay on the server.
:::

::: zh-CN
Ente（端到端加密相册）的只读挂载，提供两个驱动：

- **EnteShare**：公开分享 URL 挂载，根目录列出该分享相册内的全部文件。
- **Ente**：账号挂载，账号下的每个相册对应根目录下的一个文件夹。

两个驱动均为严格只读——上传、删除、重命名一律被拒绝。文件与缩略图以解密后的明文流经 OpenList 代理，绝不向客户端暴露直连，账号凭证因此始终留在服务端。
:::

## EnteShare configuration { lang="en" }

## EnteShare 配置 { lang="zh-CN" }

::: en

| Field | Required | Default | Description |
| --- | --- | --- | --- |
| `share_url` | yes | — | Public share URL, `https://share.ente.io/c/<token>#<key>`. The fragment carries the album key and is never sent to the server. |
| `endpoint` | no | `https://api.ente.com` | API endpoint of a self-hosted museum server. |

`device_token` is maintained by OpenList itself: the link-device token issued by the server is persisted into the storage configuration so the device slot is reused instead of consumed again. It is hidden from the storage form.
:::

::: zh-CN

| 字段 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- |
| `share_url` | 是 | — | 公开分享 URL，形如 `https://share.ente.io/c/<token>#<key>`。fragment 内为相册密钥，不会发送给服务器。 |
| `endpoint` | 否 | `https://api.ente.com` | 自托管 museum server 的 API 地址。 |

`device_token` 由 OpenList 自行维护：服务端下发的 link-device token 会持久化到存储配置中，从而复用设备位而非重复占用。该字段在存储表单中隐藏。
:::

## Ente configuration { lang="en" }

## Ente 配置 { lang="zh-CN" }

::: en

| Field | Required | Default | Description |
| --- | --- | --- | --- |
| `endpoint` | no | `https://api.ente.com` | API endpoint of a self-hosted museum server. |
| `email` | no | — | Account email. One of the credential combos below is required: `email + password`, `token + password`, `token + master_key`. |
| `password` | no | — | Account password. Set it together with `email` for SRP login, or together with `token` to decrypt the keys locally. |
| `two_fa_secret` | no | — | TOTP secret for accounts with two-factor enabled; the verification code is generated automatically. |
| `token` | no | — | App token, sent as `X-Auth-Token`. Filled in automatically after password login. |
| `master_key` | no | — | base64 master key; derives every album's collection key. Used with `token` when `password` is not set. |
| `secret_key` | no | — | base64 box private key. Shared albums deliver their collection key in a sealed box addressed to your public key, so they are skipped with a warning unless this is set. |
| `show_hidden` | no | `false` | Hidden albums (private magic metadata `visibility == 2`) are excluded unless this is enabled. |
| `root_folder_path` | no | `root` | Root folder path of the mount. |

In password mode the master key and secret key are derived in memory and never persisted; only the token is written back to the configuration.
:::

::: zh-CN

| 字段 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- |
| `endpoint` | 否 | `https://api.ente.com` | 自托管 museum server 的 API 地址。 |
| `email` | 否 | — | 账号邮箱。需填写以下凭证组合之一：`email + password`、`token + password`、`token + master_key`。 |
| `password` | 否 | — | 账号密码。与 `email` 同填走 SRP 登录，或与 `token` 同填在本地解出密钥。 |
| `two_fa_secret` | 否 | — | 开启两步验证账号的 TOTP secret，验证码自动生成。 |
| `token` | 否 | — | app token，以 `X-Auth-Token` 发送。密码登录后自动写回。 |
| `master_key` | 否 | — | base64 的 master key，用于解出各相册的 collection key。未设置 `password` 时与 `token` 搭配使用。 |
| `secret_key` | 否 | — | base64 的 box 私钥。共享相册的 collection key 以 sealed box 送达本人公钥，缺省时该相册被跳过并输出告警日志。 |
| `show_hidden` | 否 | `false` | 隐藏相册（私有 magic metadata `visibility == 2`）默认排除，开启后展示。 |
| `root_folder_path` | 否 | `root` | 挂载的根文件夹路径。 |

密码模式下 master key 与 secret key 仅在内存中派生，不会持久化；只有 token 会被写回存储配置。
:::

## Obtaining credentials { lang="en" }

## 获取凭证 { lang="zh-CN" }

## Credential modes { lang="en" }

## 凭证模式 { lang="zh-CN" }

::: en
Three credential combinations are accepted:

- **`email + password`** — full SRP login. The token is obtained automatically and written back; if the token is later revoked, login is retried with the stored credentials.
- **`token + password`** — the token fetches the key attributes and the password derives the KEK locally to decrypt the master key and secret key. Since no email is configured, a revoked token cannot fall back to SRP and returns a readable error asking you to refresh the token or add the email.
- **`token + master_key`** — credential mode, as before. Existing configurations keep working without migration.

Two-factor accounts: with `two_fa_secret` set, the TOTP code is submitted automatically. Email OTP is not supported — use the OpenList-APIPages Ente login page to obtain the credentials in that case.

::: warning Password login is not available on the Worker
The Worker runtime cannot perform SRP login due to memory limits. When `token` is empty and `email` or `password` is set, initialization fails with a readable error. `token + password` (without email) is still accepted and works like pure credential mode. For full password support use the OpenList Go backend or obtain the credentials via the APIPages login page.
:::
:::

::: zh-CN
支持三种凭证组合：

- **`email + password`** — 完整 SRP 登录。token 自动获取并写回；token 被吊销后会用已存凭证自动重登。
- **`token + password`** — 用 token 获取 keyAttributes，密码在本地派生 KEK 解出 master key 与 secret key。因未配置 email，token 失效时无法回退 SRP，会返回提示刷新 token 或补充 email 的可读错误。
- **`token + master_key`** — 即原有凭证模式，存量配置无需迁移即可继续使用。

两步验证账号：配置 `two_fa_secret` 后验证码自动提交。邮箱 OTP 不支持，此时请使用 OpenList-APIPages 的 Ente 登录页获取凭证。

::: warning Worker 不支持密码登录
Worker 运行时因内存限制无法执行 SRP 登录。当 `token` 为空且 `email` 或 `password` 任一非空时，初始化会返回可读错误。`token + password`（无 email）仍被接受，行为与纯凭证模式一致。如需完整密码登录，请使用 OpenList Go backend 或经 APIPages 登录页获取凭证。
:::
:::

## Known limitations { lang="en" }

## 已知限制 { lang="zh-CN" }

::: en

- **Sequential decryption, no random access.** The Ente secretstream format is a chained stream: every chunk is decrypted with a key derived from the previous chunk, so a byte range can only be served by decrypting from the start of the file and discarding the prefix. Seeking in a large video re-downloads and re-decrypts everything before the seek point. Such requests are still answered with `206` and a `Content-Range` header — the range is honoured, but the cost of reaching it is unchanged.
- **Strictly read-only.** Every write operation is rejected.
- **Dedupe suffix sits before the extension.** On a name clash `日落.jpg` becomes `日落 (2).jpg`, not `日落.jpg (2)`.
- **Password-protected shares are not supported.** EnteShare returns an explicit error for them; use a share without a password.
- **Device slot risk (EnteShare).** The link-device token is persisted into the storage configuration to reuse the device slot. A token that is never consumed — for example because the isolate was recycled — cannot be detected and may cause the share's device limit to be exceeded.
- **Owner-side failures.** When the share owner's subscription has lapsed (`410`) or the device limit is exceeded (`403`), the error is surfaced to the administrator as readable text.
:::

::: zh-CN

- **顺序解密，不支持随机访问。** Ente 的 secretstream 是链式流：每个分片用由前一个分片派生出的密钥解密，因此某个字节区间只能从文件开头解密并丢弃前缀来提供。在大视频中拖动进度条会重新下载并解密拖动点之前的全部内容。这类请求仍以 `206` 与 `Content-Range` 响应——区间被正确满足，但到达该区间的代价不变。
- **严格只读。** 一切写操作被拒绝。
- **同名去重后缀加在扩展名之前。** 冲突时 `日落.jpg` 生成 `日落 (2).jpg`，而非 `日落.jpg (2)`。
- **密码保护分享不支持。** EnteShare 遇到密码保护的分享 URL 会返回明确错误，需改用无密码分享。
- **设备位风险（EnteShare）。** link-device token 会持久化到存储配置中以复用设备位。未被消费的 token（例如 isolate 被回收）无法被检测，可能超出分享的设备数上限。
- **owner 侧限制。** 分享所有者订阅失效（`410`）或设备数超限（`403`）时，错误以可读文案返回给管理员。
:::
