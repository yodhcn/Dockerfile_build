# Dockerfile Build · 通用镜像构建器

给任意一个 GitHub 仓库打包 Docker 镜像：clone 指定 ref → 在代码目录里 `find` Dockerfile → `docker build` → 导出 `tar.gz` → 上传为 workflow artifact。

**不推送任何镜像仓库**，产物只以 artifact 形式交付（GHCR / Docker Hub 都不需要，也不需要配置任何 secret）。

## 触发方式

`Actions` → `Build Image Artifact` → `Run workflow`，填写输入后运行。

## 输入参数

| 输入 | 必填 | 默认 | 说明 |
| --- | --- | --- | --- |
| `repo` | ✅ | `owner/repo` | GitHub 仓库。支持 `owner/repo`、完整 https URL、`git@github.com:` 形式 |
| `ref` | ✅ | `main` | 分支 / tag / commit SHA（用浅 fetch 单 ref，三者都支持） |
| `image_name` | ✅ | `my-app` | 目标镜像名，**必须小写**（含大写会自动转小写并告警），可含 `/` 分级 |
| `image_tag` | ✅ | `latest` | 目标镜像 tag |
| `dockerfile` | | 空 | Dockerfile 相对仓库根的路径；留空则自动 `find` |
| `context` | | 空 | 构建上下文目录（相对仓库根）；留空 = **仓库根** |
| `build_args` | | 空 | 额外 `--build-arg`，每行一个 `KEY=VALUE` |
| `repo_token` | | 空 | 仅私有仓库需要：具备 repo 读权限的 PAT。也可配置名为 `REPO_TOKEN` 的仓库 secret（优先用输入值） |
| `platform` | | `linux/amd64` | 目标平台 |

## Dockerfile 定位规则

未指定 `dockerfile` 时，会扫描仓库（跳过 `.git`、`node_modules`），收集 `Dockerfile` / `Dockerfile.*` / `*.dockerfile`，按下面优先级选一个：

1. 文件名**精确**为 `Dockerfile`
2. `Dockerfile.*`（如 `Dockerfile.cuda`）
3. `*.dockerfile`

每档内部按**目录深度浅者优先**，深度相同按字典序。所有候选会列在 job summary 的表格里并标出选中项——如果选错了，把正确路径填进 `dockerfile` 重跑即可。

**构建上下文默认是仓库根**，也就是 `docker build -f <dockerfile> .` 的语义。若目标仓库的 Dockerfile 写在子目录且 `COPY` 是相对自身目录的（monorepo 常见），把 `context` 填成那个子目录。

## 拿到产物

job 结束后在 run 页面底部下载 artifact，包含三个文件：

```
<image>-<tag>.tar.gz           # 镜像归档
<image>-<tag>.tar.gz.sha256    # 校验和（只写文件名，同目录可直接 sha256sum -c）
<image>-<tag>.manifest.txt     # 镜像 ID / 体积 / 上游 ref 与 commit / Dockerfile 路径 / run 链接
```

导入：

```bash
docker load -i my-app-latest.tar.gz
# 或 gunzip -c my-app-latest.tar.gz | docker load
sha256sum -c my-app-latest.tar.gz.sha256
docker run --rm -it my-app:latest <你的命令>
```

## 需要注意

- **artifact 会过期**：默认保留 30 天（可在 workflow 里改 `retention-days`）。镜像只存在这一份，过期前记得下载留存。
- **10 GB 硬上限**：GitHub 对单个 artifact 有 10 GB 限制且不可调。归档前已用 `pigz`（无则 `gzip`）压缩，超过 9 GB 会直接报错终止，不会传一半上去。
- **私有仓库**：填 `repo_token`，或建一个名为 `REPO_TOKEN` 的 secret。token 会被 `add-mask` 遮蔽，不会出现在日志里。
- **构建耗时**：`timeout-minutes: 120`；跑前会先清一次 runner 磁盘（hosted runner 可用空间只有 ~14 GB）。同 repo/ref 重复构建走 GHA 缓存。
- **多平台**：默认只出 `linux/amd64`。填 `platform` 为 `linux/arm64` 会在 amd64 runner 上走 QEMU 模拟，慢很多，且上游镜像本身未必有 arm64 基础镜像——需要的话先确认再改。

## 文件

```
.github/workflows/docker-build-artifact.yml   # 全部逻辑在这一个文件里
README.md
```

本仓库不提供 Dockerfile——Dockerfile 由被构建的上游仓库自带。
