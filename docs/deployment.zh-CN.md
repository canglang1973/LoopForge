# 部署

> 本文是 `docs/deployment.md` 的中文版本。英文版为内容基准，两份文档需要同步维护；两者出现差异时，以英文版为准。

## 原生开发

`bash scripts/dev.sh` 会显式配置本地密钥，仅启动当前检出目录对应的 MySQL 服务，构建平台，创建 `admin` 账号，通过一个显式脚本安装缺失的可观测性生成列，应用 `migrations/seed/*.sql`，并在 8090 端口启动服务器。现有 catalog 定义由 `INSERT IGNORE` 保留。请将 `.env`、`.local/` 和数据库卷排除在版本控制之外。

原生安装时，请在 API 主机上使用 Node.js 22+，并安装你所选择的 CLI。CodeBuddy 也用于需求澄清；定时 assistant 任务使用任务所选定的受支持 CLI：

```bash
npm install --global @tencent-ai/codebuddy-code
codebuddy --version
# Optional executors for Main Bot and SOP roles:
npm install --global @openai/codex@0.162.1 @anthropic-ai/claude-code@2.1.300
```

确保服务在 PATH 上能找到 `codebuddy` 和 Node.js，或将 `AGENTFLOW_CODEBUDDY_BIN` 设为已安装可执行文件的路径。在个人设置（Personal settings）中配置模型密钥及其网络环境。无密钥的脚本演示不需要该 CLI。生产 Docker 镜像包含 CodeBuddy、Codex 和 DeepSeek Harness（`dsh`）。Claude Code 和其他 agent CLI 不会被打包。当某次对话因缺少某个 CLI 而失败时，错误信息会说明其未安装，并提供安装按钮。该按钮会将固定版本的 `@anthropic-ai/claude-code` 发布包，或 OpenCode、Kimi Code 和 Pi 的公共包下载到 `$LOOPFORGE_DATA_DIR/cli`；在镜像外部安装时，则以相同方式获取固定版本的 `@deepseek-ai/dsh` 发布包。Cursor CLI 和 Antigravity 使用各自提供方的安装程序，并安装到服务用户的 `~/.local/bin`。关闭或重新加载浏览器不会停止已经开始的安装，按钮会一直显示“自动安装”，直到安装完成。安装完成后，原先的生成失败提示会替换为要求重新发送消息的提示。下一条消息会复用该安装。已发布镜像的 SOP（标准作业程序）sandbox（沙箱）如果选用 Claude Code、OpenCode、Kimi Code 或 Pi，会在首次使用时安装同一个包，并与对话共享。Cursor CLI 和 Antigravity 不会安装到 SOP sandbox 中。已连接的 Runner（执行器）仍需自行满足各自的 CLI 要求。

基于自定义 Dockerfile 且不含 Node.js、npm 或 Python 3 的对话工作区会使用平台自带的副本。首次使用时，平台会将其 Node.js、npm 和可重定位的 Python 复制到 `$LOOPFORGE_DATA_DIR/cli/toolchain/<build>/`；某个 sandbox 首次需要某个随附 CLI（CodeBuddy、Codex、DeepSeek Harness）时也复制该 CLI，并将该目录以只读方式挂载到 sandbox 的 `PATH` 末尾。镜像自带的工具优先。仅平台架构上的 glibc 镜像会使用这套工具链；Alpine（musl）镜像需要自备 Node.js、npm 和 Python 3。新的平台构建会创建新目录，并保留旧目录：升级后请删除其他 `cli/toolchain/<build>/` 目录（每个约 200 MB，另加复制到其中的 CLI）。

按 Ctrl-C 停止前台服务。使用同一命令再次启动。MySQL 数据保留在其 Compose 卷中。`docker compose stop mysql` 会停止数据库但不删除它。

如需使用自己的 MySQL 部署，请使用 `AGENTFLOW_MYSQL_DSN`。全新的账号命令或服务器连接会创建缺失的 schema 对象。审阅前提条件后，逐项应用适用于现有安装的数据迁移；不要盲目对新数据库重放全部历史迁移。

## 容器安装

```bash
./start.sh             # install or start in the background (same as bash scripts/install.sh)
./start.sh logs        # follow platform logs
./start.sh stop        # stop; ./start.sh down removes containers but keeps data volumes
./start.sh restart     # restart the platform container
./start.sh pull        # refresh images and recreate changed containers
MIRROR=ghcr.nju.edu.cn ./start.sh  # same install; replace the ghcr.io host with this registry
./start.sh dev         # build this checkout instead (compose.dev.yaml)
./start.sh release     # switch a source build back to the published image
```

安装器会检查 Docker、Compose、Python 3 和 curl。它会在 Linux 上使用 apt、dnf 或 yum 安装缺失的 Python/curl 软件包（macOS 上使用 Homebrew），使用官方安装程序安装 Docker，并使用带 SHA-256 校验的固定官方发布版本安装缺失的 Linux x86_64/arm64 Compose 插件。插件会为当前活跃 Docker 用户安装；当 Docker 需要 sudo 时则进行系统级安装。macOS 的 Compose 来自 Docker Desktop。设置 `LOOPFORGE_YES=1` 可接受安装提示；仍然需要互联网访问以及包管理器/管理员权限。安装器会生成带唯一密钥的 `.env`，拉取并启动容器，安装可观测性列并应用 `migrations/seed/*.sql`。可安全地重复运行。首次运行前若默认端口已被占用，请在环境中设置 `AGENTFLOW_HTTP_PORT` 或 `LOOPFORGE_MYSQL_PORT`（或稍后编辑 `.env`）。

请以安装所有者身份运行，包括使用普通 Linux 用户。如果该用户无法直接访问 Docker，安装器会校验 sudo，并对 Compose、schema 安装和 seed 一致使用 `sudo -n docker`。每次操作前，它会检查 sudo 凭据是否在长时间构建期间过期；在终端中会再次提示，在无人值守模式下则失败并给出说明。无需将用户加入 Docker 组或注销后重新登录。无人值守安装请先运行 `sudo -v`；`LOOPFORGE_YES=1` 会接受安装提示，但不会绕过操作系统权限。若没有 sudo 或 Docker 访问权限，安装会立即失败并给出说明。Rootless Docker 使用其活跃上下文的 Unix socket；远程 Docker 上下文需要另行配置部署，因为 sandbox 工作区是本地 bind mount（绑定挂载）。

`compose.yaml` 是已发布镜像的栈，且永远不会自行构建：`./start.sh`、`scripts/install.sh` 以及直接执行 `docker compose --profile platform up` 都会拉取其中固定引用的 GHCR 发布版本（当前为 `ghcr.io/tencent/loopforge:0.3.8`）。该固定版本与检出目录中的迁移和 seed 一致，因此安装时请检出发布标签。仅在需要覆盖时设置 `LOOPFORGE_IMAGE`（例如设为 `@sha256:` 摘要）；保存在 `.env` 中的值会优先于之后的固定版本变更。

升级时，先更新检出目录，再刷新容器：

```bash
git fetch --tags && git checkout vX.Y.Z
./start.sh pull
```

新的 `compose.yaml` 会同时带来镜像固定版本和任何新的平台设置；当任一内容发生变化时，`./start.sh pull` 会重建平台容器。由 `./start.sh dev` 创建的源代码构建不会被 `./start.sh pull` 重建；更新检出目录后需再次运行 `./start.sh dev`。仅运行 `./start.sh pull` 会保留固定版本。如果 `.env` 中保存了 `LOOPFORGE_IMAGE`，也要将其改为所检出的发布版本；较新的镜像搭配较旧的 `compose.yaml` 可能缺少该版本所需的设置。

如果 `ghcr.io` 下载停滞，请将 `MIRROR` 设为可替换 `ghcr.io` 的镜像仓库主机。`./start.sh` 和 `./start.sh pull` 会拉取该引用，并将其重新标记为 `compose.yaml` 中的名称。设置 `MIRROR` 即信任该主机，因为结果会以官方名称打标签。GitHub Release 会记录镜像摘要；执行 `docker image inspect --format '{{index .RepoDigests 0}}' ghcr.io/tencent/loopforge:<pin>` 后，`@` 之后显示的摘要应与其一致。安装时拉取策略保持为 `missing`，因此会保留本地镜像；`./start.sh pull` 会从同一主机刷新它。不设置 `MIRROR` 会保留直接拉取 `ghcr.io` 的行为。[ghcr.nju.edu.cn](https://doc.nju.edu.cn/books/e1654/page/ghcr) 是一个兼容主机。Docker 的 `registry-mirrors` 设置，包括 `https://docker.nju.edu.cn`, 仅适用于 Docker Hub，不能用作 `MIRROR`。不是 `ghcr.io` `name:tag` 形式的 `LOOPFORGE_IMAGE` 值会保持不变。

```bash
MIRROR=ghcr.nju.edu.cn ./start.sh
MIRROR=ghcr.nju.edu.cn ./start.sh pull
```

`./start.sh dev`（或 `LOOPFORGE_BUILD_FROM_SOURCE=1 bash scripts/install.sh`）会根据 `compose.dev.yaml` 将当前检出构建为 `loopforge-platform:local`。它会在 `.env` 中保存 `COMPOSE_FILE=compose.yaml:compose.dev.yaml`，因此后续运行和直接执行 `docker compose` 命令会继续使用源代码构建，直到运行 `./start.sh release`。源代码构建会复用 npm、Go module 和 Go 编译器缓存。

在 `main` 上推送 `vX.Y.Z` 标签后，只要 `platform` 工作流已在该提交上通过，就会运行 `release` 工作流。带标签的 `compose.yaml` 必须已经固定 `loopforge:X.Y.Z`。该工作流会把 `ghcr.io/<owner>/loopforge` 发布为 `X.Y.Z`、`X.Y`、`latest` 和 `sha-<commit>`，使用 `./start.sh` 对推送的摘要进行冒烟测试，并创建 GitHub Release。Fork 会以各自的 owner 发布；分发镜像前，请为你的团队设置 GHCR 包的可见性/访问权限。默认仅缺失时拉取镜像；请使用不可变标签/摘要，或在更新可变标签时设置 `LOOPFORGE_PULL_POLICY=always`。现有 `.env` 值优先；更新安装时，请在该文件中修改已保存的镜像或 URL。在安装 schema 扩展和 seed 之前，Compose 会在有限超时内等待 API 健康检查通过。

打开平台并在浏览器中创建第一个账号；该账号获得平台管理权限。在该账号存在之前，登录页（login page）仅提供此设置步骤。

初始设置期间，请将云安全组或防火墙限制为操作员的 IP。在管理员账号创建之前，不要将首个账号设置页面暴露给任意互联网客户端。

平台监听 `LOOPFORGE_BIND`，默认值为 `0.0.0.0`，不会提示输入访问 URL。安装器会打印公网、局域网和本地地址；在腾讯云、阿里云、AWS、GCP 和 Azure 上，它会从提供方的链路本地元数据服务读取公网 IP；在其他位于 NAT 后的环境中，你需要自行开放公网 IP。`LOOPFORGE_PUBLIC_URL` 是可选项，仅改变打印出的地址。请在云安全组/防火墙中允许 HTTP 端口，或使用 HTTPS 反向代理。现有 `.env` 文件会被保留；没有 bind 设置的旧安装现在也会监听 `0.0.0.0`。仅本地安装可设置 `LOOPFORGE_BIND=127.0.0.1`。请代理 WebSocket 和流式请求。当 HTTPS 在代理处终止时，设置 `AGENTFLOW_SECURE_COOKIES=true`。在多个 API 实例之间以及重启后，保持 `AGENTFLOW_SESSION_SECRET` 稳定不变。

企业身份认证网关可通过设置 `AGENTFLOW_TRUST_IDENTITY_HEADERS=true` 让用户通过请求头登录。该选项默认关闭。之后，API 仅接受来自回环对等端的 `Staffname` / `x-staff-name`，并且仅当其能够判断请求来自网关时才会接受：

- 将 `AGENTFLOW_IDENTITY_HEADER_SECRET` 设为随机值，并让网关通过 `X-AgentFlow-Gateway-Secret` 发送该值。网关必须覆盖而非透传客户端的 `Staffname`、`Staffid`、`x-staff-name` 和 `X-AgentFlow-Gateway-Secret` 请求头。
- 没有密钥时，任何携带 `X-Forwarded-For`、`Forwarded` 或 `X-Real-IP` 的请求都会被视为转发客户端，其身份请求头会被忽略。此机制只保护会添加上述请求头之一的代理：同主机代理如果在透传客户端请求头时不添加这些头（例如 nginx 的 `proxy_pass` 未配置 `proxy_set_header X-Forwarded-For`），以及主机上的任何其他进程，仍可冒充任意用户登录。启用基于请求头的登录时必须设置该密钥；若缺少密钥，平台会在启动时记录警告。

### Docker sandbox（沙箱）

即使尚未安装 catalog SQL，平台也会暴露一个内置默认 sandbox（`local`）。它是内存中的回退实现，不会插入数据库行。已有保存的 `local` 配置文件优先，包括其已配置的驱动和环境。默认配置不能删除；请编辑其配置或创建其他配置文件来改变任务运行方式。完整安装器仍会导入初始 SOP 和欢迎项目。

对于以普通主机用户身份运行的原生服务，default/root Docker 镜像会以该服务用户的 UID/GID 运行，因此私有工作区无需放宽权限或添加 capability 即可保持可访问。显式选择非 root 用户的镜像会保留该身份；对这些镜像，操作员必须安排兼容的 bind mount（绑定挂载）所有权。

Compose 部署会把主机 Docker socket 挂载到平台容器中，使内置 AI SOP 可以开箱即用。`AGENTFLOW_ISOLATED_IMAGE` 指向已安装的平台镜像：默认 sandbox 会从同一个镜像启动独立的同级容器，无需第二次下载 CLI 或构建镜像。它们不会挂载 Docker socket。自定义 Dockerfile 仍会构建并缓存自己的镜像。运行工作区挂载在 Compose 网络上的 `/workspace`。由于 bind mount 由主机 Docker daemon 解析，`LOOPFORGE_DATA_DIR`（由 `scripts/configure.sh` 写入 `.env`，默认为 `./.data`）会以相同的绝对路径挂载到平台容器中；工作区和 CLI 状态位于其下。原生对话工作区会以同样方式 bind mount 各自的对话根目录，因此 `compose.yaml` 将 `AGENTFLOW_CONVERSATION_DIR` 设为 `${LOOPFORGE_DATA_DIR}/conversations`。附件和其他按用户隔离的 assistant 状态保留在 `AGENTFLOW_ASSISTANT_SESSION_DIR`（位于 `platform-data` 卷中的 `/data/assistant`），该目录绝不会挂载到工作区。若平台容器的 `AGENTFLOW_CONVERSATION_DIR` 未设置，对话根目录也会保留在那里，此时打开对话工作区会因 `mkdirat ... read-only file system` 失败。

可访问 Docker socket 等同于在主机的 root 权限。由此需要两项防护措施：平台会在 sandbox 容器中运行每个任务，包括为 `local` 驱动配置的 SOP（`AGENTFLOW_LOCAL_SANDBOX=docker`）；并且只有平台管理员才能让仓库指向主机源路径。即便如此，任何能够运行 SOP 的人都可在此主机的容器中执行代码，因此只应向信任的人授予账号，不要将服务暴露给不受信任的网络。要在不使用该 socket 的情况下运行，请从 `compose.yaml` 中移除 `docker.sock` 卷和 `AGENTFLOW_LOCAL_SANDBOX` 设置，然后为 AI SOP 连接具备 Docker 能力的独立 Worker 或执行机器；此后仅脚本的欢迎 SOP 会在平台容器内运行。

### 慢速网络：克隆与镜像构建限制

克隆仓库和构建自定义 sandbox 镜像时，会在工作过程中报告进度，出错时会保留 git 或 Docker 输出的最后几行。在平台主机上，进度会记录到对应运行中。在执行机器上，进度会写入 Runner 的日志，只有最终错误会到达平台。某一步骤在其停滞窗口内没有任何输出时，会被视为挂起并停止。持续有输出的步骤可能很慢，但只会被总时限停止。所有设置都接受 Go duration，例如 `90s` 或 `20m`；`0` 会关闭限制。每台主机读取自己的环境，因此执行机器上也需设置克隆限制。

| 设置 | 默认值 | 适用范围 |
|---|---|---|
| `AGENTFLOW_GIT_CLONE_STALL_TIMEOUT` | `5m` | 无输出的 `git clone`。当平台从仓库的本地源路径克隆时不适用，因为其最终 repack 没有输出。 |
| `AGENTFLOW_GIT_CLONE_TIMEOUT` | 平台上关闭，执行机器上为 `10m` | 整个 `git clone`。执行机器保留其早先固定的 10 分钟限制作为默认值。 |
| `AGENTFLOW_DOCKER_BUILD_STALL_TIMEOUT` | `10m` | 无输出的 `docker build`。静默的 `RUN` 步骤（长时间编译）也算无输出。 |
| `AGENTFLOW_DOCKER_BUILD_TIMEOUT` | 关闭 | 整个 `docker build`。 |

以这种方式停止的克隆不会重试，因为重试会等待同一网络。当执行主机访问 GitHub 或 Debian 镜像源缓慢时，请让 git 使用代理（`git config --global http.proxy`），并在 Dockerfile 中使用附近的软件包镜像，而不是提高这些限制。

### 通过 SSH 访问 Git 仓库

对于 GitLab.com、自托管 GitLab、Gitea、Gogs、GitHub 以及其他提供该传输方式的 Git 主机，请使用 Git SSH。在 **Project settings → Code repositories**（项目设置 → 代码仓库）或 **Quick access**（快速访问）中，输入克隆地址，例如 `git@gitlab.com:group/repo.git` 或 `ssh://git@git.example.org:2222/group/repo.git`，并选择现有默认分支。GitHub HTTPS 仍继续使用 GitHub 授权或 PAT。

1. 部署管理员创建专用的 OpenSSH `known_hosts` 文件。添加主机密钥前，请通过提供方发布的指纹或服务器管理员进行核对。使用精确主机名或 IP 地址；非默认端口写成 `[host]:port`。通配符和哈希条目不会授予访问权限。对于可信团队部署，请在同一文件中显式添加内部主机。
2. 将 `AGENTFLOW_GIT_SSH_KNOWN_HOSTS` 设为该文件在平台、Worker 和执行机器上的绝对路径。使用 Compose 时，将其放在现有的 `LOOPFORGE_DATA_DIR` bind mount 下，并在 `.env` 中设置绝对路径。在执行机器和自定义 sandbox 镜像中安装 Git 与 OpenSSH 客户端。将执行机器升级到 Runner **0.9.3** 或更高版本。Docker sandbox 命令需要 sandbox 和执行进程共享 Linux 内核，以使用签名 agent socket。当 Docker VM 与原生 Runner 分离时，请使用托管检出与交付，或在交互式 Git 命令中使用原生 sandbox。
3. 为无人值守执行生成不带口令的专用 SSH 密钥。将每个公钥添加到你的托管账号，并授予其所需仓库的访问权限。如果运行需要推送，请授予写权限。在 **Personal settings → Git SSH keys**（个人设置 → Git SSH 密钥）中，为每个 SSH 主机添加一个私钥，例如 `gitlab.com` 或 `git.example.org:2222`。省略端口表示 22。编辑该主机的条目即可替换其密钥。

仓库会自动使用当前用户在其 SSH 主机和端口上的密钥；同一主机和端口上的仓库共享该密钥。仅当该主机没有个人密钥时，仓库表单才会显示红色警告。此提醒不会阻止保存仓库或完成设置；凭据和主机信任可以在首次 Git 操作前配置。不存在仓库专用密钥或默认回退密钥。检出和 SOP 执行使用负责人的密钥。对话提交和工作区终端使用请求用户的密钥。临时签名 agent 会在命令或终端结束时关闭。终端的 agent 绝不会驻留在共享工作区中：使用 `bwrap` 驱动时，它位于仅绑定到该终端的私有目录中；使用 `local` 驱动（无隔离）时，它位于工作区之外的 `0700` 目录中。Docker 和 seatbelt 工作区中的所有进程共享同一文件系统视图，因此其终端不会获得个人 SSH 密钥；请从这些环境使用平台的提交操作推送。除非 `provider` 显式设为 `ssh`，`git@github.com:` 和 `ssh://git@github.com/` 地址会保存为 GitHub 仓库（HTTPS + token、自动拉取请求）。要轮换密钥，请在设置中替换密钥，并在主机上吊销旧公钥。

SSH 负责 clone、fetch 和 push。对话交付会返回分支和提交 SHA。你可以选择添加仓库 Web URL，以便打开主机并手动创建 PR/MR。Issue 导入和 PR/MR 自动化需要托管平台 API 凭据。要自动执行这些操作，请在 **Settings → Assets → MCP**（设置 → 资产 → MCP）中添加特定托管平台的 MCP，为其提供独立且限定范围的 token，并将其挂载到相关 agent 或工作流。[CNB 不支持 Git SSH](https://docs.cnb.cool/en/guide/git-access.html)；它使用 HTTPS 和 CNB 访问令牌，这不在本集成范围内。

## 账号

在浏览器中创建的第一个账号会成为平台管理员。之后，只要 `AGENTFLOW_ALLOW_SIGNUP` 为 `true`（默认值），任何能访问登录页（login page）的人都可以注册为普通用户。注册用户可以在你的 Worker 上创建需求并运行 SOP，因此对于任何可从不受信任网络访问的安装，请设置 `AGENTFLOW_ALLOW_SIGNUP=false`，并改为显式创建账号。

平台管理员在 **Platform administration → Member accounts**（平台管理 → 成员账号）中创建账号：输入登录名和初始密码，可选择授予管理权限，然后将密码交给队友。同一页面会列出所有账号，并可重置密码账号的密码；通过 GitHub 或网关登录的账号无法在此处设置密码。新账号会和自助注册用户一样加入欢迎项目。

LoopForge 假定使用可信团队模型。每个已登录账号都能列出所有项目、需求和运行。项目设置和仓库只能由项目所有者或平台管理员更改，个人凭据只能由其所有者更改。不要用同一套安装来隔离不受信任的租户。

没有浏览器时，请导出原生安装的 `.env` 值，并运行显式账号命令：

```bash
set -a; source .env; set +a
read -r -s -p 'Password: ' account_password; echo
printf '%s\n' "$account_password" | go run ./cmd/loopforge-account -user alice
unset account_password
```

仅在授予平台管理权限时使用 `-admin`。再次运行该命令会轮换该账号的密码。密码哈希使用 bcrypt。浏览器会话使用签名的 HttpOnly、SameSite cookie，并在七天后过期。任何具名人员都不会获得隐式管理员角色；其他管理员可通过现有平台管理 UI（Platform administration UI）进行管理。

## Worker 与 Runner

上游进程角色仍为 `-role api`、`-role worker` 和 `-role all`。API 与 Worker 实例必须共享 MySQL 数据库及适合其部署的配置。分离控制与执行时，请设置 `AGENTFLOW_CONTROL_TOKEN`、`AGENTFLOW_CONTROL_URL` 和 `AGENTFLOW_WORKER_URL`。将 `AGENTFLOW_PLATFORM_MCP_URL` 设为执行容器可访问的实际平台 URL。

个人 Runner 通过执行机器页面（execution-machine page）上的短期绑定码进行连接。安装脚本位于 `/runners/install.sh`。`AGENTFLOW_RUNNER_BINARY` 可指向支持 `{os}` 和 `{arch}` 替换的二进制文件。容器镜像会打包 Linux/macOS amd64/arm64 的 runner 和 gate 二进制文件。原生源码安装可使用 Go 按需构建。

平台管理员可在 **Platform administration → Shared CodeBuddy Key**（平台管理 → 平台共享 CodeBuddy Key）下共享一个 CodeBuddy API Key：选择签发该密钥的网络环境以及成员可使用的模型（第一个为默认）。没有自己密钥的成员随后可使用该密钥聊天和运行 SOP，但仅限这些模型；如果某个角色配置了其他模型，它会改用默认模型运行，且运行记录会说明这一点。拥有自己密钥的成员仍可将其用于任意模型。定时任务、澄清、报告、bot 和运行评估仍需要 Runner 自己的密钥。定时任务为其所选 CLI 使用该密钥，不会借用共享 CodeBuddy 密钥。该密钥会像个人密钥一样存储在数据库中，保存后不再显示。`AGENTFLOW_ASSISTANT_SPONSOR_LOGIN` 会借出一个账号的密钥并将其限制为 `deepseek-v4.1-flash`；在没有配置共享密钥时，该设置仍然生效。

请在平台 UI 中配置提供方凭据，而不是主机的全局 CLI 配置。凭据由上游运行时绑定到责任执行主体。首页输入区（home composer）和 SOP 角色各自独立选择内置 CLI。在首页模型选择器（home model selector）旁选择执行器；其 API 密钥（以及可选的 Codex/Claude base URL）来自你的个人设置。更改执行器会保留草稿；已保存的对话会记录所用执行器。Main Bot 写入仍会经过平台工具网关及其审批检查。需求澄清和通知继续使用 CodeBuddy。定时 assistant 任务可选择任何受支持的 CLI，并作为普通对话轮次运行；Codex 和 Claude Code 仍需要该账号自己的凭据。实时模型测试需主动启用；仅对需要加载本地凭据的验收套件设置 `AGENTFLOW_LIVE_TESTS=1`。

## 可观测性数据库结构

基础表创建后，请为 Compose 数据库运行 `bash scripts/install-schema-extensions.sh`。该脚本会根据上游迁移 `001_run_obs_generated_columns.sql` 安装缺失的 `run_telemetry.adapter` 和 `run_logs.level` 列及其索引，然后重放使用 `INSERT IGNORE` 的初始 catalog。这两个步骤均可重复执行，且不会改动现有行。安装器会在打印地址前运行该脚本，而 `./start.sh pull` 会在重建容器后再次运行，因此升级到更新镜像的旧数据卷会获得新代码读取的列。应用启动不会运行数据迁移。

对于外部 MySQL 部署，请先审阅该迁移并检查列是否存在，然后再应用相应语句。存储的生成列可能重建大型现有表；请据此安排操作时间。

### 升级需要哪些迁移

`migrations/` 包含三类脚本。请查看发行说明以了解特定版本新增了哪些文件，然后在运行前阅读脚本自身的头部说明。

| 类型 | 示例 | 操作员需要做什么 |
| --- | --- | --- |
| 安装器和 `./start.sh pull` 已应用 | `001_run_obs_generated_columns.sql`, `migrations/seed/*.sql`, `031_generic_backend_openssh.sql` | 无需操作；上述脚本已涵盖 |
| 升级时必须运行一次 | `009_bot_interactions_utf8mb4.sql`（对现有表执行 `CONVERT TO`）、手工新增的表和列 | 审阅前提条件后在数据库上运行 |
| 对现有行进行回填和一次性重写 | `016_public_optional_repositories.sql`, `021_assistant_session_sources.sql` | 如果历史数据重要则运行；跳过不会更改 schema，但会保留旧行未转换 |

## 更新现有的公共 SOP

仓库绑定对于所有内置通用 SOP 都是可选的。要更新现有安装上的原始 catalog 指令，请显式应用：

```bash
docker compose exec -T mysql sh -c 'MYSQL_PWD="$MYSQL_PASSWORD" mysql -u loopforge loopforge' < migrations/016_public_optional_repositories.sql
```

该可重复执行的 SQL 只会更改仍与原始公共 seed 匹配的指令字段。它会保留操作员的编辑和项目副本。新安装已通过公共 catalog seed 获得这些指令；应用启动不会迁移数据。

## GitHub 凭据

用户可以在个人凭据（personal credentials）中选择 **Connect GitHub**（连接 GitHub），或在添加仓库时选择该操作，复制显示的 PIN，并在 GitHub 上批准。平台会将该授权保存到该已登录用户现有的 Git 凭据槽位。仓库选择器（repository picker）会列出该账号拥有的、与其共享的或可通过其组织访问的仓库，支持对已加载仓库分页和搜索，并将所选仓库的真实默认分支复制到表单中。仍可使用 PAT 输入方式。

新安装使用内置的 LoopForge OAuth App。用户无需创建应用、配置 Client ID 或申请 PAT：选择 **Connect GitHub**（连接 GitHub），复制 PIN，并在 GitHub 上批准 LoopForge 应用即可。GitHub 的确认无法绕过；设备授权不是零交互登录。

应用配置属于部署事项，不会显示在个人凭据中。需要自有应用的操作员可以注册 OAuth App，启用 **Device Flow**（设备流程），在刷新支持可用前保持 token 过期功能关闭，并在 `.env` 中使用 `AGENTFLOW_GITHUB_CLIENT_ID` 设置其公开 Client ID。无需 client secret。部署环境会覆盖内置应用以及任何数据库覆盖值。仅管理员可用的 `PUT /api/auth/github/config` 端点仍可用于显式数据库覆盖；空的 Client ID 会显式禁用设备授权，而不是静默恢复默认值。现有显式配置会被保留。

该流程会请求 OAuth `repo` scope，以便私有克隆、推送、Issue 和 PR 操作正常工作。GitHub 的 OAuth scope 覆盖可访问的仓库；在 LoopForge 中选择某个仓库只会将其绑定到项目，不会缩小 OAuth 授权范围。组织 SSO 或第三方应用限制可能需要组织批准。参见 [GitHub 的设备授权文档](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#device-flow)。待处理的授权会过期，且仅保存在 API 内存中：在多 API 实例后方为 `/api/auth/github/*` 使用会话粘性，并在 API 重启后重新启动授权。token 和 GitHub 设备码绝不会发送到浏览器。

GitHub Issue 导入和 PR 状态同步使用项目所有者的 Git token。Git 操作和可选的 GitHub MCP 集成使用责任用户的 token。公开读取可能无需 token 即可工作；私有仓库和写入需要仓库访问权限。GitHub MCP 必须显式安装，并且仅将凭据发送到其官方 HTTPS 端点。

平台创建的 Git 提交使用责任本地账号名和不可投递的 `@users.noreply.loopforge.invalid` 邮箱。本地账号不会建立企业邮箱或已验证的 GitHub 身份。

## CodeBuddy 身份认证环境

在个人设置（Personal settings）中，选择 API 密钥所属的网络环境；“get API key”（获取 API 密钥）链接会打开该版本的密钥页面（key page）。来自某一版本的密钥发送到另一版本时会返回 HTTP 401，第三方模型提供方（DeepSeek、OpenAI 等）的密钥不是 CodeBuddy 密钥。该设置按用户生效；公共安装不假定为企业环境。

| 版本（Edition） | 存储值（Stored value） | 密钥页面（Key page） |
| --- | --- | --- |
| 国际版 | 空 | https://www.codebuddy.ai/profile/keys |
| 中国版 | `internal` | https://copilot.tencent.com/profile/ |
| iOA | `ioa` | https://tencent.sso.copilot.tencent.com/profile/keys |

`PUT /api/users/:id/credentials` 仅接受这些值（也接受 `public` 和 `China`），其他任何值都会以 HTTP 400 拒绝，因为 CLI 对无法识别的值会静默回退到国际版。

其他专用/自托管配置见 [CodeBuddy 官方身份文档](https://www.codebuddy.ai/docs/cli/iam)。使用 `AGENTFLOW_ASSISTANT_MODEL` 和 `AGENTFLOW_ASSISTANT_MODELS` 配置模型 catalog；普通角色可独立选择模型。

## 隔离容器验收

运行 `python3 scripts/test-install.py -v` 可执行安装器回归测试，覆盖 sudo 选择、直接 Docker 访问、无人值守权限失败、重复安装、镜像模式和访问 URL。使用本地构建的平台镜像时，`LOOPFORGE_INSTALL_TEST_IMAGE=your-image bash scripts/test-install-live.sh` 会在随机端口上的隔离 Compose 项目中验证真实的 root 和 UID 65534 安装、非回环 HTTP 访问、首个账号创建以及重复安装。该实时测试需要 Linux root 测试运行器来降权；测试后会自行移除其容器和卷。若还要在一次性 UID 1000 安装器容器内实际测试 sudo，请使用 `--build-arg LOOPFORGE_TEST_BASE=your-image` 构建 `scripts/testdata/install-sudo.Dockerfile`，然后将该测试镜像作为 `LOOPFORGE_INSTALL_SUDO_TEST_IMAGE` 传入。生产安装器本身不需要 root。

在仓库根目录下，安装 `e2e/` 依赖和 Chromium 后，运行 `bash scripts/test-container.sh`。它会构建生产镜像，在端口 8093/33317 上创建隔离的 Compose 项目和数据库，创建测试账号，显式应用可观测性基线和公共 catalog，检查随附 CLI 以及跨平台 runner/gate 二进制文件，并运行公共浏览器套件。测试后会移除其容器和卷。如果这些端口已被占用，请覆盖 `LOOPFORGE_CONTAINER_TEST_PORT` 和 `LOOPFORGE_CONTAINER_MYSQL_PORT`。此次干净安装检查会禁用模型调用。
