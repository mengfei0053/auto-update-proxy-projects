# Auto Update Proxy Projects

这个仓库用于自动跟踪若干上游项目的 GitHub Release，并将对应版本构建成 Docker 镜像推送到 GHCR（GitHub Container Registry）。

仓库本身不包含应用源码，主要维护的是 `.github/workflows/` 下的 GitHub Actions 工作流。

## 工作流列表

| Workflow | 上游项目 | GHCR 镜像 | 架构 | 定时任务 |
| --- | --- | --- | --- | --- |
| `.github/workflows/update-cliproxyapi-image.yml` | [`router-for-me/CLIProxyAPI`](https://github.com/router-for-me/CLIProxyAPI) | `ghcr.io/mengfei0053/proxy-agents-cliproxyapi` | `linux/amd64`, `linux/arm64` | 每天 UTC 16:07（北京时间 00:07） |
| `.github/workflows/update-9router-image.yml` | [`decolua/9router`](https://github.com/decolua/9router) | `ghcr.io/mengfei0053/proxy-agents-9router` | `linux/amd64` | 每天 UTC 16:17（北京时间 00:17） |
| `.github/workflows/update-omniroute-image.yml` | [`diegosouzapw/OmniRoute`](https://github.com/diegosouzapw/OmniRoute) | `ghcr.io/mengfei0053/proxy-agents-omniroute` | `linux/amd64`, `linux/arm64` | 每天 UTC 16:27（北京时间 00:27） |

所有工作流都支持在 GitHub Actions 页面通过 `workflow_dispatch` 手动触发。

## 自动更新逻辑

每个工作流的核心逻辑一致：

1. 使用 GitHub CLI 查询上游仓库最新 Release tag：

   ```bash
   gh release view --repo <upstream> --json tagName -q .tagName
   ```

2. 登录 GHCR。
3. 使用 `docker manifest inspect "$IMAGE:$TAG"` 检查目标镜像仓库是否已经存在该 tag。
   - 已存在：跳过构建。
   - 不存在：认为上游有新版本，需要构建并推送。
4. 检出上游仓库指定 tag。
5. 使用 Docker Buildx 构建镜像，并推送：
   - `:<tag>`
   - `:latest`
6. 构建成功后，在当前仓库创建一条 Release，记录镜像、上游 Release、CHANGELOG 等信息。

> 版本对比只基于镜像 tag 是否存在，不做 semver 比较。

## 拉取镜像

示例：

```bash
docker pull ghcr.io/mengfei0053/proxy-agents-cliproxyapi:latest
docker pull ghcr.io/mengfei0053/proxy-agents-9router:latest
docker pull ghcr.io/mengfei0053/proxy-agents-omniroute:latest
```

如果需要固定版本，请将 `latest` 替换为上游 Release tag。

## GHCR 权限配置

工作流使用以下凭据登录 GHCR：

```yaml
password: ${{ secrets.GHCR_TOKEN || secrets.GITHUB_TOKEN }}
```

建议在仓库 Secrets 中配置 `GHCR_TOKEN`：

- 类型：classic PAT
- 权限：`write:packages`

原因：这些镜像是用户作用域的 GHCR package，默认 `GITHUB_TOKEN` 不一定具备写入已有 package 的权限。

工作流中已声明：

```yaml
permissions:
  contents: write
  packages: write
```

其中 `contents: write` 用于创建当前仓库的 Release，`packages: write` 用于推送镜像。

## 添加新的上游项目

新增工作流时，可以复制现有 workflow，并重点调整：

```yaml
env:
  UPSTREAM_REPO: <owner>/<repo>
  IMAGE: ghcr.io/mengfei0053/proxy-agents-<name>
  PLATFORMS: linux/amd64
```

添加前需要确认：

1. 上游仓库是否发布 GitHub Release。
2. 上游 tag 是否能直接用于 `actions/checkout`。
3. 上游根目录是否存在可用的 `Dockerfile`。
4. 目标构建架构是否与项目实际支持一致。
5. 如需多架构构建，是否需要 `docker/setup-qemu-action`。
6. 是否需要传入 build args，并确认这些参数确实在上游 `Dockerfile` 中声明。

## 运行注意事项

- GitHub Actions 的定时任务只会在默认分支上运行。
- GitHub 可能会在仓库长期无活动时禁用 scheduled workflow。
- 定时任务使用 UTC 时间，当前配置对应北京时间每天凌晨左右执行，并通过分钟偏移避免多个任务同时运行。
- 本仓库没有应用源码，也没有额外的本地构建、测试或 lint 步骤。
