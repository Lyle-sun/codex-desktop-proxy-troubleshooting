# Codex Desktop 代理与 WebSocket 排查路径

这份记录用于排查 macOS 上 Codex Desktop 出现以下现象：

- `Selected model is at capacity. Please try a different model.`
- `tls handshake eof`
- `WebSocket was closed before the connection was established`
- 对话卡住、重试或无法建立连接

## 先判断问题类型

不要先把 `Selected model is at capacity` 当成模型真的满载。先确认网络链路：如果 HTTP 代理可以建立 CONNECT，但 Codex 的 WebSocket 握手失败，容量提示可能只是连接失败后的表层错误。

本次问题的根因是：

1. 本地 DNS 对 `*.chatgpt.com` 的解析不可靠。
2. `ALL_PROXY=socks5://127.0.0.1:9099` 让客户端在本机解析域名。
3. WebSocket 拿到错误地址后，在 TLS 握手阶段被断开。

## 1. 检查本地代理入口

VPN 客户端应提供本机回环代理：

```text
HTTP 代理：127.0.0.1:9098
SOCKS5 代理：127.0.0.1:9099
```

推荐设置：

- 开启“设置系统代理”。
- 保持“允许其他设备连接”关闭。
- 不要随意更改 9098/9099 端口。

这一步是系统代理设置，不等于 TUN 或远程 DNS。若客户端支持 TUN、增强模式或远程 DNS，长期方案应优先启用这些能力。

## 2. 确认 macOS 系统代理

```bash
scutil --proxy
```

应看到类似结果：

```text
HTTPEnable : 1
HTTPProxy  : 127.0.0.1
HTTPPort   : 9098
HTTPSEnable: 1
HTTPSProxy : 127.0.0.1
HTTPSPort  : 9098
```

SOCKS 代理如果同时显示为启用也可以，但 Codex 的环境变量不要因此改成 `ALL_PROXY`。

## 3. 验证 HTTP CONNECT 链路

```bash
curl -sS -I --noproxy '' \
  --proxy http://127.0.0.1:9098 \
  --max-time 15 \
  https://chatgpt.com/ | head -n 12
```

正常时通常先看到：

```text
HTTP/1.1 200 Connection established
```

后续可能是 `401`、`403` 或 Cloudflare challenge。这些说明代理 CONNECT 和 TLS 链路已建立，不代表鉴权或网页挑战通过。

如果连 `200 Connection established` 都没有，先处理 VPN 客户端、端口或系统代理，不要继续调 Codex。

## 4. 检查 DNS 是否污染

分别查询本机 DNS 和可信的远程 DNS，比较返回地址：

```bash
dig +short chatgpt.com
dig +short ws.chatgpt.com
```

如果 VPN 客户端提供 DoH/DoT 或远程 DNS，再用它查询同一域名。两边结果明显不一致，且本机结果落到与目标无关的地址段时，说明本地解析路径有问题。

SOCKS 测试要区分本地解析和代理端解析：

```bash
curl -I --max-time 15 --proxy socks5://127.0.0.1:9099 https://chatgpt.com/
curl -I --max-time 15 --proxy socks5h://127.0.0.1:9099 https://chatgpt.com/
```

`socks5h://` 中的 `h` 表示由代理端解析 hostname。若 `socks5h://` 能返回 HTTP 响应，而 `socks5://` 失败，就不要让客户端使用会触发本地解析的 SOCKS 路径。

## 5. 设置命令行环境变量

在 `~/.zshrc` 中保留明确的 HTTP 代理：

```bash
export HTTP_PROXY=http://127.0.0.1:9098
export HTTPS_PROXY=http://127.0.0.1:9098
```

当前方案不设置：

```bash
ALL_PROXY=socks5://127.0.0.1:9099
```

原因不是 `ALL_PROXY` 永远不能用，而是不同客户端对 `ALL_PROXY` 的优先级、SOCKS 协议和 DNS 解析方式不一致。HTTP CONNECT 已经能把 hostname 原样交给代理，少一条歧义路径更稳定。

检查当前 shell：

```bash
source ~/.zshrc
printf 'HTTP_PROXY=%s\n' "$HTTP_PROXY"
printf 'HTTPS_PROXY=%s\n' "$HTTPS_PROXY"
printf 'ALL_PROXY=%s\n' "${ALL_PROXY-}"
```

不要在日志或截图中暴露 API key、token 或其他密钥；同时检查 `~/.zshrc` 是否误放了明文密钥。

## 6. 完全重启 Codex Desktop

旧进程不会自动更新环境变量。执行以下步骤：

1. `Cmd+Q` 完全退出 ChatGPT/Codex Desktop。
2. 确认没有残留主进程。
3. 重新启动应用。

先从终端启动可以验证 shell 环境：

```bash
open -a ChatGPT
```

然后再从 Dock 启动一次，验证系统代理路径。Dock 启动的主进程可能没有 `HTTP_PROXY` 环境变量，但开启系统代理后，Codex 子进程可能会获得：

```text
HTTP_PROXY=http://127.0.0.1:9098
HTTPS_PROXY=http://127.0.0.1:9098
```

检查进程环境时只输出代理地址，不要把完整环境变量复制到公开渠道：

```bash
pgrep -af '/Applications/ChatGPT.app/Contents/MacOS/ChatGPT'
ps eww -p <PID> | tr ' ' '\n' | grep -E '^(HTTP_PROXY|HTTPS_PROXY|ALL_PROXY)='
```

## 7. 验证真实对话链路

`codex exec` 主要走 HTTP SSE，不能单独证明 Desktop 的 WebSocket 回合。应在 Codex Desktop 中发送一条短消息，例如：

```text
Dock WebSocket 测试
```

验证标准：

- 消息正常返回。
- 没有新的 `tls handshake eof`。
- 没有新的 `failed to connect to websocket`。
- 没有绕过代理的直连 443。

日志位置通常是：

```text
~/Library/Logs/com.openai.codex/
```

可以按关键词检索：

```bash
rg -n -i \
  'responses_websocket|tls handshake eof|failed to connect to websocket|websocket' \
  "$HOME/Library/Logs/com.openai.codex"
```

某些版本不会直接记录 `responses_websocket`，此时以“Dock 启动、消息正常返回、旧的 TLS 错误不再出现”为端到端证据，不要把其他 `AppServerConnection` 内部记录直接等同于远端 `wss://chatgpt.com/backend-api/codex/responses`。

## 8. 可选的长期优化

如果 VPN 客户端支持，优先采用：

1. 远程 DNS 或 DoH/DoT。
2. TUN/增强模式。
3. 系统代理保持开启。
4. 命令行仅保留 `HTTP_PROXY` 和 `HTTPS_PROXY`。

这样可以从根上减少本地 DNS 污染，同时避免在每个 GUI 应用里重复注入 `ALL_PROXY`。

## 最小稳定配置

```text
系统代理：开启
允许其他设备连接：关闭
HTTP 代理：127.0.0.1:9098
HTTPS 代理：127.0.0.1:9098
SOCKS5：127.0.0.1:9099
HTTP_PROXY：http://127.0.0.1:9098
HTTPS_PROXY：http://127.0.0.1:9098
ALL_PROXY：不设置
```
