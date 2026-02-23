---
categories: [Developing Tools]
date:
  created: 2026-02-22
  updated: 2026-02-23
draft: false
comments: true
links: []
readtime: 8
slug: tmux-output-to-web
authors:
  - <qihang>
---

# tmux 输出映射网页

问题：能不能把 `tmux` 输出直接映射到网页里看，而且不一定要允许网页端操作 `tmux`？  
可以。核心是先区分目标：

1. 把终端“原样镜像”到网页
2. 只把输出当“日志文本流”推到网页

<!-- more -->

## Overview

[TOC]

## 一页结论

| 目标 | 推荐方案 | 为什么 |
|---|---|---|
| 想最快把 tmux 画面放网页里“只看不控” | `ttyd + tmux attach -r` | 开箱即用，配置最少 |
| 想嵌到自己的 dashboard，只展示输出 | `pipe-pane + SSE/WebSocket` | 可控、可二次加工 |
| 想展示“当前屏幕快照”而不是持续流 | `capture-pane` 轮询 | 实现简单、稳定 |

## 路线 A: 直接镜像 tmux 终端画面（推荐起步）

这类方案本质是把 TTY 渲染到浏览器（通常基于 xterm.js）。  
如果你只想“看”不想“操作”，就用只读。

### A1. `ttyd`（推荐，开箱即用）

`ttyd` 只有加 `-W/--writable` 才允许写入，默认就是只读；`tmux attach -r` 也是只读 attach。两层只读叠加，最省心。

```bash
# 将 mysession 只读映射到 http://<host>:7681
ttyd -p 7681 tmux attach -r -t mysession
```

建议加安全参数（本机绑定 + 认证 + Origin 检查）：

```bash
# 仅绑定本机 + 基本认证 + Origin 检查
ttyd -i 127.0.0.1 -p 7681 -c user:pass -O tmux attach -r -t mysession
```

如果你本来就在用 Tailscale，推荐把 `ttyd` 绑定到 `127.0.0.1`，再通过 Tailscale 或反向代理暴露，而不是直接开放公网端口。

### A2. `gotty`（常见替代）

```bash
gotty --port 12345 tmux attach -r -t mysession
```

同样通过 `tmux attach -r` 保持只读观看。和 `ttyd` 相比，它是可替代方案，不是首选。

## 路线 B: 只把 tmux 输出推给自定义网页（更可控）

如果你不需要完整终端 UI，只想在 dashboard 展示输出内容（日志、命令结果），`pipe-pane` / `capture-pane` 更灵活。

### B1. `pipe-pane`（实时流式）

```bash
# 将 pane 的新增输出持续追加到文件
tmux pipe-pane -o -t mysession:0.0 'cat >> /tmp/pane.log'

# 停止 pipe
tmux pipe-pane -t mysession:0.0
```

然后做一个小服务读取 `/tmp/pane.log`，通过 SSE 或 WebSocket 推到前端。

适合：`tail -f` 这类行式输出。  
限制：`top/htop/nvim` 这类全屏 TUI 会包含大量 ANSI 控制序列，前端展示往往会混乱。

### B2. `capture-pane`（截图式轮询）

```bash
# 抓取当前 pane 屏幕文本
tmux capture-pane -p -t mysession:0.0

# 抓取最近 2000 行（含 scrollback）
tmux capture-pane -p -t mysession:0.0 -S -2000
```

你可以在后端每 1 秒执行一次 `capture-pane` 并返回文本，前端轮询刷新。  
优点是实现简单、语义稳定；缺点是不是逐字符实时流。

## 最短可运行实现（`pipe-pane + SSE + HTML`）

如果你要快速做一个“只看输出”的网页，这个最小例子可以直接跑通。

1. 先把 tmux 输出持续写入日志：

```bash
tmux pipe-pane -o -t mysession:0.0 'cat >> /tmp/pane.log'
```

2. 创建 `server.py`：

```python
#!/usr/bin/env python3
import os
import time
from http.server import SimpleHTTPRequestHandler, ThreadingHTTPServer

LOG_FILE = "/tmp/pane.log"
HOST = "127.0.0.1"
PORT = 8765


class Handler(SimpleHTTPRequestHandler):
    def do_GET(self):
        if self.path != "/events":
            return super().do_GET()

        self.send_response(200)
        self.send_header("Content-Type", "text/event-stream; charset=utf-8")
        self.send_header("Cache-Control", "no-cache")
        self.send_header("Connection", "keep-alive")
        self.end_headers()

        try:
            with open(LOG_FILE, "a+", encoding="utf-8", errors="replace") as f:
                f.seek(0, os.SEEK_END)
                while True:
                    line = f.readline()
                    if not line:
                        time.sleep(0.2)
                        continue
                    data = line.rstrip("\n").replace("\r", "")
                    self.wfile.write(f"data: {data}\n\n".encode("utf-8"))
                    self.wfile.flush()
        except (BrokenPipeError, ConnectionResetError):
            pass


if __name__ == "__main__":
    httpd = ThreadingHTTPServer((HOST, PORT), Handler)
    print(f"Serving on http://{HOST}:{PORT}")
    httpd.serve_forever()
```

3. 创建 `index.html`：

```html
<!doctype html>
<meta charset="utf-8" />
<title>tmux pane stream</title>
<style>
  body { font: 14px/1.5 ui-monospace, SFMono-Regular, Menlo, monospace; margin: 24px; }
  pre { white-space: pre-wrap; word-break: break-word; }
</style>
<h1>tmux pane stream</h1>
<pre id="out"></pre>
<script>
  const out = document.getElementById("out");
  const es = new EventSource("/events");
  es.onmessage = (e) => {
    out.textContent += e.data + "\n";
    window.scrollTo(0, document.body.scrollHeight);
  };
  es.onerror = () => {
    out.textContent += "[disconnected]\n";
  };
</script>
```

4. 在同目录启动服务并打开网页：

```bash
python3 server.py
# 浏览器访问 http://127.0.0.1:8765
```

5. 结束时关闭 pipe：

```bash
tmux pipe-pane -t mysession:0.0
```

## 常见坑

- 只读观看优先用：`tmux attach -r`
- `ttyd` 不要加 `-W`，并尽量加：`-i 127.0.0.1 -c user:pass -O`
- 需要结构化输出就选 `pipe-pane`，需要屏幕快照就选 `capture-pane`
- 如果要显示全屏 TUI，请优先走“终端镜像”路线（A），不要直接渲染 `pipe-pane` 文本流
- `pipe-pane` 日志会持续增长，建议配合 logrotate 或定期清理

## 如何选择

- 要最快把 `tmux` 画面放上网页：`ttyd + tmux attach -r`
- 要嵌入自定义 dashboard，只展示关键输出：`pipe-pane + SSE/WebSocket`
- 要“接近截图”的当前屏幕展示：`capture-pane` 轮询

## References

- [ttyd Manual (Arch)](https://man.archlinux.org/man/extra/ttyd/ttyd.1.en)
- [tmux Advanced Use Wiki](https://github.com/tmux/tmux/wiki/Advanced-Use)
- [gotty + tmux 示例文章](https://franckpachot.medium.com/sharing-my-screen-to-the-web-from-oracle-free-tier-compute-instance-using-tmux-and-gotty-e6406c0203a8)
