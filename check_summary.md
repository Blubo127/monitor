# monitor 仓库安全检查摘要

检查日期：2026-10-09  
检查对象：当前工作区 `/home/nlpz/wp/proj/moni/monitor`  
分支：`diy`  
提交：`d8d8283ee9a637b3896f7e89808f8610a4462076`  
远程：`git@github.com:Blubo127/monitor.git`  
审计开始时：除本报告外无代码改动；当前新增未跟踪文件为 `check_summary.md`

## 结论

在当前提交的 Rust、Shell、前端、Docker、CI 和锁定依赖配置中，没有发现“后台定时把数据库或服务器文件上传到固定陌生地址”的明显后门，也没有发现运行时执行任意 Shell 命令的代码。源码中的外联大多对应明确功能。

这不等于可以无条件信任。当前版本有几个实际的泄密和供应链风险：

1. 每个节点的公网 IP 会发送给 `ipinfo.io` 做国家查询。
2. 配置通知后，节点名称、登录 IP、流量/掉线/到期信息会发送到 Telegram 或管理员填写的 Webhook。
3. 主题包是可执行的浏览器 JavaScript；不可信主题在管理员已登录并访问公开页时，可能借同源浏览器会话读取或操作面板数据，并将结果发往外部。
4. agent 二进制由 hub 从 GitHub 或配置的 HTTPS 代理中继，安装脚本只检查 ELF 文件头，没有独立的哈希或签名校验。代理、发布物或 fork 被替换时，会把不可信程序安装到节点服务中。
5. hub 手动运行和 Docker 默认监听通配地址。若直接暴露端口，面板会以明文 HTTP 传输会话和 agent token；`--insecure` 还允许通过明文链路下载 agent。

因此，按默认官方安装器、只使用默认主题、全程 HTTPS、关闭不需要的通知和 OAuth 时，未见明显的隐藏外传后门；按当前仓库直接编译、安装第三方主题、使用 HTTP/任意代理或把端口直接暴露公网时，风险会明显升高。不能把这个静态检查当作对二进制、依赖漏洞或远端发布物的保证。

## 发现明细

| 编号 | 风险 | 证据 | 影响与判断 |
|---|---|---|---|
| F-01 | 中 | `src/agent_ws.rs:524-593` | 节点连接时选择公网接口地址，并最多每小时请求 `https://ipinfo.io/{公网IP}/country`。发送的是节点公网 IP，属于明确的第三方数据外传；代码只保存两位国家代码。 |
| F-02 | 中（可选功能） | `src/notify.rs:252-323,424-429` | 只有管理员配置渠道后才发送。Telegram 消息或 Webhook 默认包含事件、节点名、标题、消息；登录提醒包含登录方式和客户端 IP。Webhook URL、请求头和模板都存于设置中，URL 允许 `http://`，因此可能泄露凭据或告警内容。 |
| F-03 | 高（安装不可信主题时） | `src/api.rs:1900-2023,2223-2255`；`src/frontend.rs:104-110,236-247` | 管理员可上传或从 GitHub 安装主题，主题的 `dist/index.html` 和 JS 会直接以 hub 同源页面提供，没有脚本沙箱。HttpOnly cookie 不能被 JS 读取，但同源 `fetch` 会自动携带登录 cookie；管理员访问公开页时，应把主题视为可执行的同源扩展。写设置接口只要求 `Admin` session，未见单独的 Origin/CSRF 校验，因此恶意主题可能读取或修改设置，并把结果外传。 |
| F-04 | 高（供应链） | `src/main.rs:269-310`；`install.sh:234-343` | hub 从 `monitor-probe/agent` 的 GitHub release 或 `github_proxy` 中继 agent 字节；源码明确写明“relayed unverified”，安装脚本只检查 ELF 头。脚本以 root 写入文件并配置服务，运行服务本身使用 `monitor-agent` 用户。必须独立核对 agent 发布物并审计 agent 仓库。 |
| F-05 | 高（部署错误时） | `src/main.rs:328-341,443-464`；`Dockerfile:9-31`；`install.sh:138-200` | hub 默认监听 `[::]:28080` 或 `0.0.0.0:28080`，仅发出告警，不会阻止明文监听。直接发布 Docker 端口或手动启动时，session cookie、面板请求和 agent token 可能被窃听；`--insecure` 明确允许 HTTP 下载和 WebSocket。 |
| F-06 | 中（凭据/数据暴露） | `src/api.rs:1547-1670`；`src/db.rs:691-713,969-978` | 管理员可以下载整库备份。数据库包含明文 node token、GitHub client secret、密码哈希、会话等敏感数据；程序会尽力设置 `0600`，但备份文件和传输链路仍需按密钥材料保护。 |
| F-07 | 低至中（预期外联） | `src/auth.rs:208-216,334-398`；`src/api.rs:1808-1820,1970-2006` | GitHub OAuth 会向 GitHub 发送 client ID/secret、授权 code，并用 access token 请求 `/user`；版本检查和主题更新会请求 GitHub API/release。它们不是隐藏通道，但会让 hub 与 GitHub 通信并把 OAuth 凭据交给 GitHub。 |
| F-08 | 中（构建供应链） | `build.rs:7-20`；`scripts/theme.sh:19-49`；`.github/workflows/*.yml` | `cargo build` 会执行仓库脚本并下载默认主题；主题包按 `web-theme.pin` 的 SHA-256 校验。锁文件使用 crates.io/npm registry 和完整性哈希，未见自定义 `postinstall`。Docker 的 `alpine:3`、CI action 的 major/stable 标签仍是可变供应链输入。 |
| F-09 | 中（来源可信度） | `Cargo.toml:7`；Git remote；`git log --show-signature` | 当前 remote 是 `Blubo127/monitor`，而 Cargo、agent、主题和发布地址主要指向 `monitor-probe/*`。当前 `diy` 与本地 `main/origin/main` 同为上述提交，但 fork 与上游/发布物的关系需要自行确认。提交显示 GPG key `B5690EEEBB952194`，本机没有公钥，签名尚未验证。 |
| F-10 | 信息泄露面（产品行为） | `src/api.rs:107-130,324-357` | 默认 public page 开启；标记为 public 的节点会向匿名访问者展示系统/内核/架构、CPU/内存/磁盘、流量、国家、在线状态等指标。这不是隐藏上传，但会暴露服务器指纹和运行状态；可关闭 public page 或取消节点的 public 标记。 |

## 当前代码中的外联清单

| 目标 | 触发条件 | 发送内容 |
|---|---|---|
| `ipinfo.io` | 节点 hello、地址变化或每小时重试 | 节点公网 IP（URL 路径） |
| `github.com` / `api.github.com` | agent 下载、版本检查、主题安装/更新、GitHub 登录 | release/repo 路径；OAuth 流程还包括 client secret、授权 code、access token 请求 |
| `api.telegram.org` | 配置 Telegram 通知后产生告警 | bot token 在 URL 中；chat ID、事件标题和消息在 JSON 中 |
| 管理员设置的 Webhook | 配置 Webhook 后产生告警或测试通知 | 模板决定的事件、节点名、登录 IP、告警消息和站点地址；自定义请求头也会发送 |

没有发现除上述目标之外的固定外联域名，也没有发现读取 `/etc/shadow`、SSH 私钥、Docker socket 等敏感主机文件的代码。源码中真正的外部命令调用主要是 `build.rs` 调用 `sh scripts/theme.sh`，不是运行 hub 时的隐藏命令执行。

## 建议的安全使用方式

1. **先确认来源。** 不要只看当前 fork 的分支名；将提交与可信的 `monitor-probe/monitor` 上游逐项比较，并从可信渠道取得 GPG 公钥后再验证签名。生产环境固定到已审计的提交，不直接跟随 `latest`。
2. **不要在生产服务器直接编译陌生 checkout。** `cargo build` 会执行 `build.rs` 和下载主题脚本；在隔离构建机生成并核对产物，再部署到生产机。
3. **只使用 HTTPS。** hub 放在 loopback（例如 `--listen 127.0.0.1:28080`），由反向代理终止 TLS；Docker 发布端口时绑定宿主 `127.0.0.1`。不要使用 `--insecure`，也不要把 28080 直接映射到公网。
4. **审计 agent 供应链。** 下载 agent 后，除 ELF 头外再用独立渠道核对版本、SHA-256 或签名；谨慎设置 `github_proxy`，因为它决定了 agent 和主题包的下载来源。
5. **不要安装未经审计的主题。** 主题包可以包含任意浏览器 JS。优先使用仓库固定的默认主题；如必须使用第三方主题，先在隔离域名/浏览器审查构建后的 JS，并避免在管理员登录状态下访问其公开页。
6. **收紧数据外露。** 不需要公开状态页时关闭 `public_page`；逐个关闭节点的 public 标记。若不需要 Telegram/Webhook/GitHub OAuth，则不要配置它们；Webhook 尽量使用 HTTPS，并检查模板和请求头。
7. **保护数据库和备份。** 将 `monitor.db`、WAL/SHM 和导出的备份视为包含凭据的密钥材料，限制文件权限和下载权限；备份传输完成后及时删除临时副本。
8. **做一次上线前出站监控。** 在隔离环境运行 hub，用 DNS/连接跟踪或抓包确认出站只落在预期目标；若策略要求零外联，需要阻断 `ipinfo.io`、GitHub 和通知目标，或修改代码移除相应功能，并接受相关功能失效。

## 检查边界

本次检查以当前提交的源码和配置为对象，采用静态阅读、全文搜索、Git 历史/锁文件/安装脚本检查；没有运行仓库中的未知二进制、构建脚本或安装脚本，也没有对 crates.io/npm 包逐个审计源码、执行完整 CVE 扫描或验证远端 release 的签名。因此结论是“当前源码未见明显隐藏后门，存在已列出的明确外联和供应链风险”，不是对所有未来更新、远端二进制或依赖漏洞的安全保证。
