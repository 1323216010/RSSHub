# RSSHub 自定义版本部署

本文档记录本仓库自定义订阅源的开发、自动构建、服务器部署、官方更新同步和回滚流程。

## 架构

- `origin`：`https://github.com/1323216010/RSSHub.git`，个人 fork。
- `upstream`：`https://github.com/DIYgod/RSSHub.git`，RSSHub 官方仓库。
- `master`：保持与官方 `master` 同步。
- `custom`：存放自定义订阅源及部署配置。
- GitHub Actions：将 `custom` 分支构建为 `ghcr.io/1323216010/rsshub:custom`。
- 服务器：`/opt/rsshub`，通过 Docker Compose 运行 RSSHub、Redis 和 Browserless。

服务器不直接编译源码。构建工作由 GitHub Actions 完成，服务器只拉取镜像，因此更适合内存较小的主机。

## 开发与推送

从 `custom` 分支开发新的订阅源：

```bash
git switch custom
git pull --ff-only origin custom
```

完成开发和测试后提交：

```bash
git add <changed-files>
git commit -m "feat(route): add example route"
git push origin custom
```

当提交修改了路由代码、Docker 构建文件或工作流时，GitHub Actions 会自动生成：

- `ghcr.io/1323216010/rsshub:custom`
- `ghcr.io/1323216010/rsshub:custom-<commit-sha>`

`custom` 标签始终指向最近一次成功构建，带提交哈希的标签用于精确回滚。

## 首次部署

服务器仓库目录为 `/opt/rsshub`。部署时使用官方 Compose 文件和自定义覆盖文件：

```bash
cd /opt/rsshub
git fetch origin
git switch custom
git pull --ff-only origin custom
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml pull
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml up -d
```

如果 GHCR 镜像为私有，需要先登录。Token 只授予 `read:packages` 权限，不要写入仓库：

```bash
docker login ghcr.io -u 1323216010
```

随后在交互提示中输入 Token。

## 日常部署更新

GitHub Actions 构建成功后，在服务器执行：

```bash
cd /opt/rsshub
git pull --ff-only origin custom
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml pull rsshub
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml up -d rsshub
```

## 同步官方更新

在本地更新 `master`：

```bash
git switch master
git fetch upstream
git merge --ff-only upstream/master
git push origin master
```

然后将官方更新合并到 `custom`：

```bash
git switch custom
git merge master
```

解决可能的冲突并完成测试后：

```bash
git push origin custom
```

新增订阅源应放入独立 namespace 目录，并尽量避免修改公共核心文件，这样同步官方代码时发生冲突的概率较低。

## 检查状态

```bash
cd /opt/rsshub
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml ps
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml logs --tail=100 rsshub
curl -fsS http://127.0.0.1:1200/healthz
```

公网默认访问地址为 `http://39.102.75.98:1200/`。还需要在云服务器安全组中允许 TCP 1200 入站，或使用反向代理通过 HTTPS 暴露服务。

## 回滚

先确定要恢复的成功构建提交哈希，然后临时修改 `deploy/docker-compose.custom.yml` 中的镜像标签：

```yaml
services:
    rsshub:
        image: ghcr.io/1323216010/rsshub:custom-<commit-sha>
```

重新拉取并启动：

```bash
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml pull rsshub
docker compose -f docker-compose.yml -f deploy/docker-compose.custom.yml up -d rsshub
```

确认恢复后，将镜像标签改回 `custom`。

## 安全注意事项

- 不要将 SSH 密码、GitHub Token、Cookie 或站点账户凭据提交到 Git。
- GitHub Actions 使用仓库自动提供的 `GITHUB_TOKEN` 推送镜像。
- 服务器拉取私有镜像时只使用 `read:packages` 权限。
- 服务器应使用 SSH 密钥登录，并关闭 root 密码登录。
- 对公网部署建议设置 RSSHub `ACCESS_KEY`，并通过 HTTPS 反向代理访问。

