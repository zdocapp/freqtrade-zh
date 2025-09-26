# 使用 Docker 运行 Freqtrade

本页介绍了如何使用 Docker 运行交易机器人。这并非开箱即用的解决方案，您仍需要阅读文档并理解如何正确配置。

## 安装 Docker

首先为您的平台下载并安装 Docker / Docker Desktop：

* [Mac](https://docs.docker.com/docker-for-mac/install/)
* [Windows](https://docs.docker.com/docker-for-windows/install/)
* [Linux](https://docs.docker.com/install/)

!!! Info "Docker compose 安装"
    Freqtrade 文档假定使用 Docker desktop（或 docker compose 插件）。  
    虽然独立的 docker-compose 安装仍然可用，但需要将所有 `docker compose` 命令从 `docker compose` 改为 `docker-compose` 才能工作（例如 `docker compose up -d` 将变为 `docker-compose up -d`）。

??? Warning "Windows 上的 Docker"
    如果您刚在 Windows 系统上安装了 Docker，请确保重启系统，否则可能会遇到与 Docker 容器网络连接相关的无法解释的问题。

## 使用 Docker 运行 Freqtrade

Freqtrade 在 [Dockerhub](https://hub.docker.com/r/freqtradeorg/freqtrade/) 上提供了官方 Docker 镜像，以及一个可直接使用的 [docker compose 文件](https://github.com/freqtrade/freqtrade/blob/stable/docker-compose.yml)。

!!! Note
    - 以下章节假设已安装 `docker` 且当前登录用户具有使用权限。
    - 所有下方命令均使用相对目录，必须在包含 `docker-compose.yml` 文件的目录中执行。

### Docker 快速入门

创建新目录并将 [docker-compose 文件](https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml) 放置于此目录中。

``` bash
mkdir ft_userdata
cd ft_userdata/
# Download the docker-compose file from the repository
curl https://raw.githubusercontent.com/freqtrade/freqtrade/stable/docker-compose.yml -o docker-compose.yml

# Pull the freqtrade image
docker compose pull

# Create user directory structure
docker compose run --rm freqtrade create-userdir --userdir user_data

# Create configuration - Requires answering interactive questions
docker compose run --rm freqtrade new-config --config user_data/config.json
```

以上代码片段将创建名为 `ft_userdata` 的新目录，下载最新的 compose 文件并拉取 freqtrade 镜像。
片段中的最后两个步骤将创建包含 `user_data` 的目录，并根据您的选择（交互式）生成默认配置。

!!! Question "如何编辑机器人配置？"
    您可以随时编辑配置，使用上述配置时，该配置位于 `ft_userdata` 目录内的 `user_data/config.json` 文件中。

    You can also change the both Strategy and commands by editing the command section of your `docker-compose.yml` file.

#### 添加自定义策略

1. 配置现已保存为 `user_data/config.json`
2. 将自定义策略文件复制到 `user_data/strategies/` 目录
3. 将策略的类名添加到 `docker-compose.yml` 文件

默认运行的是 `SampleStrategy` 策略。

!!! Danger "`SampleStrategy` 仅作为示例！"
    `SampleStrategy` 仅供您参考并为您自己的策略提供思路。
    请务必在投入真实资金前对策略进行回测，并先使用模拟交易运行一段时间！
    您可以在[策略文档](strategy-customization.md)中找到更多关于策略开发的信息。

完成此步骤后，您就可以启动机器人的交易模式（模拟交易或实盘交易，具体取决于您对上述相应问题的回答）。

``` bash
docker compose up -d
```

!!! Warning "默认配置"
    虽然生成的配置基本可用，但在启动机器人前，您仍需确认所有选项（如定价、交易对列表等）是否符合您的需求。

#### 访问用户界面

如果您在 `new-config` 步骤中选择了启用 FreqUI，则可以在端口 `localhost:8080` 访问 freqUI。

现在您可以在浏览器中输入 localhost:8080 来访问用户界面。

!!! Note "在远程服务器上访问 UI"
    如果您在 VPS 上运行，应考虑使用 SSH 隧道或设置 VPN（如 OpenVPN、WireGuard）来连接您的交易机器人。
    这将确保 FreqUI 不会直接暴露在互联网上，出于安全考虑不建议这样做（FreqUI 默认不支持 HTTPS）。
    这些工具的设置不在本教程范围内，但网上可以找到许多优质教程。
    请同时阅读 [Docker 下的 API 配置](rest-api.md#configuration-with-docker) 部分以了解更多关于此配置的信息。

#### 监控交易机器人

您可以使用 `docker compose ps` 检查运行中的实例。
这应该会显示 `freqtrade` 服务状态为 `running`。如果不是这种情况，最好检查日志（参见下一点）。

#### Docker compose 日志

日志将写入：`user_data/logs/freqtrade.log`。  
您还可以使用命令 `docker compose logs -f` 查看最新日志。

#### 数据库

数据库将位于：`user_data/tradesv3.sqlite`

#### 使用 Docker 更新 Freqtrade

使用 `docker` 更新 Freqtrade 只需运行以下两个命令：

``` bash
# Download the latest image
docker compose pull
# Restart the image
docker compose up -d
```

这将首先拉取最新镜像，然后使用刚拉取的版本重启容器。

!!! Warning "检查更新日志"
    您应始终检查更新日志以了解重大变更/需要手动干预的内容，并确保更新后机器人能正确启动。

### 编辑 docker-compose 文件

高级用户可以进一步编辑 docker-compose 文件以包含所有可能的选项或参数。

所有 freqtrade 参数都可通过运行 `docker compose run --rm freqtrade <command> <optional arguments>` 来使用。

!!! Warning "交易命令使用 `docker compose`"
    交易命令（`freqtrade trade <...>`）不应通过 `docker compose run` 运行，而应使用 `docker compose up -d`。
    这能确保容器正确启动（包括端口转发），并保证系统重启后容器会自动重启。
    如果您打算使用 freqUI，请确保相应调整[配置](rest-api.md#configuration-with-docker)，否则 UI 将不可用。

!!! Note "`docker compose run --rm`"
    包含 `--rm` 参数将在命令执行完成后删除容器，强烈推荐在除交易模式（使用 `freqtrade trade` 命令运行）外的所有模式下使用。

!!! Note "Using docker without docker compose"
    "`docker compose run --rm`" 命令需要提供 compose 文件。
    某些不需要身份验证的 freqtrade 命令（例如 `list-pairs`）可以使用 "`docker run --rm`" 来运行。  
    例如 `docker run --rm freqtradeorg/freqtrade:stable list-pairs --exchange binance --quote BTC --print-json`。  
    这对于获取交易所信息以添加到 `config.json` 中非常有用，且不会影响正在运行的容器。

#### 示例：使用 docker 下载数据

从 Binance 交易所下载 ETH/BTC 交易对、1 小时时间框架的 5 天回测数据。数据将存储在宿主机上的 `user_data/data/` 目录中。

``` bash
docker compose run --rm freqtrade download-data --pairs ETH/BTC --exchange binance --days 5 -t 1h
```

前往 [数据下载文档](data-download.md) 获取更多关于数据下载的详细信息。

#### 示例：使用 docker 进行回测

在 docker 容器中运行 SampleStrategy 的回测，使用指定的历史数据时间范围，时间框架为 5 分钟：

``` bash
docker compose run --rm freqtrade backtesting --config user_data/config.json --strategy SampleStrategy --timerange 20190801-20191001 -i 5m
```

前往 [回测文档](backtesting.md) 了解更多信息。

### Docker 的额外依赖项

如果您的策略需要默认镜像中未包含的依赖项，则需要在主机上构建镜像。
为此，请创建一个包含额外依赖项安装步骤的 Dockerfile（可参考 [docker/Dockerfile.custom](https://github.com/freqtrade/freqtrade/blob/develop/docker/Dockerfile.custom) 示例）。

随后您还需要修改 `docker-compose.yml` 文件，取消构建步骤的注释，并重命名镜像以避免命名冲突。

``` yaml
    image: freqtrade_custom
    build:
      context: .
      dockerfile: "./Dockerfile.<yourextension>"
```

接着可以运行 `docker compose build --pull` 构建 Docker 镜像，并使用上述命令运行。

### Docker 环境下的绘图功能

通过将 `docker-compose.yml` 文件中的镜像改为 `*_plot`，即可使用 `freqtrade plot-profit` 和 `freqtrade plot-dataframe` 命令（[文档](plotting.md)）。
具体使用方式如下：

``` bash
docker compose run --rm freqtrade plot-dataframe --strategy AwesomeStrategy -p BTC/ETH --timerange=20180801-20180805
```

输出结果将保存在 `user_data/plot` 目录中，可通过任何现代浏览器打开查看。

### 使用 Docker Compose 进行数据分析

Freqtrade 提供了可启动 jupyter lab 服务器的 docker-compose 文件。
通过以下命令即可运行该服务器：

``` bash
docker compose -f docker/docker-compose-jupyter.yml up
```

这将创建一个运行 jupyter lab 的 docker 容器，可通过 `https://127.0.0.1:8888/lab` 访问。
请使用启动后控制台打印的链接进行简化登录。

由于此镜像部分是在您的机器上构建的，建议定期重新构建镜像以保持 freqtrade（及其依赖项）处于最新状态。

``` bash
docker compose -f docker/docker-compose-jupyter.yml build --no-cache
```

## 故障排除

### Windows 上的 Docker

* 错误：`"Timestamp for this request is outside of the recvWindow."`  
  市场 API 请求需要时钟同步，但 Docker 容器中的时间会随时间推移略有偏差。
  要临时解决此问题，需要运行 `wsl --shutdown` 并重新启动 Docker（Windows 10 会弹出提示要求您执行此操作）。
  永久解决方案是在 Linux 主机上托管 Docker 容器，或通过计划任务定期重启 wsl。

  ``` bash
  taskkill /IM "Docker Desktop.exe" /F
  wsl --shutdown
  start "" "C:\Program Files\Docker\Docker\Docker Desktop.exe"
  ```

* 无法连接到 API（Windows）  
  如果您在 Windows 上且刚安装 Docker Desktop，请确保重启系统。Docker 在未重启的情况下可能出现网络连接问题。
  当然，您还应确保已相应配置[设置](#accessing-the-ui)。

!!! Warning
    鉴于上述原因，我们不建议在 Windows 上使用 Docker 进行生产环境部署，仅适用于实验、数据下载和回测。
    最佳实践是使用 Linux VPS 来可靠地运行 freqtrade。