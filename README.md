# DockerToAliRepo

使用 GitHub Actions 将上游 Docker/OCI 镜像转存到阿里云容器镜像服务（ACR）。保留原有的 images.txt 批量同步方式，并新增 Linux 命令行工具 gdocker，支持按需同步单个镜像。

## 功能

- **批量同步**：编辑 images.txt 后 push 到 main，或手动运行 Actions。
- **按需同步**：Linux 服务器运行 gdocker pull IMAGE[:TAG]，脚本触发本仓库的 workflow_dispatch，等待转存完成后再调用原生 Docker 从 ACR 拉取。
- **多架构支持**：按需模式不指定平台时，使用 Skopeo 复制完整多架构 manifest；指定 --platform 时使用独立的平台 tag。
- **统一 ACR namespace**：按需模式默认目标命名空间为 lvbey，并把上游注册表和 repository 映射为确定性的目标 repository 名称。

## GitHub Actions 配置

进入 **Settings → Secrets and variables → Actions**。

### Repository Variables

| 变量 | 示例 | 说明 |
|---|---|---|
| ALIYUN_REGISTRY | registry.cn-hangzhou.aliyuncs.com | ACR 控制台提供的登录地址，通常只填主机名，不含协议 |
| ALIYUN_NAME_SPACE | lvbey | 目标命名空间；按需模式默认 lvbey |
| ALIYUN_REGISTRY_USER | ACR 用户名 | 推送镜像的账号 |

### Repository Secrets

| Secret | 说明 |
|---|---|
| ALIYUN_REGISTRY_PASSWORD | ACR 推送账号密码 |

不要将真实密码或 PAT 提交到仓库。ACR 地址以控制台提供的登录地址为准，个人版与企业版实例的域名格式可能不同。

## ACR 首次推送前的必要设置（旧版个人版）

按需同步会把镜像推送到 `lvbey/<目标仓库名>:<tag>`。旧版个人版 ACR 在目标仓库不存在时，只有当命名空间允许自动创建仓库，推送才会自动创建仓库。否则即使日志显示 `Login Succeeded!`，Skopeo 仍可能在上传 Layer 时返回 `requested access to the resource is denied`。

请在阿里云控制台完成以下检查：

1. 进入容器镜像服务 ACR，切换到杭州地域的旧版个人版实例。
2. 打开 **仓库管理 → 命名空间**，找到 `lvbey` 并进入命名空间设置。
3. 确认 **自动创建仓库** 已开启并保存。
4. 确认 GitHub Actions 中的 `ALIYUN_REGISTRY_USER` 和 `ALIYUN_REGISTRY_PASSWORD` 是该实例的有效访问凭证，并且该凭证可推送镜像。
5. 重新运行失败的 GitHub Actions workflow。

如果不希望开启自动创建仓库，也可以在 **仓库管理 → 镜像仓库** 中，先手动创建本次目标仓库 `docker-io--library--nginx`，然后重新运行工作流。后续每个新上游镜像会映射为不同的仓库名；若自动创建保持关闭，就需要提前创建每一个目标仓库。

官方说明：
- [命名空间管理：自动创建仓库设置](https://help.aliyun.com/zh/acr/user-guide/manage-namespaces)
- [Docker push 失败排查](https://help.aliyun.com/zh/acr/support/faq-about-errors-of-docker-login-docker-push-and-docker-pull)

## gdocker 安装

适用于已安装 Docker 的 Linux 主机，需要 Bash、curl、jq、flock、sed 和 sha256sum。

### 1. 下载脚本

~~~bash
sudo install -d -m 755 /usr/local/bin
sudo curl -fsSL https://raw.githubusercontent.com/Lvbey/DockerToAliRepo/main/gdocker -o /usr/local/bin/gdocker
sudo chmod 755 /usr/local/bin/gdocker
~~~

### 2. 创建配置

创建当前用户自己的配置文件 ~/.config/gdocker.conf：

~~~bash
mkdir -p ~/.config
chmod 700 ~/.config
install -m 600 /dev/null ~/.config/gdocker.conf
editor ~/.config/gdocker.conf
~~~

配置示例（请替换 Token 和实际 ACR 地址）：

~~~bash
GH_TOKEN='github_pat_REPLACE_ME'
GH_REPOSITORY='Lvbey/DockerToAliRepo'
GH_WORKFLOW_FILE='docker.yaml'
GH_REF='main'

ALIYUN_REGISTRY='registry.cn-hangzhou.aliyuncs.com'
ALIYUN_NAMESPACE='lvbey'

WAIT_TIMEOUT_SECONDS=900
POLL_INTERVAL_SECONDS=5
~~~

GitHub PAT 必须有目标仓库触发 Actions 工作流所需的权限。建议使用 fine-grained PAT，并遵循最小权限原则。配置文件包含敏感令牌，必须限制为当前用户可读（权限 600）。gdocker 默认读取 ~/.config/gdocker.conf；如需指定其他位置，可设置 GDOCKER_CONFIG 环境变量。

ACR 推送账号密码只保存在 GitHub Actions Repository Secret 中。Linux 主机需提前通过 docker login 登录 ACR，使用具有拉取权限的凭据；gdocker 不会保存 ACR 密码。

### 3. 使用

~~~bash
gdocker pull nginx:1.27
gdocker pull redis:7.4.1
gdocker pull ghcr.io/open-webui/open-webui:cuda

# 指定平台；平台使用独立的 ACR tag
gdocker pull --platform linux/amd64 nginx:1.27

# 查看最近一次任务状态和 GitHub Actions 日志入口
gdocker status
gdocker logs
gdocker config
~~~

工作流成功后，gdocker 使用原生 Docker 从 ACR 拉取镜像，并额外打上原始镜像 tag，后续可以直接执行 docker run。

## 镜像名称映射

按需模式将上游 registry 与 repository 转换为小写的确定性 ACR repository 名，并将路径分隔符转换为双连字符。示例：

| 输入镜像 | ACR 目标镜像（namespace 为 lvbey） |
|---|---|
| nginx:1.27 | {ACR_REGISTRY}/lvbey/docker-io--library--nginx:1.27 |
| redis:7.4.1 | {ACR_REGISTRY}/lvbey/docker-io--library--redis:7.4.1 |
| bitnami/redis:7.4 | {ACR_REGISTRY}/lvbey/docker-io--bitnami--redis:7.4 |
| ghcr.io/open-webui/open-webui:cuda | {ACR_REGISTRY}/lvbey/ghcr-io--open-webui--open-webui:cuda |

显式指定平台时，目标 tag 会追加平台后缀，例如 1.27--linux-amd64，避免不同平台覆盖同一个 tag。未指定平台时，工作流尝试保留上游完整的多架构 manifest。

当前按需模式支持常规 IMAGE[:TAG] 引用；digest 引用（@sha256:...）暂未支持。需要认证的上游私有 registry 暂不在第一版范围内。

## 原有 images.txt 批量模式

批量模式继续支持每行一个镜像、空行、注释以及可选平台参数：

~~~text
#nginx:1.27
redis:7.4.1
--platform=linux/amd64 mysql:8.4
~~~

修改并 push images.txt 到 main 后会自动运行批量工作流；也可以在 Actions 页面手动触发。批量模式继续沿用原有镜像命名逻辑，按需模式使用上文的确定性映射。

## 故障排查与安全

- 无法触发工作流：检查 PAT 权限、仓库名、workflow 文件名和分支配置。
- 工作流失败：执行 gdocker logs 打开 GitHub Actions 运行页面。
- 最终 docker pull 认证失败：先在服务器执行 docker login <ACR登录地址>，确保账号至少有拉取权限。
- 不指定平台时按需模式使用 Skopeo 复制多架构 manifest；指定平台时会复制所选平台并使用独立 tag。
- GitHub PAT 与 ACR 凭据不应出现在命令行参数、镜像名或日志中。
- 仅转存你有权访问和使用的镜像。
