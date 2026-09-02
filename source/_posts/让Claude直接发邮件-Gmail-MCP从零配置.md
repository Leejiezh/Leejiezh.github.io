---
title: 让 Claude 直接发邮件：Gmail MCP 从零配置
top: false
cover: false
img: /medias/featureimages/GmailMCP从零配置文章封面图生成需求.png
toc: true
mathjax: true
date: 2026-09-02 11:00:00
password:
summary: 从零配置 Gmail MCP，让 Claude Code 直接完成发件、读件、搜索、标签管理，附 OAuth 授权全流程与三大踩坑实录。
tags:
    - Claude Code
    - MCP
    - Gmail
categories:
    - 工具
---

## 一、为什么要让 Claude Code 发邮件

那天我想让 AI 帮我搜一下宋亚东 vs 乌马尔的比赛数据，然后把结果发到我邮箱。三种方案摆在面前：手抄、导出文件让用户转发、配置 MCP 让 AI 走 Gmail API。

我没选前两个。手抄意味着每次都要人来中转。导出文件本质上还是人肉操作。配置 MCP 是唯一"一次配好、长期复用"的路径。

MCP 全称 Model Context Protocol，是 Anthropic 推的让 AI 进程和外部工具对话的协议。配置完成后，AI 调用一组 `mcp__gmail__` 工具就能完成发件、读件、搜索、标签管理这些事。整套链路对人是透明的。

## 二、准备 Google Cloud 项目与 OAuth 凭据

1. 打开 https://console.cloud.google.com，新建一个项目（项目名随意，我用 claude-gmail-mcp）

   {% asset_img image-20260902090853916.png 新建Google Cloud项目 %}

   {% asset_img image-20260902090943794.png 项目创建完成 %}

2. 在「API和服务 → API库」里搜 Gmail API，启用。

   {% asset_img image-20260902091359507.png 启用Gmail API %}

3. 下一步是关键：去「API和服务 → 凭证 → 创建凭证 → OAuth客户端ID」创建一个 OAuth 客户端ID，类型必须是 Desktop app。

   {% asset_img image-20260902091702442.png 创建OAuth客户端ID %}

   {% asset_img image-20260902091832866.png 选择Desktop app类型 %}

   为什么必须是 Desktop app？因为 Google 的 Desktop app 类型自带 http://localhost 回调地址，OAuth 流程中会启动本地 HTTP server 监听这个地址接回调码，整个授权可以在本机闭环完成，不需要你部署一个公网可访问的回调服务。Web application 类型则需要你自己挂回调，对本地开发机是反人类的。

   {% asset_img image-20260902092126391.png OAuth客户端创建完成 %}

   创建后下载 JSON，凭据里会有 client_id 和 client_secret。但下载下来的格式不一定能直接用，需要在 `C:\Users\leeji\.gmail-mcp\gcp-oauth.keys.json` 落盘成 Google "installed" 标准格式：

   ```json
   {
       "installed": {
           "client_id": "xxx.apps.googleusercontent.com",
           "client_secret": "GOCSPX-xxx",
           "auth_uri": "https://accounts.google.com/o/oauth2/auth",
           "token_uri": "https://oauth2.googleapis.com/token",
           "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
           "redirect_uris": ["http://localhost"]
       }
   }
   ```

## 三、选定 MCP server 包

**MCP选择：**

npm 上现成的 Gmail MCP 不少，我选了 [@gongrzhe/server-gmail-autoauth-mcp](https://github.com/GongRzhe/Gmail-MCP-Server)。原因有三个：

- 第一，它自带 auth 子命令，能独立完成首次 OAuth 流程，不需要写胶水脚本。
- 第二，它内置 HTTP server 监听 localhost:3000 接 OAuth 回调，授权码换 token 一气呵成。
- 第三，它用 google-auth-library，会自动用 refresh_token 续期 access_token，token 过期无需人工干预。

**配置写入：**

Claude Code 支持三个层级：项目级 `.mcp.json`、用户级 `~/.claude.json` 顶层 `mcpServers`、企业级 managed-mcp.json。

我选用户级顶层。理由很简单：Gmail 跟具体项目无关，跨项目复用价值高；不污染任何仓库；改起来就改一个 JSON 文件。

打开 `C:\Users\leeji\.claude.json`，在顶层加：

```json
  "mcpServers": {
    "gmail": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@gongrzhe/server-gmail-autoauth-mcp"],
      "env": {
        "HTTPS_PROXY": "http://127.0.0.1:7897",
        "HTTP_PROXY": "http://127.0.0.1:7897"
      }
    }
  }
```

这个 **env** 块的核心作用是：让 Gmail MCP 那个 Node.js 子进程走本地代理去访问 Google，因为你的网络环境（中国大陆家庭宽带/公司网络）默认直连 Gmail API 是被墙的，连不上就等于 MCP 没法工作。

*强烈建议：*改之前先备份。`cp ~/.claude.json ~/.claude.json.bak.20260901-150703`。如果 JSON 写错导致 Claude Code 启动失败，备份能救命。

## 四、OAuth 首次授权

第一次运行需要在 PowerShell 里手动跑 auth 子命令：

```powershell
$env:HTTPS_PROXY="http://127.0.0.1:7897"
$env:HTTP_PROXY="http://127.0.0.1:7897"
npx -y @gongrzhe/server-gmail-autoauth-mcp auth
```

{% asset_img image-20260902101810909.png 运行auth子命令 %}

npx 会拉包、启动 MCP server，识别到 auth 子命令后做三件事：

1. 在控制台打印授权 URL
2. 调系统默认浏览器打开 Google 授权页
3. 启动 HTTP server 监听 localhost:3000 等回调。

在 Google 授权页点同意后，浏览器会跳到 `http://localhost:3000/oauth2callback?code=...`，本地 HTTP server 收到这个 code，POST 到 https://oauth2.googleapis.com/token 换出 access_token 和 refresh_token，**写入 ~/.gmail-mcp/credentials.json**。

> 注意：access_token 通常 1 小时过期，但 refresh_token 可以长期使用，access_token 过期时 google-auth-library 会自动用 refresh_token 换新的，全程对用户透明。

授权完成后，关掉当前 Claude Code 会话、重启。Claude Code 启动时会读 ~/.claude.json 顶层 mcpServers，把 gmail MCP server 作为子进程拉起来。这一次不再执行 auth 分支，直接进入 stdio 主循环。重启后工具列表里出现 `mcp__gmail__` 一整组工具。

## 五、踩坑实录

真正折磨我的是下面这几个坑。

### 坑 1：OAuth 回调"无法访问"

第一次跑 `npx ... auth`，Google 授权页同意后浏览器跳到 `http://localhost:3000/oauth2callback?code=...`，我看到页面打不开。我第一反应是怀疑端口冲突、怀疑回调地址配错。反复验证后发现根本不是这些：我在跑 auth 命令的 PowerShell 里已经 Ctrl+C 退出了进程，HTTP server 早就停掉了。

正确做法：粘贴 URL 前不要关终端。或者粘贴到任何能访问本机的浏览器都行，只要在授权时限内（一两分钟）保持进程在跑就够。

### 坑 2：ETIMEDOUT 74.125.195.95:443

第二次跑 `npx ... auth`，屏幕上出现：

```
Server error: GaxiosError: request to https://oauth2.googleapis.com/token failed, reason: connect ETIMEDOUT 74.125.195.95:443
```

表层原因：GFW 屏蔽了 oauth2.googleapis.com、gmail.googleapis.com 这些域名，直接 connect 会超时。

但 GFW 不是全部原因。我已经开着 Clash Verge，浏览器访问 Google 一切正常。为什么 Node 进程就连不上？

**关键点：**Node 不读 Windows 系统代理。Windows 在「Internet 属性 → 连接 → 局域网设置」里配的代理只对 WinHTTP 生效，对 Node 进程不生效。Node 的 gaxios 库（被 google-auth-library 内部使用）只读 `HTTPS_PROXY` 和 `HTTP_PROXY` 这两个环境变量。

我的 Clash Verge 跑在 127.0.0.1:7897，只在系统层挂载了 HTTP 代理，Node 进程完全感知不到。

验证手段：跑一行 curl 走代理试试。`curl -v https://oauth2.googleapis.com --proxy http://127.0.0.1:7897`，返回 401。401 恰好说明"代理 + Google 链路"是通的，Google 只是因为没带 token 才拒绝的。如果直接 curl 不带 --proxy 也会超时，那就更印证了"Node 进程不读系统代理"的判断。

解决：把代理写到 mcpServers.gmail.env 里：

```json
"env": {
    "HTTPS_PROXY": "http://127.0.0.1:7897",
    "HTTP_PROXY": "http://127.0.0.1:7897"
}
```

这样 Claude Code 每次拉起 MCP 子进程时都会自动注入这两个环境变量，不需要我每次手动 export。

### 坑 3：HTTPS_PROXY 设在错误位置

第一轮我只在跑 npx auth 的 PowerShell 终端里临时设了 `$env:HTTPS_PROXY`，没写到 mcpServers.gmail.env 里。

结果：auth 流程能跑通（因为终端 export 影响了当前 shell），但关掉终端、用 Claude Code 调 send_email 时又超时（因为 Claude Code 拉起的 MCP 子进程拿不到这个 env）。

教训：auth 时要在终端临时设，运行时必须写在 mcpServers 配置里。两层都要。

## 六、调通后的使用

调通后整条链路就顺了。我在 Claude Code 里直接调用 mcp__gmail__send_email：

> to: leejieqaq@163.com
> subject: Song Yadong vs Umar Nurmagomedov 比赛详情
> body: <完整的比赛数据>

Gmail API 返回 messageId: 1a05be662242fad4，发送完成。163 邮箱在几秒内收到（可能在「垃圾邮件」分类里，163 偶尔会把 Gmail 发来的邮件归到那里，记得查一下）。

这次配置好之后，下次再让 AI 发邮件不用做任何额外操作。credentials.json 里的 access_token 过期时，google-auth-library 会自动用 refresh_token 换新的。refresh_token 本身长期有效（除非用户在 Google 账户安全页里主动 revoke）。

## 七、同样套路配置其他 Google 系 MCP

这套配置逻辑不只适用于 Gmail。

任何走 googleapis.com 的 MCP（Google Calendar、Google Drive、Google Sheets、Notion 等等）都需要同样处理：Google Cloud Console 建项目、启用对应 API、创建 Desktop app OAuth client、跑 auth 子命令、在 mcpServers 里加 HTTPS_PROXY。

三个固定文件位置：

- ~/.gmail-mcp/gcp-oauth.keys.json（凭据，不同 SaaS 用不同子文件夹）
- ~/.gmail-mcp/credentials.json（token，OAuth 流程自动生成）
- ~/.claude.json 顶层 mcpServers.\<name\>（告诉 Claude Code 怎么拉 MCP server）

env 通用项就是 HTTPS_PROXY + HTTP_PROXY，Clash Verge 用户填 7897，SSR/V2RayN 用户查自己本机实际代理端口替换。

*Token 自动续期原理：*google-auth-library 在每次 HTTP 请求前检查 access_token 是否过期（通常 1 小时），如果过期就自动用 refresh_token 调 https://oauth2.googleapis.com/token 换新，然后把新 token 写回 credentials.json。整个过程对调用方完全透明。
