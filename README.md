# aps2api

aps2api 是一个 Google Gemini API 反向代理服务，将客户端的标准 Gemini 请求转发至上游端点，并提供并发竞速与代理节点管理。

## 特性

- 仅暴露官方 `/v1beta` 端点，透传原生 JSON 与 SSE 流式增量。
- 基于滑动窗口的并发竞速（RunRace），首包胜出后取消其他候选，超时或 429 自动轮换。
- 内嵌 Mihomo 节点池，支持 VLESS、VMess、Trojan、Shadowsocks、Hysteria2 等协议。
- 内置 Web 控制台，管理 API 密钥、节点、订阅及系统参数。

## 运行

```sh
# Linux / macOS / Android (Termux)
./start.sh

# Windows
启动.bat
```

首次运行会在标准错误输出打印随机生成的管理员密码。

Web 控制台地址：`http://127.0.0.1:2156/admin/`

### Debian VPS 使用 Docker 镜像

仓库的 GitHub Actions 会在推送到 `master`、推送 `v*` 标签或手动触发时，构建并发布镜像到 `ghcr.io/l1ngth6/aps2api`，支持 `linux/amd64` 和 `linux/arm64`。PR 仅验证构建，不发布镜像。`latest` 跟随默认分支；版本标签（如 `v1.3.0`）及 `sha-<完整提交 SHA>` 可用于固定版本。

首次使用前，将工作流推送到 GitHub 并等待构建成功。发布使用内置 `GITHUB_TOKEN`（工作流已声明 `packages: write`），无需配置 Docker Hub 密钥。首次发布的 GHCR 包默认为私有：如需免登录拉取，请到 GitHub 个人主页的 **Packages → aps2api → Package settings → Change visibility** 设置为 Public。私有镜像需先在 VPS 执行 `docker login ghcr.io -u <GitHub用户名>`，密码使用具有 `read:packages` 权限的 PAT（classic）。参见 [GitHub Container Registry 文档](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)。

在已安装 Docker Engine 和 Compose 插件的 Debian VPS 上：

```sh
mkdir -p ~/aps2api && cd ~/aps2api
curl -fsSL https://raw.githubusercontent.com/l1ngth6/APS2api/master/docker-compose.ghcr.yml -o compose.yml
docker compose pull
docker compose up -d
docker compose logs --tail=100 vproxy
```

在日志中查看首次生成的管理员密码，访问 `http://<VPS-IP>:2156/admin/`。`config`（含数据库）、`logs`、`assets` 保存在当前目录。对外提供服务时可按[部署指南](部署指南.md#nginx-反向代理配置)配置 HTTPS 反向代理。

更新镜像并重建容器：

```sh
docker compose pull && docker compose up -d
```

固定版本时，在 `compose.yml` 同目录的 `.env` 中设置 `APS2API_IMAGE=ghcr.io/l1ngth6/aps2api:v1.3.0`（替换为已发布标签）。Fork 仓库请改为自己的镜像地址。本地源码构建仍可使用原有 `docker-compose.yml`。

## 客户端配置

在支持 Google Gemini API 的客户端或 SDK 中配置：

- Base URL: `http://127.0.0.1:2156`
- API Key: `sk-...`（在 Web 控制台的密钥管理中添加）

### 支持端点

- `GET  /v1beta/models`
- `GET  /v1beta/models/{model}`
- `POST /v1beta/models/{model}:generateContent`
- `POST /v1beta/models/{model}:streamGenerateContent`

## 编译

环境依赖：Go 1.22+；修改前端需要 Bun。

```sh
# 前端构建（可选，仓库已包含内置静态资源）
cd web && bun install && bun run build && cd ..

# 二进制编译
go build -trimpath -ldflags="-s -w" -o aps2api ./cmd/vproxy
```

## 配置

配置文件路径为 `config/config.json`，支持运行时热重载。

| 字段 | 类型 | 默认值 | 说明 |
| :--- | :--- | :--- | :--- |
| `port_api` | int | `2156` | 服务监听端口 |
| `admin_password` | string | 自动生成 | Web 控制台密码 |
| `max_retries` | int | `1` | 候选轮换重试预算 |
| `parallel_pool_enabled` | bool | `true` | 是否启用并发竞速 |
| `parallel_pool_size` | int | `15` | 滑动窗口并发数 |
| `race_timeout` | int | `60` | 单节点首包等待超时（秒） |
| `stream_idle_timeout_seconds` | int | `20` | 流式包间超时（秒） |
| `request_timeout` | int | `180` | 会话超时（秒） |
| `max_request_mb` | int | `128` | 最大请求体限制（MB） |
| `max_spill_mb` | int | `32` | 内存缓冲溢出阈值（MB） |
