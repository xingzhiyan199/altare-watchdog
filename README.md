# OpenList Watchdog

Altare 上的 OpenList（Docker App）外部看门狗。

## 为什么需要它

Altare 的 Docker App 自己带进程级自愈：容器主进程退出（崩溃、退出码非 0、被 OOM kill）后，平台会在 **1~2 秒**内自动重新拉起，这个不需要额外配置。

平台自愈覆盖不了的是「平台主动停机」：节点维护、资源超限、额度耗尽、误操作 stop。这种情况下服务会一直躺着，直到有人手动 start。

这个仓库就是补这一刀。

## 它做什么

每 5 分钟：

1. 请求 `https://openlist.go.altare.global/ping`，期望返回 `200` + `pong`
2. 失败重试 3 次（间隔 10 秒），排除单次网络抖动
3. 仍失败 → 调 Altare API `POST /power {"signal":"start"}`
4. 每 10 秒轮询，最多 150 秒
5. 还没起来 → 再发一次 `restart`，再等 90 秒
6. 依然失败 → 任务标红，GitHub 会给你发失败邮件

## 部署

### 1. 建仓库

在 GitHub 上建一个仓库（public 私有都行，public 不消耗 Actions 额度），把本目录推上去。

### 2. 配 Secret

仓库 `Settings → Secrets and variables → Actions → New repository secret`

- Name: `ALTARE_TOKEN`
- Value: Altare 的 API token

### 3. 启用 Actions

`Actions` 标签页 → 如果提示 workflow 被禁用，点 `I understand my workflows, go ahead and enable them`。

### 4. 手动跑一次验证

`Actions → OpenList Watchdog → Run workflow`。

看日志里探活是否 `http=200 body=pong`。

## 已知限制

- **GitHub cron 不准点**。`*/5` 只是「大约每 5 分钟」，负载高时会延迟 3~15 分钟，极端情况会跳过。所以最坏情况下从挂掉到被拉起可能隔十几分钟。要更快只能上自建 VPS 的 cron 或 Cloudflare Worker。
- **60 天失活**。仓库连续 60 天无提交，GitHub 会自动停用 scheduled workflow。`keepalive.yml` 每月 1 号自动提交一次时间戳来绕开这个限制。
- **只看 HTTP 可达性**。`/ping` 返回 pong 说明 OpenList 进程活着、路由通。它不检查 115 网盘挂载是否失效——网盘 token 过期时 OpenList 依然会回 pong。

## 相关配置

| 项 | 值 |
|---|---|
| 健康检查 | `https://openlist.go.altare.global/ping` |
| API 入口 | `https://console.altare.gg/api` |
| Tenant | `78b0fb5e-e781-48d5-9664-1c365b5bb7f7` |
| Server | `79f32fd0-3af2-4174-bc75-8fc88322a6e3` |
| 节点 | se-sto02（Stockholm） |
