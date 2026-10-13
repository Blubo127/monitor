# 发布自己的 hub 和 agent

本 fork 的 hub 发布目标是 [Blubo127/monitor](https://github.com/Blubo127/monitor)，配套 agent 独立发布到 [Blubo127/agent](https://github.com/Blubo127/agent)。安装方式见 [INSTALL.md](./INSTALL.md)。本指南以本仓库的 [.github/workflows/release.yml](./.github/workflows/release.yml) 为准；agent 操作需在其独立仓库中执行。

## 首次发布准备

首次发布前，分别检查两个仓库的 Actions 是否启用、用于发布的提交是否包含自有仓库地址，以及工作流是否有写入 Release 的 `contents: write` 权限。hub 的发布 job 还需要 `packages: write` 权限推送 GHCR。Docker Hub 凭据是可选项；不配置时，hub 工作流仅推送 GHCR。

2026-10-12 通过 GitHub API 核查的发布准备状态如下，后续以仓库页面为准：

- [hub Releases](https://github.com/Blubo127/monitor/releases)、[agent Releases](https://github.com/Blubo127/agent/releases) 和[可选主题 Releases](https://github.com/Blubo127/Theme-Glassmorphism/releases) 均为空；首次发布完成前，对应的 `releases/latest/download/...` 无法提供安装包。
- hub 和 agent 的 GitHub 默认分支均为 `main`。本 fork 的定制开发使用 `diy`；创建 tag 时需要明确选择包含定制改动的提交。
- [hub Actions](https://github.com/Blubo127/monitor/actions) 已列出 `ci`、`release` 两个 active workflow；agent 的 Actions API 未列出 workflow，应到 [agent Actions](https://github.com/Blubo127/agent/actions) 确认是否需要启用 fork 工作流。API 列表为空不等于仓库没有工作流文件。

`ghcr.io/blubo127/monitor` 是发布目标。实际可拉取的镜像标签和权限应在发布工作流成功后核对。

## 版本和触发方式

两个仓库的 release workflow 都监听 `v*` tag 的 push，并提供 `workflow_dispatch`。推送版本 tag 才是正式发布流程；在 `main` 或 `diy` 分支手动运行 `workflow_dispatch` 只构建和收集 Actions artifacts，发布步骤受 `refs/tags/` 条件限制，不会创建 Release 或推送镜像。

版本 tag 必须与该 tag 所指提交的 `Cargo.toml` 中 `[package].version` 完全一致，格式为前缀 `v` 加版本号。工作流在构建时检查两者，不一致会直接失败。例如：

| 仓库 | 示例 package 版本 | 对应 tag |
|---|---|---|
| `Blubo127/monitor` | `1.4.0` | `v1.4.0` |
| `Blubo127/agent` | `1.2.0` | `v1.2.0` |

表中的版本仅用于说明格式，实际版本以各自发布提交的 `Cargo.toml` 为准，两个仓库的版本号无需相同。发布自己的改动时，先决定自有版本号并同步 `Cargo.toml` 和 `Cargo.lock`，再将 tag 指向已经审查、检查通过的提交。

本仓库的 CI push 触发分支为 `main` 和 `diy`，pull request 也会触发 CI；仅修改 Markdown 或 `LICENSE` 时会按 `paths-ignore` 跳过。agent 的触发分支和集成测试下载地址以其独立仓库的 `.github/workflows/ci.yml` 为准，发布前确认已指向 `Blubo127/monitor`。GitHub 的手动运行入口依赖默认分支上的 workflow，选择 `diy` 运行时也应确认实际 ref。

## 首次发布前的集成检查

两个仓库的 CI 会将当前源码构建产物与另一个仓库的最新 hub/agent Release 配对测试。如果配套仓库尚无自己的 Release，集成测试的下载步骤会返回 404；首次发布前先用两个源码构建产物配对检查，其余格式、静态检查和单元测试仍应通过。

按[源码构建](./INSTALL.md#从源码构建)准备 hub 的构建环境、前端和默认主题，另外安装 `jq`、ShellCheck 以及 Rust 的 rustfmt、Clippy 组件。将配套 agent 的定制分支检出到 `../agent` 后，从 hub 仓库根目录执行：

```bash
(cd web-admin && npm run lint && npm test && npm run build)
shellcheck -S warning install.sh install-hub.sh scripts/*.sh
cargo fmt --all --check
cargo clippy --locked --all-targets -- -D warnings
cargo test --locked
cargo build --locked
(cd ../agent && cargo fmt --all --check && cargo clippy --locked --all-targets -- -D warnings && cargo test --locked && cargo build --locked)
sh scripts/e2e.sh target/debug/monitor-hub ../agent/target/debug/monitor-agent
```

在两个仓库分别手动运行 release workflow 还能预检双架构 release 构建；它本身不依赖 CI job。源码配对检查和构建预检完成后，按下面的顺序创建首批 Release，双方资产齐全后再重新运行完整 CI。这些是发布前需要执行的步骤，本文件不表示它们已经完成。

## 推荐发布顺序

1. 在 agent 仓库确认 fork Actions 已启用，检查自有下载/版本配置；首次发布先完成上一节的源码配对检查，后续发布运行完整 CI。在开发分支手动执行 release workflow 预检构建。修改版本后，在目标提交创建并推送匹配的 `v*` tag。
2. 等待 agent release workflow 成功，确认 Release 不是 draft/prerelease，并检查两个架构二进制及 `sha256sums.txt` 都已上传。hub 的 agent 下载和 CI 使用 `Blubo127/agent` 的 latest Release，首次 hub 发布前先有 agent 发布版。
3. 在 hub 仓库运行 CI，确认二进制构建使用 `web-theme.pin` 固定的上游默认主题，安装器和更新检查指向自有仓库。可在 `diy` 或 `main` 手动运行 release workflow 预检。
4. 在 hub 的目标提交创建并推送与 package 版本匹配的 `v*` tag，等待两个架构构建、GitHub Release 和 GHCR 推送全部完成。
5. 从自己的 Release 下载二进制与校验文件，在隔离环境检查安装、升级和 agent 连接，再按 [INSTALL.md](./INSTALL.md) 部署。

本仓库的发布工作流不会自动发布 agent，也不会自动发布可选的 Glassmorphism 主题。`Blubo127/Theme-Glassmorphism` 需要在独立主题仓库发布 `theme.tar.gz` 后才能在面板中使用其仓库地址安装。上游默认主题、文档站和主题站不属于本 fork 的发布物。

## Release 资产与下载地址

hub 工作流通过 `cross build --release --locked` 分别构建 x86_64 与 aarch64 的 musl 二进制，随后创建 GitHub Release 并上传：

| 资产名 | 用途 |
|---|---|
| `monitor-hub-x86_64-unknown-linux-musl` | x86_64 / amd64 Linux hub |
| `monitor-hub-aarch64-unknown-linux-musl` | aarch64 / arm64 Linux hub |
| `install-hub.sh` | 与该发布版对应的安装和升级脚本 |
| `sha256sums.txt` | 两个 hub 二进制和安装脚本的 SHA-256 |

发布成功后的 latest 下载前缀是：

```text
https://github.com/Blubo127/monitor/releases/latest/download/
```

例如安装脚本地址为 `https://github.com/Blubo127/monitor/releases/latest/download/install-hub.sh`。固定版本的资产下载地址使用 `download/v<版本>/`，并从同一发布版下载二进制和校验文件。安装器自身始终安装 `latest` 且可能自更新；严格固定版本应手动部署已核对的二进制。

agent 工作流同样构建两个 Linux musl 架构，创建 Release 并上传：

| 资产名 | 用途 |
|---|---|
| `monitor-agent-x86_64-unknown-linux-musl` | x86_64 / amd64 Linux agent |
| `monitor-agent-aarch64-unknown-linux-musl` | aarch64 / arm64 Linux agent |
| `sha256sums.txt` | 两个 agent 二进制的 SHA-256 |

发布成功后的 latest 下载前缀是：

```text
https://github.com/Blubo127/agent/releases/latest/download/
```

hub 的 `/agent/x86_64` 和 `/agent/aarch64` 会从该仓库下载并中继对应文件。保持上述资产名，否则面板提供的 agent 安装和升级命令会下载失败。版本查询使用这两个仓库各自的 `releases/latest` API；没有公开的正式 Release 时，面板会显示查不到发布版本。

## hub 镜像

hub release job 以 `GITHUB_REPOSITORY` 的小写值生成 GHCR 路径。该 fork 发布时的镜像名是：

```text
ghcr.io/blubo127/monitor:<版本>
ghcr.io/blubo127/monitor:latest
```

镜像同时包含 `linux/amd64` 和 `linux/arm64`，使用同一次工作流构建出的 hub 二进制。首次推送后检查 GitHub Packages 中包的可见性及访问权限：若镜像保持 private，部署机需要具备拉取权限；若要公开拉取，应把包设置为 public 后验证。

若配置可选的 `DOCKERHUB_USERNAME`、`DOCKERHUB_TOKEN`，同一工作流也会推送 `docker.io/<配置的用户名>/monitor:<版本>` 和 `latest`。这里没有预设 Docker Hub 用户名，也没有声称该地址已发布。

## 发布后核对

- 打开两个自有仓库的 Releases 页面，确认最新正式 Release 及完整资产；不要只看 Actions 的 build job 成功。
- 下载每个 Release 的 `sha256sums.txt` 并对对应文件校验。校验和与文件同一来源，能发现传输损坏，但仍需信任发布仓库和构建流程。
- 查看 hub release job，确认 GHCR 多架构镜像推送成功；检查镜像标签、权限和每个架构能否拉取。
- 在隔离环境验证 hub 安装、回滚、agent 下载/连接及版本页面。首次 agent Release 缺失、资产名变更或镜像未公开都会影响实际部署。
