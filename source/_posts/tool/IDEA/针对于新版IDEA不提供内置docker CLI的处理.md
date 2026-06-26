---
title: 针对于新版IDEA不提供内置docker CLI的处理
date: 2026-05-01 
keywords: Docker IDEA
categories: [IDEA]
---


# 注意：如果你本地已经安装了 Docker Desktop 或其他完整 Docker 发行版，一般可以忽略本文

# 1. 背景

新版 IDEA 不再内置 `docker` CLI 和 `buildx` 相关指令，需要用户自行安装并在 IDEA 中手动指定路径。

这类场景通常出现在：

1. 本机没有安装 Docker Desktop，只希望补齐客户端工具。
2. 本机有单独的 Docker daemon，需要手动指定本地 `docker` 和 `buildx` 可执行文件。
3. IDEA 无法自动探测到本地 Docker CLI。

# 2. 配置位置

![[attachments/Pasted image 20260319095859.png]]

进入 Docker 配置页面后：

1. 关闭自动检测。
2. 手动指定 `docker` 可执行文件路径。
3. 确认 `buildx` 插件已放到 Docker CLI 约定目录中。

如果只配置了 `docker`，但没有安装 `buildx`，在执行镜像构建时仍然可能报错。

# 3. 相关文件下载

## 3.1 安装 Docker CLI

需要先准备一个可以连接 Docker daemon（服务端）的 `docker` client。

[Docker CLI 下载地址](https://download.docker.com/mac/static/stable/)

注意事项：

1. 根据自己的系统架构选择目录，不要只看 `x86_64`。
2. Apple Silicon 机器优先选择 `aarch64` 或对应 ARM 目录。
3. 下载版本需要和 Docker daemon 的 API 版本兼容。

[Docker Engine API 版本对应关系](https://docs.docker.com/reference/api/engine/#api-version-matrix)

如果你已经有远端或本机 Docker daemon，只是缺少 CLI，那么下载并解压后，把 `docker` 可执行文件放到本地可访问目录即可，例如：

```bash
mkdir -p ~/bin
cp docker ~/bin/docker
chmod +x ~/bin/docker
```

然后在 IDEA 中指定该路径。

## 3.2 安装 Docker Buildx 插件

### 3.2.1 下载地址

[Docker Buildx 下载地址](https://github.com/docker/buildx?tab=readme-ov-file#windows-and-macos)

### 3.2.2 安装方式

下载后，把文件放到对应目录：

| OS      | Binary name         | Destination folder                  |
| ------- | ------------------- | ----------------------------------- |
| Linux   | `docker-buildx`     | `$HOME/.docker/cli-plugins`         |
| macOS   | `docker-buildx`     | `$HOME/.docker/cli-plugins`         |
| Windows | `docker-buildx.exe` | `%USERPROFILE%\.docker\cli-plugins` |

macOS / Linux 示例：

```bash
mkdir -p ~/.docker/cli-plugins
cp docker-buildx ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx
```

IDEA 一般会自动从上述目录中寻找 `buildx` 指令。

# 4. 安装完成后的检查

建议先在终端验证一遍，再回到 IDEA 测试：

```bash
docker version
docker buildx version
```

如果 `docker version` 能正常输出，但 `docker buildx version` 报错，通常说明 `buildx` 插件没有放对位置，或者没有执行权限。

# 5. 常见错误处理

## 5.1 `ERROR: failed to build: Error response from daemon: client version 1.53 is too new. Maximum supported API version is 1.43`

原因：

`docker` client 版本过新，但当前连接的 Docker daemon 版本较老，二者 API 不兼容。

处理方式：

1. 参考 3.1 中的版本对应关系。
2. 降低本地 `docker` CLI 版本，或者升级 Docker daemon。
3. 确保 client 和 daemon 的 API 版本在兼容范围内。

## 5.2 `failed to fetch metadata: fork/exec /.docker/cli-plugins/docker-buildx: no such file or directory`

原因通常有以下几种：

1. `buildx` 文件不存在。
2. 文件放错目录。
3. 文件没有执行权限。

处理方式：

1. 参考 3.2，把 `docker-buildx` 放到正确目录。
2. 执行 `chmod +x ~/.docker/cli-plugins/docker-buildx`。
3. 再次执行 `docker buildx version` 验证。

# 6. 补充说明

1. 仅安装 `docker` CLI 和 `buildx`，并不等于本机已经有 Docker daemon。
2. 如果本地没有 Docker 服务，还需要额外准备 Docker Desktop、Colima、远端 Docker Host 等运行环境。
3. 如果 IDEA 中仍然无法识别，优先检查可执行文件路径、文件权限以及 daemon 连通性。
