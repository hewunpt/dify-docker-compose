# dify部署教程：Docker Compose 从零跑通，附容器自检与升级避坑清单

部署 Dify 最快的路子是 Docker Compose。一台 2 核 CPU、4 GiB 内存的机器，装好 Docker 和支持 2.24.0 以上版本的 Docker Compose，进入项目的 `docker` 目录复制一份环境文件，执行 `docker compose up -d`，等容器起来后打开 `http://服务器IP/install` 设置管理员账号，就能进控制台。

命令本身不多。容易出问题的是两个地方：`init_permissions` 容器跑完就退出，看着像部署失败；升级时把旧的 `docker-compose.yaml` 直接留着用，新版本需要的新服务起不来。

> **选购速览**：个人试用或小团队内部用，一台满足官方最低配置的云服务器加 Docker Compose 就够，不必一上来就折腾 Kubernetes。👉 [开通一台满足 2 核 4 GiB 的云服务器](https://bit.ly/yunhuodong)

## 部署 Dify 的机器要达到什么配置

官方给的最低线是 CPU 不少于 2 核、内存不少于 4 GiB。低于这个配置，容器未必起不来，但在插件调用、知识库文档处理这些环节会明显吃力。

软件这边按系统分：

| 环境 | 需要安装 | 官方说明 |
| --- | --- | --- |
| macOS 10.14 或更高 | Docker Desktop（含 Compose 2.24.0+） | Docker 虚拟机至少分配 2 个虚拟 CPU 和 8 GiB 内存 |
| Linux 发行版 | Docker 19.03+、Docker Compose 2.24.0+ | 分别按 Docker 引擎和 Compose 的安装说明操作 |
| Windows（启用 WSL 2） | Docker Desktop（含 Compose 2.24.0+） | 源码和绑定到 Linux 容器的数据建议放在 Linux 文件系统里，不要放 Windows 盘 |

这里有个很多人会忽略的差异：最低要求写的是 4 GiB，但 macOS 上用 Docker Desktop 时，官方要求给虚拟机分配 8 GiB。原因是 Dify 启动时要同时拉起 7 个核心服务、8 个依赖组件和 1 个一次性任务，本地开发环境只给 4 GiB，很容易在启动阶段就开始互相抢内存。

放到云服务器上则是另一回事：Linux 直接跑容器，没有虚拟机那一层开销，4 GiB 是可以跑的。要放进团队日常使用，内存留些余量更稳，具体开多大按并发人数和知识库规模估。

| 使用场景 | CPU | 内存 | 依据 |
| --- | --- | --- | --- |
| 功能试用、单人开发 | 2 核 | 4 GiB | 官方最低要求 |
| Mac 本地用 Docker Desktop | 分配给 Docker 2 vCPU | 分配给 Docker 8 GiB | 官方对 Docker Desktop 的要求 |
| 团队多人使用 | 2 核以上 | 4 GiB 以上，按并发留余量 | 官方只给下限，需自行评估 |

具体机型、配置和价格以云厂商页面当前展示为准。想省去挑机器这一步，可以直接从这里进：👉 [选择部署 Dify 的云服务器配置](https://bit.ly/yunhuodong)

## Dify 的三种部署方式怎么选

官方主推两条路：Docker Compose 和源码部署。Kubernetes 由社区维护，适合需要高可用的团队。

| 方式 | 适合谁 | 前置工具 | 上手难度 |
| --- | --- | --- | --- |
| Docker Compose | 绝大多数自托管用户、想先跑起来看效果的人 | Docker、Docker Compose 2.24.0+ | 低 |
| 源码部署 | 要改后端 Flask 或前端 Next.js 的开发者 | Python、Node.js、pnpm、uv | 中高 |
| Kubernetes | 要多副本、滚动升级的运维团队 | 集群，加社区 Helm Chart 或 YAML 文件 | 高 |

选 Docker Compose 的理由很直接：一条命令把 PostgreSQL、Redis、Web、API、Nginx 全套服务拉起来，不用逐个配依赖。源码部署适合改代码，前后端要分别配环境。Kubernetes 的 Helm Chart 和 YAML 有多个社区作者在维护，配置项比 Compose 多得多，人少的时候不划算。

## Docker Compose 部署 Dify 的完整步骤

按顺序走这四步，中途不要跳。

1. **克隆代码**。官方给的命令会先用 `curl` 取最新 release 的 tag 名，再传给 `git clone` 的 `--branch` 参数。所以本机要有 `git`、`curl`、`jq` 三个工具，缺一个都会报 `command not found`。如果报 `fatal: Remote branch null not found in upstream origin`，说明版本号没取到，改成显式写版本号重试：

   
   git clone --branch <version> https://github.com/langgenius/dify.git
   

   最新版本号在项目的 Releases 页面能看到。

2. **进 docker 目录，复制环境文件**。启动需要的配置值都在 `.env` 里，这份文件由 `.env.example` 复制而来：

   
   cd dify/docker
   cp .env.example .env
   

3. **确认 Compose 版本**。Dify 要求 Docker Compose 2.24.0 或更高，先跑一次 `docker compose version` 看清楚。版本低了直接升级，别在这一步省事。

4. **启动容器**：

   
   docker compose up -d
   

   它会拉起这些容器：
   - 7 个核心服务：`api`、`api_websocket`、`worker`、`worker_beat`、`web`、`plugin_daemon`、`agent_backend`
   - 8 个依赖组件：`weaviate`、`db_postgres`、`redis`、`nginx`、`ssrf_proxy`、`agent_ssrf_proxy`、`sandbox`、`local_sandbox`
   - 1 个一次性任务：`init_permissions`，负责设置存储文件权限，跑完自动退出

## 怎么确认 Dify 真的部署成功了

执行 `docker compose ps`，看每个容器的状态是不是 `Up` 或 `healthy`。这一步比看启动日志更快。

`init_permissions` 显示 `Exited` 是正常的，它只做一次权限设置，做完就退出，不会常驻。第一次部署的人常常在这里以为出事了，其实要看的是 `api`、`web`、`worker` 这些有没有正常起来。

如果某个容器反复重启，先 `docker compose logs` 看对应服务的报错，八成是端口被占用或内存不够。

## 访问 Dify 并完成管理员初始化

容器状态正常后，打开初始化页面设置管理员账户：

- 本地环境：`http://localhost/install`
- 服务器环境：`http://your_server_ip/install`

设置完成后从 `http://localhost` 或 `http://your_server_ip` 登录。用云服务器的话，访问不通先检查 80 端口有没有放行，这一步和 Dify 本身无关，但卡住的人不少。

## 部署后改配置：.env 和 docker/envs 谁优先

必要的启动值写在 `docker/.env` 里，直接编辑就行。可选配置和特定提供商的设置在 `docker/envs/` 下有对应模板，要用哪项就复制一份去掉 `.example` 后缀的副本再改。比如换向量库为 Milvus：


cd dify/docker
cp envs/vectorstores/milvus.env.example envs/vectorstores/milvus.env


要记住一条规则：`docker/.env` 里的值优先于任何 `docker/envs/*.env` 文件。改了配置之后重启：


docker compose down
docker compose up -d


## 升级 Dify 时最容易出错的地方

升级步骤不同版本可能不一样，按 Releases 页面里目标版本的升级说明走，别套用旧版本的流程。

升级后要做两件事。对照各 `.env.example` 检查自己的 `.env`，看有没有新增或变更的变量。如果你改过 `docker-compose.yaml`，把自定义内容重新应用到新版本的文件上，不要继续用旧的那份。新版本可能加了 Dify 运行必需的服务，旧文件里没有，起来就是缺胳膊少腿。

## 常见问题

**Dify 可以不用 Docker 部署吗？**
可以，走源码部署。后端是 Flask，前端是 Next.js，需要 Python、Node.js、pnpm、uv 这些工具，前后端要分别配置和启动。适合需要改代码的人，只想用功能的话没必要。

**能接本地大模型吗？**
可以。Dify 支持数十家推理提供商和自托管方案，包括 GPT、Mistral、Llama3 以及任何 OpenAI API 兼容的模型。本地跑 Ollama 再接入是常见做法。

**自托管和 Dify Cloud 怎么选？**
Dify Cloud 零部署，沙盒方案包含 200 次免费 GPT-4 调用，适合先试功能。自托管的数据和模型密钥都在自己机器上，适合对数据位置有要求、或者要长期高频调用的场景，成本结构也不一样。

**一台低配机器能跑吗？**
2 核 4 GiB 能跑起来，但同时运行十几个容器，余量不大。做功能验证够用，真要用起来建议加内存。

## 最后怎么选

只想尽快看到界面、验证功能：Docker Compose，一台 2 核 4 GiB 的云服务器，按上面四步走完就行。

要改后端逻辑或前端界面：源码部署，先按官方指南把 Python 和 Node.js 环境配好，再动代码。

要多副本、要滚动升级：看社区的 Helm Chart 或 YAML 文件，别在 Compose 上硬撑。

下单前确认两件事：机器能不能装到 Docker Compose 2.24.0 以上，80 端口能不能对外访问。这两个条件不满足，后面所有步骤都会卡住。

👉 [去腾讯云开通一台用于部署 Dify 的服务器](https://bit.ly/yunhuodong)
