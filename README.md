# DeepSeek Harness (dsh) 插件手册

dsh Web 插件生态手册:插件清单与安装方法。

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
| `dsh-btw` | BTW 侧问面板:对话中随时弹出浮动面板,向全新独立会话提问(按消息锚定、保留各自历史,不占用主会话上下文) | `dsh plugin --profile <p> add github:LuckVd/dsh-btw` |
| `dsh-pin-color` | 会话置顶 + 标签颜色/emoji:侧栏会话置顶(组内或全局),每个会话设颜色与 emoji,host 持久化 | `dsh plugin --profile <p> add https://github.com/LuckVd/dsh-pin-color/releases/latest/download/dsh-pin-color.tgz` |
| `dsh-skill-hub` | 技能中心:技能浏览/开关 + 技能市场一键更新 | 同上 |
| `dshmarket` | 可视化插件市场:页面内浏览、搜索、一键安装插件 | 同上(装了它,后面插件都可以在 GUI 里装) |

> ⚠️ `dsh-btw` 的 npm 裸名已被他方占用(npm 上是别人的 0.0.3),**不要用裸包名 `dsh plugin add dsh-btw`**,一律用带 `github:` 前缀的地址指向本仓库。

本地开发插件:package.json 依赖写 `"my-plugin": "link:/abs/path"`,bundles 加 `my-plugin`,`pnpm install` 后重启生效。

## 常见坑

- **dsh 0.1.1-rc.x + `dsh-better-sidebar` ≤ 0.15.2 渲染报错** `cannot get property "betterSidebar" without inject`(浏览器控制台 `[dsh-better-sidebar] render error: …`,右侧栏空白/错误条)。根因:dsh 0.1.1 的浏览器端插件加载器下,插件 client 的 `ctx.provide("betterSidebar", …)` 注册到的 fiber 不在插件 apply ctx 的 parent 链上(实测 owner fiber ≠ apply ctx 的 fiber),插件自己渲染时读 `ctx.betterSidebar`(源码 27 处)被 cordis 的 inject 语义拦截。插件自己的 `inject` 也不能加 `betterSidebar` —— fiber 会等服务就绪才执行 apply,而该服务要等 apply 才 provide,死锁。上游 issue #135(dsh-web-mobile 场景,已关)确认过同一机制并做过部分修复(改为传 service 实例 + `ctx.get()`),但 web 端渲染路径仍留直读;**0.15.2 在 npm 版 dsh 0.1.1-rc.2 实测同样必现**(#321 报"侧边栏正常"系源码启动环境,不适用)。本地修复:改 profile 下 `node_modules/dsh-better-sidebar/lib/client.js`,`ctx.provide("betterSidebar", service)` 之后把 service 存进模块级变量,并将全部 `ctx.betterSidebar` 读访问替换为该变量的 getter(已验证报错消失、侧边栏正常);重装/升级插件会丢,需重打。已向上游提 [issue #356](https://github.com/omdsh-dev/DSH-better-sidebar/issues/356) + [PR #357](https://github.com/omdsh-dev/DSH-better-sidebar/pull/357)(内部读改 `ctx.get("betterSidebar")`,831 测试全绿 + 实测通过),合并发版后升级插件并移除本地补丁。
- **patch 整体替换 config**:只写一个字段会把其余字段全部清掉(端口、`!!js` 表达式等都会丢),导致端口回落默认值。
