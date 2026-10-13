# 安装与使用

本分支的 hub 安装与更新来自 [Blubo127/monitor](https://github.com/Blubo127/monitor/releases)，agent 来自 [Blubo127/agent](https://github.com/Blubo127/agent/releases)。向 GitHub 推送分支代码不会自动生成可下载的安装包；使用下面的安装命令前，需要按[发布指南](./RELEASE.md) 发布正式 Release 并上传对应资产。没有正式 Release 或缺少资产时，下载会返回 404；hub 也可以先[从源码构建](#从源码构建)。

本说明面向 `diy` 定制分支。面板和安装器的帮助链接指向 `diy` 下的本文件；将自定义版本迁到其他分支时，也应同步修改这些帮助链接。

## 安装 hub

本仓库安装器支持 Linux、systemd、x86_64 和 aarch64，需要 `curl` 和 `sha256sum`，安装步骤需要 root 权限。它会创建 `monitor` 服务用户，hub 服务以该用户运行。

首次发布完成后，下载并校验本项目的安装器，再执行：

```bash
curl -fsSL https://github.com/Blubo127/monitor/releases/latest/download/install-hub.sh -o install-hub.sh
curl -fsSL https://github.com/Blubo127/monitor/releases/latest/download/sha256sums.txt -o sha256sums.txt
sha256sum --ignore-missing --check sha256sums.txt
```

确认输出中 `install-hub.sh` 校验成功后再执行：

```bash
sudo sh ./install-hub.sh
```

有终端时，安装器会显示菜单，选择“安装 / 升级”即可。安装器还会从同一仓库下载和校验 hub 二进制，写入 `/opt/monitor/monitor-hub`，数据和主题保存在 `/opt/monitor/data/`。`latest` 表示最新正式 Release。将下载 URL 的 `releases/latest/download` 换成 `releases/download/v<版本号>`，只能固定下载得到的资产；安装器运行时会检查最新安装器并自动更新，其二进制下载逻辑也始终选择 `latest`。严格固定版本时，应从目标 tag 下载二进制与校验文件，校验后手动部署，不能依靠安装器固定版本。

常用命令：

```bash
sudo sh ./install-hub.sh --port 28080
sudo sh ./install-hub.sh --site https://hub.example.com
sudo sh ./install-hub.sh --reset-password
sudo systemctl status monitor-hub
sudo journalctl -u monitor-hub -f
```

把 `hub.example.com` 换成自己实际部署的域名。安装器默认只监听 `127.0.0.1:28080`；面板通过 HTTPS 反向代理访问，路径为 `/admin`。首次安装会显示应急密码。通常不必填写 `--site`；通过 SSH 隧道访问面板、节点使用另一域名、或代理不传 `X-Forwarded-Proto` 时，用它指定节点可达的 HTTPS 域名。

## 反向代理

先为域名配置 DNS 和 TLS。以下 nginx 示例中的证书路径需替换为本机实际证书；`map` 放在 `http` 上下文。WebSocket 升级头同时用于浏览器推送和 agent 通信。

```nginx
map $http_upgrade $monitor_connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen 443 ssl;
    server_name hub.example.com;
    ssl_certificate /etc/letsencrypt/live/hub.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hub.example.com/privkey.pem;
    client_max_body_size 5m;

    location / {
        proxy_pass http://127.0.0.1:28080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $monitor_connection_upgrade;
        proxy_read_timeout 300s;
        proxy_buffering off;
    }
}
```

也可以用 Caddy 自动管理 TLS：

```caddyfile
hub.example.com {
    reverse_proxy 127.0.0.1:28080
}
```

完成代理配置后通过 `https://hub.example.com/admin` 登录。手动启动 hub 时也应显式使用 `--listen 127.0.0.1:28080`，因为二进制自身的默认监听地址是通配地址。

## Docker

发布工作流会将镜像推到 `ghcr.io/blubo127/monitor`。先确认本项目已发布镜像并允许当前用户拉取；该路径是发布目标，不代表首次发布前已经存在。以下示例使用命名卷，并只向宿主机 loopback 发布端口：

```bash
docker volume create monitor-data
docker run -d --name monitor-hub --restart unless-stopped \
  -p 127.0.0.1:28080:28080 \
  --mount source=monitor-data,target=/data \
  -e TZ=Asia/Shanghai \
  ghcr.io/blubo127/monitor:latest
docker logs monitor-hub
```

按上一节配置 HTTPS 反向代理。镜像以 UID/GID `65534:65534` 运行；使用宿主目录挂载时，要给这个用户数据目录写入权限。固定版本时，将 `:latest` 替换为实际发布的版本标签，版本号不带 `v`。

## 安装 agent

在面板添加节点，使用面板生成的安装命令。安装器从你部署的 hub 下载；hub 再从 `Blubo127/agent` 的正式 Release 中继对应架构的二进制。这个流程要求自有 agent 仓库已经发布 `monitor-agent-x86_64-unknown-linux-musl` 和/或 `monitor-agent-aarch64-unknown-linux-musl`。

如果希望先阅读脚本，再执行它，可以把面板命令改为先下载：

```bash
curl -fsSL https://hub.example.com/install.sh -o install-agent.sh
# 阅读 install-agent.sh 后，替换下面的 NODE_TOKEN 为面板生成的节点 token。
sudo sh ./install-agent.sh --server https://hub.example.com --token NODE_TOKEN
```

支持 systemd 和 OpenRC。二进制位于 `/opt/monitor/monitor-agent`，token 和 hub 地址写入 `/opt/monitor/agent.env`。常用选项为 `--interval 1`（1–3600 秒）和 `--iface eth0`（只计指定网卡；`--iface -eth0` 表示排除该网卡）。

## 批量安装与升级

在后台“批量添加”打开注册窗口，复制包含 `--register` 的命令到目标机器执行。注册窗口有效期为一小时，每台机器会获得独立 node token；安装完可以立即关闭窗口。`--name 名称` 可为新节点指定名称。

升级已有 agent 时，从自己的 hub 取安装器，沿用节点已有的配置：

```bash
curl -fsSL https://hub.example.com/install.sh -o install-agent.sh
sudo sh ./install-agent.sh --upgrade
```

升级 hub 时，重新运行本项目的 `install-hub.sh`。它会读取已有端口和 `--site`，下载本项目最新正式 Release，并在服务启动失败时尝试回滚旧二进制。

之前从上游安装的旧 `install-hub.sh` 内部仍然指向上游，必须用本项目版本替换后再执行。已运行的旧 hub 也仍然使用编译时写入的 agent 下载和版本查询地址；先部署包含本次地址修改的 hub，再通过它安装或升级节点，地址迁移才会生效。

## 主题

hub 构建时使用上游 [monitor-probe/monitor-theme-default](https://github.com/monitor-probe/monitor-theme-default) 的默认主题，下载逻辑在 `scripts/theme.sh`，版本和 SHA-256 固定在 `web-theme.pin`。如果将来改为自有内置主题，需要先发布主题包，再同步修改下载源和 pin。

自有可选主题仓库为 [Blubo127/Theme-Glassmorphism](https://github.com/Blubo127/Theme-Glassmorphism)。发布有效的 `theme.tar.gz` 后，可以在后台“主题”中填该仓库 URL 安装，或手动上传主题包。主题包的 `theme.json.url` 决定以后“从 GitHub 更新”的来源；自有构建包也应把此字段改为自有仓库地址，否则更新仍会回到原作者仓库。

主题包会在访客浏览器执行 JavaScript，安装前应核对来源。上游文档和主题目录仅作参考：[上游文档站](https://monitor-document.pages.dev)、[上游主题站](https://monitor-themes.pages.dev)。这些站点不是本项目部署的站点。

## 从源码构建

需要 Git、Rust stable、本机 C 编译器和链接器、Node.js 24、npm、`curl`、`sha256sum` 和 `tar`。在构建环境中检出 `diy` 分支：

```bash
git clone --branch diy https://github.com/Blubo127/monitor.git
cd monitor
cd web-admin
npm ci
npm run build
cd ..
sh scripts/theme.sh
cargo build --release --locked
```

`build.rs` 也会调用 `scripts/theme.sh`；主题已有匹配的 `.pin` 时会复用。构建产物为 `target/release/monitor-hub`，可手动运行：

```bash
./target/release/monitor-hub --listen 127.0.0.1:28080 --db ./monitor.db
```

首次运行会显示应急密码，本机后台地址为 `http://127.0.0.1:28080/admin`。添加节点前仍需配置节点可达的 HTTPS 域名；源码构建不会改变 agent 的下载来源，面板安装节点仍需要 `Blubo127/agent` 的正式 Release。首次发布前的 hub/agent 联调见[源码配对检查](./RELEASE.md#首次发布前的集成检查)。

## 卸载和备份

```bash
sudo sh ./install-hub.sh --uninstall
sudo sh ./install-agent.sh --uninstall
```

上述两条命令分别用于 hub 主机和 agent 节点。hub 默认卸载保留数据；`--purge` 会删除整个 `/opt/monitor/data/`，包括数据库和主题。数据库和导出的备份包含节点 token、OAuth secret 等凭据，应限制访问。历史安全检查见 [check_summary.md](./check_summary.md)，它记录的是地址迁移前的受检提交。
