# DeepSeek Harness (dsh) 插件手册

dsh Web 插件生态手册:插件清单、安装方法,以及公网部署(登录认证 + HTTPS)的标准方案。

## 插件机制速览

- 插件挂在 **profile** 上(默认 `~/.dsh/profiles/<name>`),`dsh --profile <name> --port <端口>` 启动。
- 安装/移除:`dsh plugin --profile <name> add <包名>` / `remove <包名>`,会同时写进 `package.json` 的 `dsh.profile.bundles` 并执行 pnpm 安装。
- 精细配置:profile 目录下的 `cordis.patch.yml` 做 id 定向覆盖。**注意:patch 会整体替换目标条目的 config**,覆盖时必须完整重述所有字段(含 `!!js` 表达式)。
- 本地/仓库安装:`link:/本地路径`(开发联动)、`file:/路径`(快照)、`github:<owner>/<repo>[#ref]`。
- 新 profile 需自带 `@deepseek-ai/dsh-base` 与 `@deepseek-ai/dsh-web-app` 两个基础 bundle,否则插件会卡在 "waiting for services: webServer, webStartup"。

## 功能 / UI 插件

| 插件 | 功能 | 安装 |
|---|---|---|
| `@linxin666/dsh-client-ui-task-board` | 任务看板:真实会话执行 + Host 定时调度 | `dsh plugin --profile <p> add @linxin666/dsh-client-ui-task-board` |
| `@linxin666/dsh-client-ui-skin-center` | 皮肤中心:内置 + `$DSH_HOME/skins` 皮肤,页面内试穿切换 | 同上,换包名 |
| `@linxin666/dsh-client-ui-git-graph` | Git 分支选择器 + 提交图(真实 git 操作) | 同上 |
| `dsh-better-sidebar` | VSCode 风右侧栏:资源管理器/编辑器/终端/git/浏览器 | 同上 |
| `dsh-skill-hub` | 技能中心:技能浏览/开关 + 技能市场一键更新 | 同上 |
| `dshmarket` | 可视化插件市场:页面内浏览、搜索、一键安装插件 | 同上(装了它,后面插件都可以在 GUI 里装) |

本地开发插件:package.json 依赖写 `"my-plugin": "link:/abs/path"`,bundles 加 `my-plugin`,`pnpm install` 后重启生效。

## 公网部署标准方案:Caddy + dsh-auth-gateway

### 为什么需要这两层

dsh 的 `/api` 有两层限制,公网/局域网直接访问会失败:

1. **服务端围栏**:`settings.*`、`credentials.*`、`agentPreset.*`、`llm.discoverModels` 等特权方法硬编码仅接受回环 Host,`--trusted-host` 也放不开;
2. **客户端判定**:浏览器地址栏不是 `localhost`/`127.x.x.x` 时,设置面板直接显示 "settings are unavailable in this browser",连请求都不发。

`dsh-auth-gateway` 同时解决这两层:网关转发时把 Host/Origin 改写为回环(过服务端围栏),并在页面注入脚本把浏览器连接标记为 loopback-trusted(过客户端判定)。它负责**认证**但不负责 **TLS**,所以公网标准形态是三层:

```
浏览器 ──https──> Caddy(:443, TLS 终结) ──> auth-gateway(回环, 登录门禁) ──Host 改写──> dsh webserver(回环)
```

### 第一步:安装 dsh-auth-gateway

```sh
dsh plugin --profile <p> add dsh-auth-gateway
```

网关默认占 `--port`(对外),真实 webserver 挪到 `端口+1`(仅回环)。因为流量全部经 Caddy 进入,在 `cordis.patch.yml` 把网关钉在回环:

```yaml
- id: dsh-auth-gateway
  inject: [webStartup]
  config:
    listenHost: 127.0.0.1
    listenPort: !!js ctx.webStartup.port ?? 28080
    upstreamHost: 127.0.0.1
    upstreamPort: !!js (ctx.webStartup.port ?? 28080) + 1
```

重启服务,首次启动会在日志(`journalctl -u <dsh服务>`)打印一次性初始密码;用它登录后引导页强制设置个人密码,可顺手绑定 TOTP。忘记密码:删 `$DSH_HOME/auth-gate/password.json` 重启,会重新打印初始密码(认证状态按 `$DSH_HOME` 存,跨 profile 共享)。

### 第二步:Caddy 做 HTTPS 反代

```caddyfile
{
	admin off
	auto_https off
}

# 有域名:用域名做 site 地址并让 Caddy 自动签发证书(去掉上面 auto_https off)
# 无域名:用 IP + 自签证书(浏览器需手动信任一次)
https://:443 {
	tls /path/to/cert.pem /path/to/key.pem
	reverse_proxy 127.0.0.1:28080
}
```

要点:

- 反代只需普通 `reverse_proxy` 指向网关端口,不需要任何 `header_up` 改写(网关自己会改写 Host/Origin);
- `admin off` 时 `systemctl reload caddy` 不可用(admin API 被关),改配置只能 `systemctl restart caddy`;
- 若用 systemd 常驻,建议 `After=network-online.target` 并让 dsh 服务与 Caddy 分开管理。

### 为什么保留 Caddy(而不是让网关直接对公网)

网关默认 `listenHost: 0.0.0.0`,技术上可以直接对外、省掉 Caddy。但网关只服务纯 HTTP(其会话 cookie 未加 `Secure` 标志,源码注明 MVP 阶段服务 HTTP),直接暴露意味着密码、会话 token、全部对话内容明文过公网,且浏览器将页面判为非安全上下文(部分 Web API 受限)。Caddy 常驻内存约 30MB,留它做 TLS 终结是标准分工。替代路线:有域名可换 `dsh-auth`(自带托管 Caddy 与自动证书),或 Cloudflare Tunnel(`dsh-auth-tunnel`)。

### 其他认证插件对比(都不完整解决设置面,记录备查)

| 插件 | 认证 | 设置面(公网) | 说明 |
|---|---|---|---|
| `dsh-auth-gateway` | 密码 + TOTP + 防爆破 | ✅ 两端都解锁 | 本手册推荐方案 |
| `dsh-auth` | 托管 Caddy 边缘 + forward_auth | ❌ | 运维最完善(e2e 测试),但未补客户端判定 |
| `dsh-auth-gate` | 进程内登录门 | ❌ | 文档明确"只加认证修不了设置页 403" |
| `@tyler9061/dsh-auth` | 登录 + 空闲登出 + LAN 反代 | ❌ | Host/Origin 改写只解决服务端 |
| `dsh-auth-tunnel` | Cloudflare Tunnel + 密码 | 视情况 | 适合已有 CF 的场景 |

## 常见坑

- **patch 整体替换 config**:只写一个字段会把其余字段(端口表达式等)全部清掉,导致端口回落默认值。
- **Caddy `admin off`**:`reload` 失败,只能 `restart`。
- **dsh 升级后**:回来验证 auth-gateway 的浏览器补丁是否仍兼容(不兼容时浏览器控制台显式报错,设置页退回 "unavailable"),以及各插件 peer 范围。
- **改密码会吊销全部会话**(所有浏览器需重新登录),内存会话在服务重启后也会清空 —— 设计行为。
- **公网不要裸奔 dsh**:agent 有工作区写权限,凭据文件里有模型 API key,必须有认证层。
