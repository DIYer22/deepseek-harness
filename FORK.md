# DIYer22/deepseek-harness fork 说明

本 fork 只在上游 [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) 之上做最小改动：`dsh web` 默认监听所有网卡，并关闭 Host / Origin / `Sec-Fetch-Site` 校验。浏览器访问只靠启动时打印的 token 链接鉴权。上游 README 中关于 “wildcard 地址不受支持”“`trustedHosts` 决定可访问域名” 的描述不适用于本 fork。

## 改动

| 改动 | 文件 |
| --- | --- |
| `webserver` 接受 `0.0.0.0` / `::` 等通配地址；通配绑定时打印和打开的 URL 使用 `127.0.0.1`（或 `[::1]`） | `packages/host/webserver/src/index.ts` |
| `--host` 不再拒绝通配地址 | `packages/bundle/web-app/src/startup.ts` |
| Web profile 默认 `host` 从 `127.0.0.1` 改为 `0.0.0.0` | `packages/bundle/web-app/cordis.patch.yml` |
| `/api`、`/connect`、WebSocket 及复用该检查的路由不再校验 Host、Origin、`Sec-Fetch-Site`，未登录统一返回 401 | `packages/client/connection/src/rpc-host.ts` |

Desktop 应用固定传入 `--host 127.0.0.1`，不受影响。

## 使用

```sh
pnpm install
pnpm run build
pnpm dsh web            # 或 make web；默认监听 0.0.0.0:3080
```

启动后打印 `dsh web: http://127.0.0.1:3080/?token=...`。从其他地址访问时，只需把主机部分换成实际地址，保留 `?token=...`：

- 局域网：`http://192.168.x.x:3080/?token=...`
- 端口转发 / 反向代理：`http://<转发地址>:<端口>/?token=...`
- 只允许本机访问：`pnpm dsh web --host 127.0.0.1`

## 鉴权方式

- token 是每个进程启动时生成的 32 字节随机值，重启后变化。
- 打开 token 链接后，服务端写入 `HttpOnly; SameSite=Strict` 的签名 cookie，并重定向到不带 token 的地址。cookie 绑定当前访问的 `host:port`，有效期 30 天（`connection` 的 `cookieMaxAgeDays`），进程重启后仍有效。
- 每个不同的访问地址（如 `127.0.0.1:3080`、`192.168.1.5:3080`、转发后的 `example.com:8080`）都需要各自打开一次 token 链接。
- 局域网上的明文 HTTP 可能被嗅探到 token 和 cookie。在不可信网络上请使用 `--tls-cert/--tls-key`、HTTPS 反向代理或 SSH 隧道。
- 通过 LAN IP 访问的明文 HTTP 页面不是浏览器的 secure context，剪贴板等部分浏览器能力可能不可用；`ssh -L 3080:127.0.0.1:3080 <host>` 后访问 `http://127.0.0.1:3080/?token=...` 可以避免。

## 同步上游

```sh
git fetch upstream
git merge upstream/master
```

`origin` 指向本 fork（`git@github.com:DIYer22/deepseek-harness.git`），`upstream` 指向官方仓库。
