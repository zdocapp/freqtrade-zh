# ![freqtrade](https://raw.githubusercontent.com/freqtrade/freqtrade/develop/docs/assets/freqtrade_poweredby.svg)

[![Freqtrade CI](https://github.com/freqtrade/freqtrade/actions/workflows/ci.yml/badge.svg?branch=develop)](https://github.com/freqtrade/freqtrade/actions/)
[![DOI](https://joss.theoj.org/papers/10.21105/joss.04864/status.svg)](https://doi.org/10.21105/joss.04864)
[![Coverage Status](https://coveralls.io/repos/github/freqtrade/freqtrade/badge.svg?branch=develop&service=github)](https://coveralls.io/github/freqtrade/freqtrade?branch=develop)
[![Documentation](https://readthedocs.org/projects/freqtrade/badge/)](https://www.freqtrade.io)

> [!NOTE]
> 本仓库旨在提供 [https://www.freqtrade.io/en](https://www.freqtrade.io/en) 的中文版本，由 [zdoc.app](https://zdoc.app) 提供翻译。

Freqtrade 是一个用 Python 编写的免费开源加密货币交易机器人。它旨在支持所有主要交易所，并可通过 Telegram 或 Web 界面进行控制。它包含回测、绘图和资金管理工具，以及通过机器学习进行策略优化。

![freqtrade](https://raw.githubusercontent.com/freqtrade/freqtrade/develop/docs/assets/freqtrade-screenshot.png)

## 免责声明

本软件仅供教育目的使用。请勿投入您无法承受损失的资金。使用本软件的风险由您自行承担。作者及其所有关联方对您的交易结果不承担任何责任。

请始终先在模拟交易模式下运行交易机器人，在完全理解其工作原理及预期盈亏之前，切勿投入真实资金。

我们强烈建议您具备编程和 Python 知识。请随时阅读源代码以了解此机器人的运行机制。

## 支持的交易市场

请阅读[交易所特定说明](docs/exchanges.md)，了解每个交易所可能需要的特殊配置。

- [x] [Binance](https://www.binance.com/)
- [x] [Bitmart](https://bitmart.com/)
- [x] [BingX](https://bingx.com/invite/0EM9RX)
- [x] [Bybit](https://bybit.com/)
- [x] [Gate.io](https://www.gate.io/ref/6266643)
- [x] [HTX](https://www.htx.com/)
- [x] [Hyperliquid](https://hyperliquid.xyz/) (去中心化交易所，即 DEX)
- [x] [Kraken](https://kraken.com/)
- [x] [OKX](https://okx.com/)
- [x] [MyOKX](https://okx.com/) (OKX EEA)
- [ ] [可能还有许多其他交易所](https://github.com/ccxt/ccxt/)。_(我们不保证它们都能正常工作)_

### 支持的期货交易所（实验性）

- [x] [Binance](https://www.binance.com/)
- [x] [Gate.io](https://www.gate.io/ref/6266643)
- [x] [Hyperliquid](https://hyperliquid.xyz/) (去中心化交易所，即 DEX)
- [x] [OKX](https://okx.com/)
- [x] [Bybit](https://bybit.com/)

在开始之前，请务必阅读[交易所特定说明](docs/exchanges.md)以及[杠杆交易](docs/leverage.md)文档。

### 社区已验证

经社区确认可正常工作的交易所：

- [x] [Bitvavo](https://bitvavo.com/)
- [x] [Kucoin](https://www.kucoin.com/)

## 文档

我们建议您阅读机器人文档，以确保您了解机器人的工作原理。

完整文档请访问 [freqtrade 网站](https://www.freqtrade.io)。

## 功能特性

- [x] **基于 Python 3.11+**：可在任何操作系统上运行 - Windows、macOS 和 Linux。
- [x] **持久化**：通过 sqlite 实现持久化。
- [x] **模拟运行**：无需真实资金即可运行机器人。
- [x] **回测**：运行您的买入/卖出策略模拟。
- [x] **基于机器学习的策略优化**：使用机器学习结合真实交易所数据优化您的买入/卖出策略参数。
- [x] **自适应预测建模**：通过 FreqAI 构建智能策略，利用自适应机器学习方法进行市场自我训练。[了解更多](https://www.freqtrade.io/en/stable/freqai/)
- [x] **加密货币白名单**：选择您想要交易的加密货币或使用动态白名单。
- [x] **加密货币黑名单**：选择您希望避免的加密货币。
- [x] **内置 WebUI**：内置 Web 界面来管理您的机器人。
- [x] **可通过 Telegram 管理**：使用 Telegram 管理机器人。
- [x] **以法币显示盈亏**：以法币显示您的盈亏。
- [x] **性能状态报告**：提供您当前交易的性能状态。

## 快速入门

请参考 [Docker 快速入门文档](https://www.freqtrade.io/en/stable/docker_quickstart/) 了解如何快速开始。

如需了解更多（原生）安装方法，请参阅 [安装文档页面](https://www.freqtrade.io/en/stable/installation/)。

## 基础用法

### 机器人命令

```
usage: freqtrade [-h] [-V]
                 {trade,create-userdir,new-config,show-config,new-strategy,download-data,convert-data,convert-trade-data,trades-to-ohlcv,list-data,backtesting,backtesting-show,backtesting-analysis,edge,hyperopt,hyperopt-list,hyperopt-show,list-exchanges,list-markets,list-pairs,list-strategies,list-hyperoptloss,list-freqaimodels,list-timeframes,show-trades,test-pairlist,convert-db,install-ui,plot-dataframe,plot-profit,webserver,strategy-updater,lookahead-analysis,recursive-analysis}
                 ...

Free, open source crypto trading bot

positional arguments:
  {trade,create-userdir,new-config,show-config,new-strategy,download-data,convert-data,convert-trade-data,trades-to-ohlcv,list-data,backtesting,backtesting-show,backtesting-analysis,edge,hyperopt,hyperopt-list,hyperopt-show,list-exchanges,list-markets,list-pairs,list-strategies,list-hyperoptloss,list-freqaimodels,list-timeframes,show-trades,test-pairlist,convert-db,install-ui,plot-dataframe,plot-profit,webserver,strategy-updater,lookahead-analysis,recursive-analysis}
    trade               Trade module.
    create-userdir      Create user-data directory.
    new-config          Create new config
    show-config         Show resolved config
    new-strategy        Create new strategy
    download-data       Download backtesting data.
    convert-data        Convert candle (OHLCV) data from one format to
                        another.
    convert-trade-data  Convert trade data from one format to another.
    trades-to-ohlcv     Convert trade data to OHLCV data.
    list-data           List downloaded data.
    backtesting         Backtesting module.
    backtesting-show    Show past Backtest results
    backtesting-analysis
                        Backtest Analysis module.
    hyperopt            Hyperopt module.
    hyperopt-list       List Hyperopt results
    hyperopt-show       Show details of Hyperopt results
    list-exchanges      Print available exchanges.
    list-markets        Print markets on exchange.
    list-pairs          Print pairs on exchange.
    list-strategies     Print available strategies.
    list-hyperoptloss   Print available hyperopt loss functions.
    list-freqaimodels   Print available freqAI models.
    list-timeframes     Print available timeframes for the exchange.
    show-trades         Show trades.
    test-pairlist       Test your pairlist configuration.
    convert-db          Migrate database to different system
    install-ui          Install FreqUI
    plot-dataframe      Plot candles with indicators.
    plot-profit         Generate plot showing profits.
    webserver           Webserver module.
    strategy-updater    updates outdated strategy files to the current version
    lookahead-analysis  Check for potential look ahead bias.
    recursive-analysis  Check for potential recursive formula issue.

options:
  -h, --help            show this help message and exit
  -V, --version         show program's version number and exit
```

### Telegram RPC 命令

Telegram 不是强制要求。但这是控制机器人的绝佳方式。更多详情和完整命令列表请参阅 [文档](https://www.freqtrade.io/en/latest/telegram-usage/)。

- `/start`: 启动交易机器人。
- `/stop`: 停止交易机器人。
- `/stopentry`: 停止开立新交易。
- `/status <trade_id>|[table]`: 列出所有或特定未平仓交易。
- `/profit [<n>]`: 列出过去 n 天内所有已平仓交易的累计利润。
- `/profit_long [<n>]`: 列出过去 n 天内所有已平仓多头交易的累计利润。
- `/profit_short [<n>]`: 列出过去 n 天内所有已平仓空头交易的累计利润。
- `/forceexit <trade_id>|all`: 立即平仓指定交易（忽略 `minimum_roi`）。
- `/fx <trade_id>|all`: `/forceexit` 的别名
- `/performance`: 显示按交易对分组的已完成交易表现
- `/balance`: 显示各币种的账户余额。
- `/daily <n>`: 显示过去 n 天内每日盈亏情况。
- `/help`: 显示帮助信息。
- `/version`: 显示版本信息。

## 开发分支

该项目目前主要设置有两个分支：

- `develop` - 该分支通常包含新功能，但也可能包含破坏性变更。我们尽力保持该分支的稳定性。
- `stable` - 该分支包含最新的稳定版本。此分支通常经过充分测试。
- `feat/*` - 这些是功能分支，正在积极开发中。除非您想测试特定功能，否则请勿使用这些分支。

## 支持

### 帮助 / Discord

对于文档未涵盖的任何问题，或想获取有关机器人的更多信息，或仅仅想与志同道合的人交流，我们鼓励您加入 Freqtrade 的 [discord 服务器](https://discord.gg/p7nuUNVfP7)。

### [错误 / 问题](https://github.com/freqtrade/freqtrade/issues?q=is%3Aissue)

如果您在机器人中发现错误，请先
[搜索问题跟踪器](https://github.com/freqtrade/freqtrade/issues?q=is%3Aissue)。
如果尚未报告，请
[创建新问题](https://github.com/freqtrade/freqtrade/issues/new/choose)并
确保遵循模板指南，以便团队能够尽快为您提供帮助。

对于每个创建的[问题](https://github.com/freqtrade/freqtrade/issues/new/choose)，请及时跟进并在达成共识后标记满意度或提醒关闭问题。

--遵守 GitHub 的[社区政策](https://docs.github.com/en/site-policy/github-terms/github-community-code-of-conduct)--

### [功能请求](https://github.com/freqtrade/freqtrade/labels/enhancement)

您是否有改进机器人的好主意想要分享？请首先搜索该功能是否[已被讨论过](https://github.com/freqtrade/freqtrade/labels/enhancement)。如果尚未被提出，请[创建新请求](https://github.com/freqtrade/freqtrade/issues/new/choose)并确保遵循模板指南，以免其淹没在错误报告中。

### [拉取请求](https://github.com/freqtrade/freqtrade/pulls)

觉得机器人缺少某个功能？我们欢迎您的拉取请求！

请在提交拉取请求前阅读[贡献文档](https://github.com/freqtrade/freqtrade/blob/develop/CONTRIBUTING.md)了解相关要求。

贡献不一定需要编写代码——或许可以从改进文档开始？标记为[good first issue](https://github.com/freqtrade/freqtrade/labels/good%20first%20issue)的问题可以作为不错的首次贡献，并帮助您熟悉代码库。

**注意**：在开始任何重大新功能工作之前，_请先创建一个问题描述您的计划_，或在[discord](https://discord.gg/p7nuUNVfP7)上与我们交流（请使用 #dev 频道）。这将确保相关方能够对功能提供宝贵反馈，并让其他人知道您正在从事此项工作。

**重要提示：** 请始终针对 `develop` 分支而非 `stable` 分支创建您的 PR。

## 系统要求

### 保持时钟同步

系统时钟必须精确，并需频繁与 NTP 服务器同步，以避免与交易所的通信出现问题。

### 最低硬件要求

运行此机器人我们建议您使用至少满足以下配置的云实例：

- 最低（建议）系统要求：2GB 内存，1GB 磁盘空间，2vCPU

### 软件要求

- [Python >= 3.11](http://docs.python-guide.org/en/latest/starting/installation/)
- [pip](https://pip.pypa.io/en/stable/installing/)
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [TA-Lib](https://ta-lib.github.io/ta-lib-python/)
- [virtualenv](https://virtualenv.pypa.io/en/stable/installation.html)（推荐）
- [Docker](https://www.docker.com/products/docker)（推荐）
