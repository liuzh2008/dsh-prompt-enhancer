# remote-web-ui 版本错配导致 100.66.1.3:3081 "Failed to load plugins" + 历史加载 HTTP 405 — 问题修复记录

- **日期**：2026-09-08
- **影响范围**：DeepSeek Harness Web GUI（dev 实例，端口 3081，`dsh web --port 3081 --trusted-host 100.66.1.3 --allow-remote-privileged-methods`）
- **状态**：✅ 已修复（用户确认）
- **关联文档**：[DSH升级后100.66.1.3无历史对话及randomUUID报错.md](./DSH升级后100.66.1.3无历史对话及randomUUID报错.md)（2026-08-25，同源问题的前传）

---

## 一、背景

Web profile 装配了全家桶聚合包 `@linxin666/dsh-web-all@0.3.17`（内含
`dsh-remote-web-ui` 0.3.17 等 20+ 家族插件）。该包 `dsh.engines.dsh` 要求
**官方 `>=0.1.2-rc.1`**，其 remote-web-ui 按官方 **0.1.2-alpha.2** 的 wire
契约编写；而本机运行的官方 dev checkout（`upgrade-rc12` 分支）是
**0.1.1-rc.2** —— 两者契约错配，是非回环源（Tailscale/LAN IP）访问
崩溃与 405 的根源。

服务启动参数（用户既定配置）走的是**官方 trusted-host 直连**模型：
`--trusted-host 100.66.1.3 --allow-remote-privileged-methods`（webserver
patch 固定 `host: 0.0.0.0`）。remote-web-ui 的配对 `/remote` 通道模型与此
部署意图冲突且版本不兼容。

## 二、问题现象

1. 访问 `http://100.66.1.3:3081/`（本机 Tailscale 网卡 IP，非回环）时：
   **`Failed to load plugins`**，错误详情
   `failed to apply loader entry 73deabbf (@deepseek-ai/dsh-client-connection):
   transport?.createApiClient is not a function`。
   `http://127.0.0.1:3081/` 一直正常。
2. 修复上述崩溃后，页面能打开，但**对话历史加载失败**：
   `transport failure for /api/session.history: HTTP 405（internal）`。
   症状诡异：**每次需要 Ctrl+F5 才能看到列表**；普通 F5、重启浏览器都
   复现 405。

## 三、根因分析

### 3.1 现象一：remote-web-ui 注入 `__DSH_TRANSPORT__`，官方旧版 client-connection 崩溃

- remote-web-ui host 半通过 `webserver/index-inject` 向每个 index 页面注入
  `REMOTE_CHANNEL_BOOT_SCRIPT`（`src/remote-channel-boot.ts`）。
- 该脚本对**非回环源**（`localhost`/`::1`/`127.*` 之外，即 100.66.1.3）执行：
  1. `window.__DSH_TRANSPORT__ = { ownsHost: true }`；
  2. 改写 fetch/WebSocket/EventSource/src，把 `/api/*`、`/sidebar/*` 等
     请求改写为 `/remote/...` 配对通道。
- 官方 0.1.1-rc.2 的浏览器半（`packages/client/connection/src/client/index.ts`）
  仍把 `__DSH_TRANSPORT__` 当作**必须带 `createApiClient` 的完整载体**：

  ```ts
  const transport = (globalThis as ClientTransportGlobal).__DSH_TRANSPORT__
  const api = fixtureClient ?? transport?.createApiClient() ?? new WebApiClient()
  ```

  而注入的对象只有 `ownsHost: true`，无 `createApiClient` →
  `transport?.createApiClient()` 抛 TypeError → client-connection 装载失败 →
  "Failed to load plugins"。
- 官方 0.1.2-alpha.2 已改为**可选修饰**语义（npm 包源码比对确认）：
  `isLoopback: transport?.ownsHost === true || ...`，transport 只提供可选的
  `fetch`/`openStream`，不再要求 `createApiClient`。remote-web-ui 正是为此
  新版编写的。

### 3.2 现象二（405 死锁）：禁用插件反而加剧——浏览器半 fail-closed 安装 /remote 改写

remote-web-ui 的**浏览器半代码被打包进 web-ui-compat 的聚合 client bundle**
（`@linxin666/dsh-web-all/lib/client.js`，`mountClientChildren` 无条件挂载
家族 client），与 host 半装配状态无关、始终在浏览器运行。

尝试 `disabled: true` / `enabled: false` 的后果完全相同：

1. host 半不再注册配对路由 → `/api/pair/status` **404**；
2. 浏览器半启动时调用 `readPairGatePolicy()` 探测配对策略 → 404 →
   **fail-closed 判定"需要配对"** → 安装 `/remote` 改写（fetch/WS 劫持）；
3. 改写把 `/api/settings.describe` 也劫持到 `/remote` → settings 永远拉不到
   → 浏览器半永远读不到关闭态 → **改写永不撤销** → 所有 `/api/*`
   （含 session.history）→ `/remote` → 该前缀路由已不存在 → 落到底层静态
   fallback → 非 GET/HEAD 一律 **405**；
4. Ctrl+F5 偶尔能过 = 全新加载时 settings 恰好先于改写安装就绪的时序侥幸；
   普通 F5 复用旧页面已改写的 fetch 状态 → 必现 405。

### 3.3 版本对照（确认错配）

| 契约点 | 官方 0.1.1-rc.2（本机） | 官方 0.1.2-alpha.2+（remote-web-ui 依赖） |
|---|---|---|
| `__DSH_TRANSPORT__` 语义 | 必带 `createApiClient` 的完整载体 | 可选修饰：`ownsHost`/`fetch`/`openStream` |
| `connection.isLoopback` | 仅按 hostname 判定 | 额外认 `transport?.ownsHost === true`（host 模式翻转） |
| remote-web-ui engines | — | `dsh >= 0.1.2-rc.1` |

## 四、修复方案（最终）

在 profile 的 `cordis.patch.yml` 中为 `web-ui-remote-web-ui` 覆盖配置——
**保持插件 enabled、仅关闭 LAN 配对门**：

```yaml
- id: web-ui-remote-web-ui
  config:
    plugin: '@linxin666/dsh-remote-web-ui'
    config:
      enabled: true
      requirePairingForLan: false
```

为什么这是正解（而非 disabled / enabled:false）：

- host 半保持装配 → `/api/pair/status` 存活并返回
  `{"requirePairingForLan": false}`；
- 浏览器半配对策略探测成功（不再 fail-closed）→ 判定"非回环桌面无需配对"
  → **不安装 /remote 改写**（settings 就绪分支 `enabled(true) &&
  requirePairingForLan(false) = false`；未就绪分支 host 策略回报 false，
  两条路径都不劫持）；
- index 不再注入 `__DSH_TRANSPORT__`（boot 注入条件
  `enabled && requirePairingForLan` 不成立）→ client-connection 正常装载；
- `/api` 全部直连官方 trusted-host（100.66.1.3 已在信任名单 +
  `--allow-remote-privileged-methods`）→ 完整可用。

> remote-web-ui README 将此配置明确列为支持场景：
> "Set false to keep the desktop on plain /api (only useful when that origin
> is already trusted for /api)" —— 与本部署的官方 trusted-host 直连模型一致。

## 五、验证

| 检查项 | 结果 |
|---|---|
| `/api/pair/status`（127.0.0.1） | **200** `{"ok":true,"paired":false,"requirePairingForLan":false,"phase":"stopped",...}` |
| index 注入 `__DSH_TRANSPORT__` / remote-channel 脚本 | 已消失（-1） |
| `settings.describe` → `remote-web-ui` | `enabled: true, requirePairingForLan: false` |
| `POST /api/session.list`（100.66.1.3） | 200，正常返回会话列表 |
| `POST /api/session.history`（原报错端点） | 200 |
| 浏览器全新打开 `http://100.66.1.3:3081` | ✅ 历史列表正常显示（用户确认） |

修复为 patch 热重载生效（`watchUserPatches` 监视 cordis.patch.yml），
无需重启进程；最终用户**彻底关闭旧标签页后全新打开**即恢复（旧页面的
fetch 改写状态不会因 F5 重置）。

## 六、经验教训

1. **版本错配先行判断**：装配第三方全家桶前核对其 `dsh.engines` 与本地
   官方线；`dsh-web-all 0.3.17` 要求 `>=0.1.2-rc.1`，本机 0.1.1-rc.2 不满足。
   根治方向是把官方 checkout 升到支持 `ownsHost` 契约的版本，届时可删除
   本 patch 恢复配对功能。
2. **聚合包的浏览器半 ≠ host 半**：`dsh-web-all` 把家族 client 代码打进
   `web-ui-compat` 聚合 bundle，`disabled` host entry 并不能停掉浏览器半；
   其配对策略探测失败会 **fail-closed**（宁错杀不放过），在非回环源必然
   劫持 `/api`。
3. **普通 F5 vs Ctrl+F5 症状差异的指向**：页面 JS 已安装的 fetch 改写不会
   因刷新重置；"必须硬刷/清站点数据才好"通常是**旧页面状态残留**而非
   服务端未生效。
4. **配置关闭优先于装配禁用**：对双面插件，先尝试插件的官方配置开关
   （`enabled`/`requirePairingForLan`），保留 host 半存活以维持其策略端点，
   不要轻易 `disabled`/卸载 entry。

## 七、相关路径

- Profile patch：`C:\Users\Administrator\.dsh-dev\profiles\web\cordis.patch.yml`
- 中间态备份：`C:\Users\Administrator\.dsh-dev\profiles\web\cordis.patch.yml.bak-disable-remote-web-ui-20260908`（可删）
- 官方 checkout：`C:\Users\Administrator\Documents\Qoder\2026-08-13\chat-1\deepseek-harness-dev`（`upgrade-rc12`，0.1.1-rc.2）
- 全家桶包：`C:\Users\Administrator\.dsh-dev\profiles\web\node_modules\@linxin666\dsh-web-all`（0.3.17，含 `dsh-remote-web-ui` 0.3.17）
- 官方新版契约参考：npm `@deepseek-ai/dsh-client-connection@0.1.2-alpha.2`
  （`lib/client.js`：`isLoopback: transport?.ownsHost === true || ...`）
