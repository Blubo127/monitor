# monitor

本项目是 [Blubo127/monitor](https://github.com/Blubo127/monitor)，基于 [monitor-probe/monitor](https://github.com/monitor-probe/monitor) 维护的个人定制版本，提供轻量级服务器监控、管理后台和公开状态页。配套 Linux agent 在 [Blubo127/agent](https://github.com/Blubo127/agent)，定制开发分支为 `diy`。

安装与升级：[INSTALL.md](./INSTALL.md)。发布自己的二进制和镜像：[RELEASE.md](./RELEASE.md)。

定制版代码和文档位于 [`diy` 分支](https://github.com/Blubo127/monitor/tree/diy)。仓库首页展示默认分支的 README；要让首页显示本版本，需同步默认分支，具体见[分支与仓库首页](./RELEASE.md#分支与仓库首页)。

安装器、agent 下载和版本检查使用上述自有仓库的正式 Release，Docker 镜像使用 `ghcr.io/blubo127/monitor`。使用前请确认相应发布物已生成；尚未发布时，可先[从源码构建 hub](./INSTALL.md#从源码构建)。维护者首次发布的准备工作见[发布指南](./RELEASE.md#首次发布准备)。

## 特性

- 实时监控：秒级实时数据展示
- 轻量高效：Rust 语言构建，低资源占用，极简高效
- 自托管：完全掌控数据隐私，部署简单
- 通知：节点掉线、流量、到期与登录，推送到 Telegram 或自定义 Webhook

## 组成

| 仓库 | 说明 |
|---|---|
| [Blubo127/monitor](https://github.com/Blubo127/monitor) | 自有 hub：后台、API、公开页宿主；安装、更新和镜像从此仓库发布 |
| [Blubo127/agent](https://github.com/Blubo127/agent) | 自有 Linux agent；hub 从此仓库的 Release 中继 agent 二进制 |
| [Blubo127/Theme-Glassmorphism](https://github.com/Blubo127/Theme-Glassmorphism) | 自有可选公开页主题；需在主题仓库先发布 `theme.tar.gz` 才能从面板安装 |
| [monitor-probe/monitor-theme-default](https://github.com/monitor-probe/monitor-theme-default) | 上游默认主题；构建时仍按 `web-theme.pin` 的版本和 SHA-256 下载并校验 |

```text
agent (Linux)  ──WebSocket / JSON-RPC 2.0──▶  hub (axum + SQLite)  ──▶  后台 + 状态页
```

可选的 Glassmorphism 主题通过后台安装和启用；本仓库内置的仍是上游默认主题。安装方式见 [INSTALL.md](./INSTALL.md#主题)。

## 上游参考

上游项目为 [monitor-probe/monitor](https://github.com/monitor-probe/monitor)。通用功能、反向代理和主题开发可参考上游文档站 [monitor-document.pages.dev](https://monitor-document.pages.dev)；第三方主题列表可参考上游主题站 [monitor-themes.pages.dev](https://monitor-themes.pages.dev)（源码：[monitor-probe/themes](https://github.com/monitor-probe/themes)）。这些站点归上游维护，文档中的安装、升级和镜像地址可能指向上游仓库；部署本 fork 时请使用本仓库的 [INSTALL.md](./INSTALL.md) 和 [RELEASE.md](./RELEASE.md)。

历史检查记录见 [check_summary.md](./check_summary.md)，其中的审计结论和代码行号仅对应报告列出的受检提交。

## 许可证与致谢

本项目采用 [MIT License](./LICENSE)，保留原作者 `stqfdyr` 的版权声明。感谢上游 [monitor-probe/monitor](https://github.com/monitor-probe/monitor) 及其配套项目；本 fork 的定制代码和发布物在 `Blubo127` 的仓库维护。
