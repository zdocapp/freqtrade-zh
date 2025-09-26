![freqtrade](assets/freqtrade_poweredby.svg)

[![Freqtrade CI](https://github.com/freqtrade/freqtrade/actions/workflows/ci.yml/badge.svg?branch=develop)](https://github.com/freqtrade/freqtrade/actions/)
[![DOI](https://joss.theoj.org/papers/10.21105/joss.04864/status.svg)](https://doi.org/10.21105/joss.04864)
[![Coverage Status](https://coveralls.io/repos/github/freqtrade/freqtrade/badge.svg?branch=develop&service=github)](https://coveralls.io/github/freqtrade/freqtrade?branch=develop)

<!-- GitHub action buttons -->

[:octicons-star-16: 点赞](https://github.com/freqtrade/freqtrade){ .md-button .md-button--sm }
[:octicons-repo-forked-16: 复刻](https://github.com/freqtrade/freqtrade/fork){ .md-button .md-button--sm }
[:octicons-download-16: 下载](https://github.com/freqtrade/freqtrade/archive/stable.zip){ .md-button .md-button--sm }

## 项目说明

Freqtrade 是一个用 Python 编写的免费开源加密货币交易机器人。它旨在支持所有主要交易所，并可通过 Telegram 或 Web 界面进行控制。它包含回测、绘图和资金管理工具，以及通过机器学习进行策略优化。

!!! Danger "免责声明"
    本软件仅供教育用途。请勿投入您无法承受损失的资金。使用本软件风险自负。作者及其所有关联方对您的交易结果不承担任何责任。

    Always start by running a trading bot in Dry-run and do not engage money before you understand how it works and what profit/loss you should expect.

    We strongly recommend you to have basic coding skills and Python knowledge. Do not hesitate to read the source code and understand the mechanisms of this bot, algorithms and techniques implemented in it.

![freqtrade 截图](assets/freqtrade-screenshot.png)

## 功能特性

- 制定策略：使用 [pandas](https://pandas.pydata.org/) 编写您的 Python 策略。[策略库](https://github.com/freqtrade/freqtrade-strategies) 中提供了可供参考的示例策略。
- 下载市场数据：下载您可能想要交易的交易所和市场的历史数据。
- 回测：在下载的历史数据上测试您的策略。
- 优化：使用采用机器学习方法的超参数优化，为您的策略找到最佳参数。您可以优化策略的买入、卖出、止盈（ROI）、止损和追踪止损参数。
- 选择市场：创建您的静态列表，或基于最高交易量和/或价格使用自动列表（回测期间不可用）。您还可以明确将不想交易的市场加入黑名单。
- 运行：使用模拟资金（干运行模式）测试您的策略，或使用真实资金（实盘交易模式）部署它。
- 控制/监控：使用 Telegram 或 WebUI（启动/停止机器人、显示盈亏、每日摘要、当前未平仓交易结果等）。
- 分析：可以对回测数据或 Freqtrade 交易历史（SQL 数据库）进行进一步分析，包括自动生成标准图表，以及将数据加载到[交互式环境](data-analysis.md)中的方法。

## 支持的交易所市场

请阅读[交易所特定说明](exchanges.md)，了解每个交易所可能需要的特殊配置。

- [X] [Binance](https://www.binance.com/)
- [X] [BingX](https://bingx.com/invite/0EM9RX)
- [X] [Bitmart](https://bitmart.com/)
- [X] [Bybit](https://bybit.com/)
- [X] [Gate.io](https://www.gate.io/ref/6266643)
- [X] [HTX](https://www.htx.com/)
- [X] [Hyperliquid](https://hyperliquid.xyz/) (去中心化交易所，简称 DEX)
- [X] [Kraken](https://kraken.com/)
- [X] [OKX](https://okx.com/)
- [X] [MyOKX](https://okx.com/) (OKX EEA)
- [ ] [可能通过 <img alt="ccxt" width="30px" src="assets/ccxt-logo.svg" /> 支持许多其他交易所](https://github.com/ccxt/ccxt/)。_(我们无法保证它们能正常工作)_

### 支持的期货交易所（实验性）

- [X] [Binance](https://www.binance.com/)
- [X] [Bybit](https://bybit.com/)
- [X] [Gate.io](https://www.gate.io/ref/6266643)
- [X] [Hyperliquid](https://hyperliquid.xyz/) (去中心化交易所，简称 DEX)
- [X] [OKX](https://okx.com/)

在开始之前，请务必阅读[交易所特定说明](exchanges.md)以及[杠杆交易](leverage.md)文档。

### 社区已验证

经社区确认可正常工作的交易所：

- [X] [Bitvavo](https://bitvavo.com/)
- [X] [Kucoin](https://www.kucoin.com/)

## 社区展示

--8<-- "includes/showcase.md"

## 系统要求

### 硬件要求

运行此机器人我们建议您使用 Linux 云实例，最低配置为：

- 2GB 内存
- 1GB 磁盘空间
- 2vCPU

### 软件要求

- Docker (推荐)

或者

- Python 3.11+
- pip (pip3)
- git
- TA-Lib
- virtualenv (推荐)

## 支持

### 帮助 / Discord

对于文档未涵盖的任何问题，或想获取有关机器人的更多信息，或仅仅想与志同道合的人交流，我们鼓励您加入 Freqtrade 的 [discord 服务器](https://discord.gg/p7nuUNVfP7)。

## 准备尝试？

请先阅读 [Docker 快速入门指南](docker_quickstart.md)（推荐），或 [无 Docker 安装指南](installation.md)。