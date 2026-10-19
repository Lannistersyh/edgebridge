# EdgeBridge

**Edge 内嵌常驻浏览器桥 —— 让 AI Agent 像人一样操作你正在用的浏览器，零调试端口、零授权弹窗。**

```
Agent ──Unix socket──> 原生宿主(node) ──stdio 帧──> Edge/Chrome 扩展(MV3) ──chrome.debugger──> 标签页
```

## 为什么再做一遍这个轮子

| | 本项目 | BrowserSkill (9.2k★) | cdp-bridge-mcp (305★) |
|---|---|---|---|
| Agent→扩展通道 | **native messaging + Unix socket** | WebSocket + TCP 端口 | WebSocket + TCP 端口 (18765) |
| TCP 端口 | **0 个** | 1 个（daemon） | 2 个（ws + http） |
| 依赖 | **node 即可，无构建** | Rust daemon + pnpm 构建 | Python + uv + bottle |
| 代码量 | **~350 行** | 数万行 | 数千行 |
| 鉴权 | 文件系统权限（0600 socket） | handshake 协议 | 静态 token |
| 真实键鼠 | ✓（Input.dispatch* 含 paste 命令） | ✓ | ✓ |
| 截图 | ✓ 视口 + 全页 | ✓ + 录屏 | ✓ |
| 授权弹窗 | **装扩展时 1 次，永久** | 装扩展 1 次 | 装扩展 1 次 |

核心差异：**不开任何 TCP 端口**。WebSocket 方案意味着本机任意进程都能连上那个端口接管浏览器；本项目唯一通道是一个 0600 权限的 Unix socket，由浏览器拉起的宿主进程持有，文件系统权限即鉴权。

## 借鉴来源（都在源码注释中标注）

- **Tencent/BrowserSkill**（9216★）：`system.handshake` 式协议握手（版本+能力协商）、指数退避重连（0.5s→8s 封顶）、视口/全页截图实现
- **Unagi-cq/cdp-bridge-mcp**（305★）：`ext_ready` 就绪广播（连接建立即上报标签页清单）、token 字段位

## 能力

| 命令 | 作用 |
|---|---|
| `tabs` | 列出所有标签页 |
| `open <url>` | 打开标签页（等待加载） |
| `activate <id>` / `close <id>` | 切换 / 关闭标签页 |
| `eval '<js>'` | 在匹配标签页执行 JS（returnByValue） |
| `cdp <method> <json>` | 任意 CDP 命令透传（Input.dispatchKeyEvent、Page.reload…） |
| `shot [file]` | 视口截图 → PNG 文件 |
| `watch` | 事件流（连接就绪、页面加载完成等） |
| `detach` | 解除所有 debugger 附加 |

`--match <子串>` 可按 URL 选取目标标签页。

## 安装

### 1. 加载扩展

1. `edge://extensions` → 开启右下角**开发人员模式**
2. 点**加载解压缩的扩展** → 选本仓库 `extension/` 目录
3. 确认扩展 ID 与下方宿主清单中的 `allowed_origins` 一致（manifest 内置 `key`，ID 固定）

### 2. 安装原生宿主

```bash
# macOS（Edge 用户级）
mkdir -p ~/Library/Application\ Support/Microsoft\ Edge/NativeMessagingHosts
cp native/com.edgebridge.app.json ~/Library/Application\ Support/Microsoft\ Edge/NativeMessagingHosts/
# 注意：json 里的 path 需是本仓库的绝对路径，装好后核对一下
```

Chrome 用户把目录换成 `~/Library/Application Support/Google/Chrome/NativeMessagingHosts/`。

### 3. 自检

```bash
./status.sh
```

## 用法示例

```bash
node client.mjs tabs
node client.mjs open 'https://example.com'
node client.mjs eval 'document.title' --match example.com
node client.mjs shot /tmp/page.png --match example.com

# 真实粘贴（系统剪贴板 + Cmd+V）
printf 'a\tb\tc' | pbcopy
node client.mjs cdp Input.dispatchKeyEvent '{"type":"keyDown","key":"v","code":"KeyV","windowsVirtualKeyCode":86,"nativeVirtualKeyCode":9,"modifiers":4,"commands":["paste"]}' --match example.com
node client.mjs cdp Input.dispatchKeyEvent '{"type":"keyUp","key":"v","code":"KeyV","windowsVirtualKeyCode":86,"nativeVirtualKeyCode":9,"modifiers":4}' --match example.com
```

## 注意

- `chrome.debugger` 附加标签页时，浏览器会在该页顶部显示一条"正在调试"提示条（Chromium 硬性行为，非弹窗）。
- 同一标签页只能被一个调试器附加；DevTools 已打开时会附加失败。
- MV3 服务进程空闲约 30s 回收，已用 alarms + 端口长连接双保活。
- 卸载扩展后桥即失效；宿主进程随连接断开自动退出，不留后台残留。

## 目录

```
extension/    MV3 扩展（manifest.json + background.js）
native/       host.js（协议实现）、host.sh（绝对路径包装）、com.edgebridge.app.json（注册清单）
client.mjs    Agent 侧客户端
status.sh     自检
```

## License

MIT
