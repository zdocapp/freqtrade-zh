# 安装

本页说明如何准备运行机器人的环境。

freqtrade 文档介绍了多种安装 freqtrade 的方式

* [Docker 镜像](docker_quickstart.md)（单独页面）
* [脚本安装](#script-installation)
* [手动安装](#manual-installation)
* [使用 Conda 安装](#installation-with-conda)

在评估 freqtrade 工作原理时，建议使用预构建的 [docker 镜像](docker_quickstart.md)来快速入门。

------

## 信息说明

Windows 系统安装请参考 [Windows 安装指南](windows_installation.md)。

安装和运行 Freqtrade 最简单的方法是克隆机器人 Github 仓库，然后运行 `./setup.sh` 脚本（如果该脚本适用于您的平台）。

!!! Note "版本注意事项"
    克隆仓库时，默认工作分支名为 `develop`。该分支包含所有最新功能（由于自动化测试，可视为相对稳定）。
    `stable` 分支包含最新发布的代码（通常每月发布一次，基于约一周前的 `develop` 分支快照，以防止打包错误，因此可能更稳定）。

!!! Note
    假定已安装 Python 3.11 或更高版本及对应的 `pip`。若未满足条件，安装脚本将发出警告并停止运行。同时需要 `git` 来克隆 Freqtrade 代码库。  
    此外，必须安装 Python 头文件（`python<你的版本号>-dev` / `python<你的版本号>-devel`）才能顺利完成安装。

!!! Warning "系统时钟同步"
    运行机器人的系统时钟必须准确，需频繁与 NTP 服务器同步以避免与交易所通信出现问题。

------

## 系统要求

这些要求同时适用于[脚本安装](#script-installation)和[手动安装](#manual-installation)。

!!! Note "ARM64 系统"
    若您运行的是 ARM64 系统（如 MacOS M1 或 Oracle VM），请使用 [docker](docker_quickstart.md) 运行 freqtrade。
    虽然通过手动操作可实现原生安装，但目前暂不支持该方式。

### 安装指南

* [Python >= 3.11](http://docs.python-guide.org/en/latest/starting/installation/)
* [pip](https://pip.pypa.io/en/stable/installing/)
* [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
* [virtualenv](https://virtualenv.pypa.io/en/stable/installation.html)（推荐）

### 安装代码

我们已收录针对 Ubuntu、MacOS 和 Windows 的安装说明。这些仅为指导方针，在其他发行版上的成功率可能有所不同。
操作系统特定步骤列于首位，下文通用章节是所有系统必需的。

!!! Note
    假定已安装 Python 3.11 或更高版本及对应的 pip。

=== "Debian/Ubuntu"
    #### 安装必要依赖项

    ```bash
    # update repository
    sudo apt-get update

    # install packages
    sudo apt install -y python3-pip python3-venv python3-dev python3-pandas git curl
    ```

=== "MacOS"
    #### 安装必要依赖项

    Install [Homebrew](https://brew.sh/) if you don't have it already.

    ```bash
    # install packages
    brew install gettext libomp
    ```
    !!! Note
        The `setup.sh` script will install these dependencies for you - assuming brew is installed on your system.

=== "RaspberryPi/Raspbian"
    以下假设使用最新的 [Raspbian Buster 精简版镜像](https://www.raspberrypi.org/downloads/raspbian/)。
    该镜像预装了 python3.11，使得 freqtrade 能够轻松安装运行。

    Tested using a Raspberry Pi 3 with the Raspbian Buster lite image, all updates applied.


    ```bash
    sudo apt-get install python3-venv libatlas-base-dev cmake curl libffi-dev
    # Use piwheels.org to speed up installation
    sudo echo "[global]\nextra-index-url=https://www.piwheels.org/simple" > tee /etc/pip.conf

    git clone https://github.com/freqtrade/freqtrade.git
    cd freqtrade

    bash setup.sh -i
    ```

    !!! Note "Installation duration"
        Depending on your internet speed and the Raspberry Pi version, installation can take multiple hours to complete.
        Due to this, we recommend to use the pre-build docker-image for Raspberry, by following the [Docker quickstart documentation](docker_quickstart.md)

    !!! Note
        The above does not install hyperopt dependencies. To install these, please use `python3 -m pip install -e .[hyperopt]`.
        We do not advise to run hyperopt on a Raspberry Pi, since this is a very resource-heavy operation, which should be done on powerful machine.

------

## Freqtrade 代码库

Freqtrade 是一个开源加密货币交易机器人，其代码托管在 `github.com`

```bash
# Download `develop` branch of freqtrade repository
git clone https://github.com/freqtrade/freqtrade.git

# Enter downloaded directory
cd freqtrade

# your choice (1): novice user
git checkout stable

# your choice (2): advanced user
git checkout develop
```

(1) 此命令将克隆的代码库切换到使用 `stable` 分支。如果您希望停留在 (2) `develop` 分支，则无需执行此操作。

您之后可以随时使用 `git checkout stable`/`git checkout develop` 命令在分支之间切换。

??? Note "从 pypi 安装"
    安装 Freqtrade 的另一种方式是从 [pypi](https://pypi.org/project/freqtrade/) 安装。这种方法的缺点是需要事先正确安装 ta-lib，因此目前不是推荐的 Freqtrade 安装方式。

    ``` bash
    pip install freqtrade
    ```

------

## 脚本安装

安装 Freqtrade 的首选方法是使用提供的 Linux/MacOS `./setup.sh` 脚本，该脚本将安装所有依赖项并帮助您配置机器人。

请确保您已满足 [要求](#requirements) 并下载了 [Freqtrade 代码库](#freqtrade-repository)。

### 使用 /setup.sh -install (Linux/MacOS)

如果您使用的是 Debian、Ubuntu 或 MacOS，freqtrade 提供了安装脚本。

```bash
# --install, Install freqtrade from scratch
./setup.sh -i
```

### 激活虚拟环境

每次打开新终端时，您必须运行 `source .venv/bin/activate` 来激活虚拟环境。

```bash
# activate virtual environment
source ./.venv/bin/activate
```

[您现在已准备就绪](#you-are-ready) 可以运行机器人了。

### /setup.sh 脚本的其他选项

您也可以使用 `./script.sh` 来更新、配置和重置机器人的代码库。

```bash
# --update, Command git pull to update.
./setup.sh -u
# --reset, Hard reset your develop/stable branch.
./setup.sh -r
```

```
** --install **

With this option, the script will install the bot and most dependencies:
You will need to have git and python3.11+ installed beforehand for this to work.

* Mandatory software as: `ta-lib`
* Setup your virtualenv under `.venv/`

This option is a combination of installation tasks and `--reset`

** --update **

This option will pull the last version of your current branch and update your virtualenv. Run the script with this option periodically to update your bot.

** --reset **

This option will hard reset your branch (only if you are on either `stable` or `develop`) and recreate your virtualenv.
```

-----

## 手动安装

请确保您已满足 [要求](#requirements) 并已下载 [Freqtrade 代码库](#freqtrade-repository)。

### 设置 Python 虚拟环境 (virtualenv)

您将在独立的 `虚拟环境` 中运行 freqtrade。

```bash
# create virtualenv in directory /freqtrade/.venv
python3 -m venv .venv

# run virtualenv
source .venv/bin/activate
```

### 安装 Python 依赖

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
# install freqtrade
python3 -m pip install -e .
```

[您现在已准备就绪](#you-are-ready) 可以运行机器人了。

### （可选）安装后任务

!!! Note 
    如果您在服务器上运行机器人，应考虑使用 [Docker](docker_quickstart.md) 或终端多路复用器（如 `screen` 或 [`tmux`](https://en.wikipedia.org/wiki/Tmux)），以避免注销时机器人停止运行。

在使用 `systemd` 软件套件的 Linux 系统上，作为可选的安装后任务，您可能希望将机器人设置为 `systemd 服务` 运行，或将其配置为将日志消息发送到 `syslog`/`rsyslog` 或 `journald` 守护进程。有关详细信息，请参阅 [高级日志记录](advanced-setup.md#advanced-logging)。

------

## 使用 Conda 安装

Freqtrade 也可以使用 Miniconda 或 Anaconda 进行安装。我们推荐使用 Miniconda，因为它的安装占用空间更小。Conda 会自动准备和管理 Freqtrade 程序庞大的库依赖。

### 什么是 Conda？

Conda 是一个支持多种编程语言的包、依赖和环境管理器：[conda 文档](https://docs.conda.io/projects/conda/en/latest/index.html)

### 使用 conda 安装

#### 安装 Conda

[在 Linux 上安装](https://conda.io/projects/conda/en/latest/user-guide/install/linux.html#install-linux-silent)

[在 Windows 上安装](https://conda.io/projects/conda/en/latest/user-guide/install/windows.html)

回答所有问题。安装后，必须关闭并重新打开终端。

#### 下载 Freqtrade

下载并安装 freqtrade。

```bash
# download freqtrade
git clone https://github.com/freqtrade/freqtrade.git

# enter downloaded directory 'freqtrade'
cd freqtrade      
```

#### Freqtrade 安装：Conda 环境

```bash
conda create --name freqtrade python=3.12
```

!!! Note "创建 Conda 环境"
    conda 命令 `create -n` 会自动为选定的库安装所有嵌套依赖项，安装命令的一般结构为：

    ```bash
    # choose your own packages
    conda env create -n [name of the environment] [python version] [packages]
    ```

#### 进入/退出 freqtrade 环境

要检查可用环境，请输入

```bash
conda env list
```

进入已安装的环境

```bash
# enter conda environment
conda activate freqtrade

# exit conda environment - don't do it now
conda deactivate
```

使用 pip 安装最新的 python 依赖项

```bash
python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
python3 -m pip install -e .
```

[您现在已准备就绪](#you-are-ready)，可以运行机器人了。

### 重要快捷键

```bash
# list installed conda environments
conda env list

# activate base environment
conda activate

# activate freqtrade environment
conda activate freqtrade

#deactivate any conda environments
conda deactivate                              
```

### 关于 anaconda 的更多信息

!!! Info "新的大型软件包"
    可能会出现这样的情况：创建一个新的 Conda 环境，并在创建时安装选定的软件包，比在先前设置的环境中安装一个大型、资源密集的库或应用程序所需的时间更短。

!!! Warning "在 conda 环境中使用 pip"
    conda 的文档指出，不应在 conda 环境中使用 pip，因为可能会发生内部问题。
    然而，这种情况很少见。[Anaconda 博客文章](https://www.anaconda.com/blog/using-pip-in-a-conda-environment)

    Nevertheless, that is why, the `conda-forge` channel is preferred:

    * more libraries are available (less need for `pip`)
    * `conda-forge` works better with `pip`
    * the libraries are newer

祝交易愉快！

-----

## 准备就绪

您已经成功走到这一步，说明您已成功安装了 freqtrade。

### 初始化配置

```bash
# Step 1 - Initialize user folder
freqtrade create-userdir --userdir user_data

# Step 2 - Create a new configuration file
freqtrade new-config --config user_data/config.json
```

您已准备就绪可以运行，请阅读 [机器人配置](configuration.md)，记得从 `dry_run: True` 开始，并验证一切是否正常工作。

要了解如何设置配置，请参阅 [机器人配置](configuration.md) 文档页面。

### 启动机器人

```bash
freqtrade trade --config user_data/config.json --strategy SampleStrategy
```

!!! Warning
    在启用真实货币交易之前，您应该通读其余文档，对将要使用的策略进行回测，并使用模拟交易模式。

-----

## 故障排除

### 常见问题："command not found"

如果您使用 (1)`脚本` 或 (2)`手动` 安装方式，则需要在虚拟环境中运行机器人。如果出现如下错误，请确保虚拟环境已激活。

```bash
# if:
bash: freqtrade: command not found

# then activate your virtual environment
source ./.venv/bin/activate
```

### MacOS 安装错误

较新版本的 MacOS 在安装时可能会失败，并出现类似 `error: command 'g++' failed with exit status 1` 的错误。

此错误需要显式安装 SDK 头文件，这些文件在该版本的 MacOS 中默认未安装。
对于 MacOS 10.14，可以通过以下命令完成安装。

```bash
open /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg
```

如果该文件不存在，那么您可能使用的是其他版本的 MacOS，因此可能需要查阅互联网以获取具体的解决方案细节。