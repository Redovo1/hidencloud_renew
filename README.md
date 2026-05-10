# HidenCloud Auto Renew (Python版)

![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-Automated-green.svg)
![License](https://img.shields.io/badge/License-MIT-yellow.svg)

基于 Python 编写的 HidenCloud (海敦云) 自动续期与支付脚本，专为 GitHub Actions 设计。

✨ **核心亮点：使用 GitHub Actions Cache 持久化 Cookie，零外部依赖、天然私密，公开仓库也能放心用。**

## 🚀 功能特性

- **自动续期**：自动检测服务剩余时间，到期前自动延长 7 天。
- **自动支付**：续期后自动识别账单并完成 0 元支付（或余额支付）。
- **Cookie 持久化（全新方案）**：
  - 使用 **GitHub Actions Cache** 跨运行保存最新 Cookie
  - 每次运行自动读取 / 写回，`hidencloud_session` 等字段永不过期
  - **天然私密**：即使仓库公开，Cache 内容也只有本仓库的 workflow 能访问
  - **零维护**：不用注册第三方云盘、不用担心 WebDAV 401、不用处理 App Password 过期
  - （可选）仍兼容 Infinicloud WebDAV，配置了就用，没配置就走 Cache
- **多账号支持**：支持 `HIDEN_COOKIE` / `HIDEN_COOKIE_2` / ... 最多 20 个账号独立配置。
- **消息推送**：集成 WxPusher，任务完成后通过微信即时通知（可选）。
- **失败重试**：网络波动或 Cookie 失效时自动回退重试，并延时再跑一次 Job。

## 🛠️ 部署指南

### 1. 准备工作

*   **GitHub 账号**：用于 Fork 本项目。
*   **HidenCloud 账号**：获取初始 Cookie。
*   （可选）**WxPusher**：[注册地址](https://wxpusher.zjiecode.com/admin/)，用于接收微信通知。

### 2. 获取 HidenCloud Cookie

每个账号都需要单独拿一份 Cookie，**建议用无痕窗口避免互相覆盖**：

1. 无痕窗口打开 HidenCloud 控制台并登录。
2. 按 `F12` 打开开发者工具，切到 **Network** 标签。
3. 刷新页面，点第一个请求（通常是 `dashboard`），在 **Request Headers** 找到 `Cookie`。
4. 右键 → **Copy value**，整串复制（⚠️ 不要手动拖选，容易漏或带 `…`）。

至少要包含这三项：`hidencloud_session`、`cf_clearance`、`hc_cf_turnstile`。

### 3. 配置 GitHub Secrets

仓库 → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

#### 必填

| 变量名 | 说明 |
| :--- | :--- |
| `HIDEN_COOKIE` | 第 1 个账号的完整 Cookie |

#### 多账号（可选，有几个号加几个）

| 变量名 | 说明 |
| :--- | :--- |
| `HIDEN_COOKIE_2` | 第 2 个账号的 Cookie |
| `HIDEN_COOKIE_3` | 第 3 个账号的 Cookie |
| ... | 最多支持到 `HIDEN_COOKIE_20` |

> 💡 每个变量内部也支持用换行或 `&` 分隔多个 Cookie（兼容老用法）。

#### 微信推送（可选）

| 变量名 | 说明 |
| :--- | :--- |
| `WP_APP_TOKEN_ONE` | WxPusher 应用 Token |
| `WP_UIDs` | WxPusher 用户 UID（多人用逗号/分号/换行分隔） |

#### Infinicloud WebDAV（可选，**不推荐新用户配置**）

| 变量名 | 说明 |
| :--- | :--- |
| `WEBDAV_URL` | Infinicloud WebDAV 地址 |
| `WEBDAV_USER` | Connection ID |
| `WEBDAV_PASS` | Apps Password |

> ⚠️ **不填这三个就会自动走 GitHub Cache，更稳更省心。** 填了会同时启用，但 App Password 有过期风险。

### 4. 启动运行

1. 仓库顶部 → **Actions** 标签
2. 左侧选 **HidenCloud Auto Renew (Python)**
3. 右上角 **Run workflow** 手动触发一次
4. 检查日志里是否出现：
   ```
   📦 共加载 N 个账号
   [账号 1] ✅ 登录成功，发现 X 个服务。
   ```

之后脚本按 cron 自动运行，每日自动刷新 Cookie、到期前自动续期。

## 🔐 安全说明

- **公开仓库能用吗？** 能。代码、workflow 都是公开的，但：
  - Cookie 存在 **GitHub Secrets** 中，只有仓库管理员可见，代码里无法读出明文
  - 新 Cookie 存在 **GitHub Actions Cache** 中，外人无法访问，只有本仓库 workflow 能读写
  - **不会写入任何仓库文件**，push 历史不会泄露
- **Cookie 会不会自动更新？** `hidencloud_session`、`XSRF-TOKEN` 等会自动刷新并保存。
- **什么时候需要手动更新？** `cf_clearance`（Cloudflare 验证）大约 1-2 个月过期一次，过期后需手动重登取新 Cookie 更新 Secret。建议在 GitHub Settings → Notifications 开启 Actions 失败邮件提醒。

## 📂 文件结构

```text
.
├── .github/workflows/
│   ├── main.yml        # 主续期 workflow（含 Cache 读写）
│   └── cron.yml        # 随机化 cron 时间，分散运行负载
├── main.py             # Python 核心运行脚本
├── requirements.txt    # Python 依赖库
└── README.md           # 说明文档
```

## ❓ 常见问题

**Q: 我之前配了 Infinicloud，现在 401 了怎么办？**
A: 直接删掉 `WEBDAV_URL` / `WEBDAV_USER` / `WEBDAV_PASS` 三个 Secret，脚本会自动切到 GitHub Cache 方案。

**Q: 新加第三个账号会影响前两个吗？**
A: 不会。账号按变量序号独立缓存（`HIDEN_COOKIE` → 索引 0，`HIDEN_COOKIE_2` → 索引 1 ...），**只能追加不能中间插**，否则会错位。

**Q: 连续 7 天没跑，缓存会丢吗？**
A: GitHub Cache 有 7 天未访问自动清理策略。但默认每天都跑，正常情况下不会触发。即使丢了，下次会自动用 Secret 里的 Cookie 重新建立缓存。

## ⚠️ 免责声明

1. 本项目仅供学习交流使用，请勿用于非法用途。
2. 脚本涉及账号操作，作者不对因使用本脚本造成的账号封禁、资金损失等后果负责。
3. 请妥善保管你的 Secrets 配置，不要分享给他人。

---
**如果这个项目对你有帮助，请给一个 Star ⭐️**
