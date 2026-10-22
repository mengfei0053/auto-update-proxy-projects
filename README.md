# Auto Update Proxy Projects

这个仓库用于自动跟踪若干上游项目的 GitHub Release，并把对应版本的 Docker 镜像发布到 GHCR（GitHub Container Registry）。CLIProxyAPI 与 9router 在 runner 上构建；OmniRoute 从上游已发布的多架构镜像复制。

仓库本身不包含应用源码，主要维护的是 `.github/workflows/` 下的 GitHub Actions 工作流。

## 工作流列表

| Workflow | 上游项目 | GHCR 镜像 | 架构 | 定时任务 |
| --- | --- | --- | --- | --- |
| `.github/workflows/update-cliproxyapi-image.yml` | [`router-for-me/CLIProxyAPI`](https://github.com/router-for-me/CLIProxyAPI) | `ghcr.io/mengfei0053/proxy-agents-cliproxyapi` | `linux/amd64`, `linux/arm64` | 每天 UTC 16:07（北京时间 00:07） |
| `.github/workflows/update-9router-image.yml` | [`decolua/9router`](https://github.com/decolua/9router) | `ghcr.io/mengfei0053/proxy-agents-9router` | `linux/amd64` | 每天 UTC 16:17（北京时间 00:17） |
| `.github/workflows/update-omniroute-image.yml` | [`diegosouzapw/OmniRoute`](https://github.com/diegosouzapw/OmniRoute) | `ghcr.io/mengfei0053/proxy-agents-omniroute` | `linux/amd64`, `linux/arm64` | 每天 UTC 16:27（北京时间 00:27） |

所有工作流都支持在 GitHub Actions 页面通过 `workflow_dispatch` 手动触发。

## 自动更新逻辑

共同逻辑：

1. 使用 GitHub CLI 查询上游仓库最新 Release tag：

   ```bash
   gh release view --repo <upstream> --json tagName -q .tagName
   ```

2. 登录 GHCR。
3. 使用 `docker manifest inspect "$IMAGE:$TAG"` 检查目标镜像仓库是否已经存在该 tag。
   - 已存在：跳过。
   - 不存在：认为上游有新版本，需要发布。
4. 对 CLIProxyAPI 与 9router：检出上游仓库指定 tag，用 Docker Buildx 构建并推送 `:<tag>` 和 `:latest`。
5. 发布成功后，在当前仓库创建一条 Release，记录镜像、上游 Release、CHANGELOG 等信息。

> 版本对比只基于镜像 tag 是否存在，不做 semver 比较。

### OmniRoute

OmniRoute 不在 runner 上构建（Next.js 会把 `ubuntu-latest` 打到内存上限）。上游 [`diegosouzapw/OmniRoute`](https://github.com/diegosouzapw/OmniRoute) 的 release tag 带前导 `v`（例如 `v3.8.52`），已发布镜像 tag 去掉这一个 `v`：

- `ghcr.io/diegosouzapw/omniroute:3.8.52`
- `diegosouzapw/omniroute:3.8.52`（Docker Hub）

两个仓库都是公开的多架构镜像（`linux/amd64` 与 `linux/arm64`）。工作流优先用 `skopeo copy --all` 复制 GHCR，源不存在时再试 Docker Hub，推送到：

- `ghcr.io/mengfei0053/proxy-agents-omniroute:v3.8.52`（与上游 release tagName 一致）
- `ghcr.io/mengfei0053/proxy-agents-omniroute:latest`

是否需要复制仍用 `docker manifest inspect` 检查我们自己的 `:$TAG`。Release 说明仍记录上游 Release 正文；根目录 CHANGELOG 通过 Git tree API 查找，不再 checkout 上游仓库。

## 拉取镜像

示例：

```bash
docker pull ghcr.io/mengfei0053/proxy-agents-cliproxyapi:latest
docker pull ghcr.io/mengfei0053/proxy-agents-9router:latest
docker pull ghcr.io/mengfei0053/proxy-agents-omniroute:latest
```

如果需要固定版本，请将 `latest` 替换为上游 Release tag。

## 权限配置

工作流需要两类权限：

1. 推送 GHCR 镜像。
2. 在当前仓库创建 Release，记录本次更新的镜像和上游 changelog。

登录 GHCR 时使用：

```yaml
password: ${{ secrets.GHCR_TOKEN || secrets.GITHUB_TOKEN }}
```

建议在仓库 Secrets 中配置：

| Secret | 用途 | 建议权限 |
| --- | --- | --- |
| `GHCR_TOKEN` | 推送用户作用域 GHCR package | classic PAT，`write:packages` |
| `RELEASE_TOKEN` | 创建当前仓库 Release | classic PAT，`repo` |

如果仓库设置允许 `GITHUB_TOKEN` 写入 contents，也可以不配置
`RELEASE_TOKEN`。如果不允许，Release 创建会输出 warning，但不会影响已经完成的镜像推送。

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
