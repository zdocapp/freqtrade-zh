# Freqtrade 策略入门：策略开发快速指南

本快速入门指南假设您已熟悉交易基础知识，并已阅读 [Freqtrade 基础](bot-basics.md) 页面。

## 必备知识

Freqtrade 中的策略是一个 Python 类，用于定义加密货币 `资产` 的买卖逻辑。

资产被定义为 `交易对`，代表 `代币` 和 `计价货币`。代币是您使用另一种货币作为计价货币进行交易的资产。

交易所以 `K线` 形式提供数据，每条 K线包含六个数值：`日期`、`开盘价`、`最高价`、`最低价`、`收盘价` 和 `成交量`。

`技术分析` 函数通过各类计算和统计公式分析 K线数据，并生成称为 `指标` 的次级数值。

指标通过分析资产交易对的 K线数据来生成 `信号`。

信号在加密货币 `交易所` 转化为 `订单`，即 `交易`。

我们使用 `入场` 和 `离场` 而非 `买入` 和 `卖出`，因为 Freqtrade 同时支持 `做多` 和 `做空` 交易。

- **做多（long）**：您基于抵押资产购买代币，例如使用 USDT 作为抵押资产购买 BTC 代币，通过以高于买入价的价格卖出代币来获利。在做多交易中，利润来自代币价值相对于抵押资产的上涨。
- **做空（short）**：您以代币形式从交易所借入资金，并在之后偿还代币的抵押资产价值。在做空交易中，利润来自代币价值相对于抵押资产的下跌（您以更低的价格偿还贷款）。

尽管 Freqtrade 支持某些交易所的现货和期货市场，但为简化说明，我们将仅关注现货（做多）交易。

## 基础策略结构

### 主数据框

Freqtrade 策略使用称为 `dataframe` 的行列式表格数据结构来生成交易入场和出场信号。

您配置的交易对列表中的每个交易对都有其独立的数据框。数据框通过 `date` 列进行索引，例如 `2024-06-31 12:00`。

紧随其后的 5 列分别代表 `开盘价`、`最高价`、`最低价`、`收盘价` 和 `成交量`（OHLCV）数据。

### 填充指标数值

`populate_indicators` 函数会向数据框添加代表技术分析指标数值的列。

常见指标示例包括相对强弱指数（RSI）、布林带（Bollinger Bands）、资金流量指数（MFI）、移动平均线（MA）和平均真实波幅（ATR）。

通过调用技术分析函数（例如 ta-lib 的 RSI 函数 `ta.RSI()`）并将其赋值给列名（例如 `rsi`），可以将列添加到数据框中。

```python
dataframe['rsi'] = ta.RSI(dataframe)
```

??? Hint "技术分析库"
    不同的库以不同的方式生成指标值。请查阅每个库的文档以了解如何将其集成到您的策略中。您也可以查看 [Freqtrade 示例策略](https://github.com/freqtrade/freqtrade-strategies) 来获取灵感。

### 填充入场信号

`populate_entry_trend` 函数定义了入场信号的条件。

数据框列 `enter_long` 被添加到数据框中，当该列中出现值 `1` 时，Freqtrade 会将其视为入场信号。

??? Hint "做空"
    要进入空头交易，请使用 `enter_short` 列。

### 填充离场信号

`populate_exit_trend` 函数定义了离场信号的条件。

数据框列 `exit_long` 被添加到数据框中，当该列中出现值 `1` 时，Freqtrade 会将其视为离场信号。

??? Hint "做空"
    要退出空头交易，请使用 `exit_short` 列。

## 一个简单的策略

以下是 Freqtrade 策略的一个最小示例：

```python
from freqtrade.strategy import IStrategy
from pandas import DataFrame
import talib.abstract as ta

class MyStrategy(IStrategy):

    timeframe = '15m'

    # set the initial stoploss to -10%
    stoploss = -0.10

    # exit profitable positions at any time when the profit is greater than 1%
    minimal_roi = {"0": 0.01}

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # generate values for technical analysis indicators
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # generate entry signals based on indicator values
        dataframe.loc[
            (dataframe['rsi'] < 30),
            'enter_long'] = 1

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # generate exit signals based on indicator values
        dataframe.loc[
            (dataframe['rsi'] > 70),
            'exit_long'] = 1

        return dataframe
```

## 进行交易

当发现信号（入场或离场列中出现 `1`）时，Freqtrade 将尝试下单，即进行一笔 `交易` 或建立 `头寸`。

每个新的交易头寸都会占用一个`槽位`。槽位代表可以同时开启的最大新交易数量。

槽位数量由`max_open_trades`[配置](configuration.md)选项定义。

但在某些情况下，生成信号并不总是会创建交易订单。这些情况包括：

- 没有足够的剩余资金购买资产，或钱包中的资金不足以出售资产（包括任何费用）
- 没有足够的剩余空闲槽位来开启新交易（已开启的头寸数量等于`max_open_trades`选项）
- 某个交易对已有未平仓交易（Freqtrade无法堆叠头寸 - 但可以[调整现有头寸](strategy-callbacks.md#adjust-trade-position)）
- 如果在同一根K线上同时出现入场和出场信号，它们被视为[冲突信号](strategy-customization.md#colliding-signals)，不会产生任何订单
- 策略通过使用相关的[入场](strategy-callbacks.md#trade-entry-buy-order-confirmation)或[出场](strategy-callbacks.md#trade-exit-sell-order-confirmation)回调函数中的逻辑主动拒绝交易订单

请阅读[策略定制](strategy-customization.md)文档以获取更多详细信息。

## 回测与正向测试

策略开发可能是一个漫长且令人沮丧的过程，因为将我们人类的"直觉"转化为可运行的计算机控制（"算法"）策略并非总是那么简单直接。

因此，应对策略进行测试以验证其是否能按预期工作。

Freqtrade 提供两种测试模式：

- **回测**：使用您[从交易所下载](data-download.md)的历史数据，回测是评估策略性能的快速方法。然而，很容易扭曲结果，使策略看起来比实际盈利性高得多。更多信息请查阅[回测文档](backtesting.md)。
- **模拟运行**：通常称为_前向测试_，模拟运行使用来自交易所的实时数据。然而，任何可能导致交易的信号都会被 Freqtrade 正常跟踪，但不会在交易所本身实际开仓。前向测试实时运行，因此虽然获得结果需要更长时间，但它是比回测更可靠的**潜在**性能指标。

通过在[配置](configuration.md#using-dry-run-mode)中将 `dry_run` 设置为 true 来启用模拟运行。

!!! Warning "回测结果可能极不准确"
    回测结果与现实不符的原因有很多。请查阅[回测假设](backtesting.md#assumptions-made-by-backtesting)和[策略开发常见错误](strategy-customization.md#common-mistakes-when-developing-strategies)文档。
    某些列出并排名Freqtrade策略的网站展示了令人印象深刻的回测结果。切勿认为这些结果是可以实现或符合实际的。

??? Hint "实用命令"
    Freqtrade包含两个用于检查策略基本缺陷的实用命令：[前瞻性分析](lookahead-analysis.md)和[递归分析](recursive-analysis.md)。

### 评估回测和模拟运行结果

完成策略回测后务必进行模拟运行，以确认回测与模拟运行结果是否足够接近。

若存在显著差异，请验证您的入场和出场信号是否一致，并在两种模式下出现在相同的K线上。但需注意，模拟运行与回测之间始终存在差异：

- 回测假设所有订单都能成交。在模拟运行中，如果使用限价单或交易所缺乏流动性，情况可能并非如此。
- 在K线收盘时跟随入场信号，回测假设交易会在下一根K线的开盘价入场（除非策略中使用了自定义价格回调）。而在模拟运行中，信号发出与交易开仓之间通常存在延迟。
  这是因为当主时间框架（例如每5分钟）的新K线数据传入时，Freqtrade需要时间分析所有交易对的数据框。因此，Freqtrade会尝试在K线开盘后几秒内（理想情况下延迟尽可能小）开仓交易。
- 由于模拟运行中的入场价格可能与回测不符，这意味着盈利计算也会存在差异。因此，投资回报率(ROI)、止损、移动止损和回调退出等指标不完全一致是正常现象。
- 新K线数据传入与信号产生、交易开仓之间的计算"延迟"越大，价格不可预测性就越高。请确保您的计算机性能足够强大，能在合理时间内处理交易对列表中的所有交易对数据。如果出现显著的数据处理延迟，Freqtrade会在日志中发出警告。

## 控制或监控运行中的机器人

当机器人在模拟或实盘模式下运行时，Freqtrade提供六种机制来控制或监控运行中的机器人：

- **[FreqUI](freq-ui.md)**: 最易上手的界面，FreqUI 是一个网页界面，用于查看和控制机器人的当前活动。
- **[Telegram](telegram-usage.md)**: 在移动设备上，可使用 Telegram 集成来接收机器人活动警报并控制某些方面。
- **[FTUI](https://github.com/freqtrade/ft-ui)**: FTUI 是 Freqtrade 的终端（命令行）界面，仅允许监控运行中的机器人。
- **[freqtrade-client](rest-api.md#consuming-the-api)**: REST API 的 Python 实现，便于从您的 Python 应用程序或命令行发出请求并处理机器人响应。
- **[REST API endpoints](rest-api.md#available-endpoints)**: REST API 允许程序员开发自己的工具与 Freqtrade 机器人交互。
- **[Webhooks](webhook-config.md)**: Freqtrade 可以通过 Webhook 向其他服务（例如 Discord）发送信息。

### 日志记录

Freqtrade 生成广泛的调试日志，以帮助您了解正在发生的情况。请熟悉您可能在机器人日志中看到的信息和错误消息。

默认情况下，日志记录发生在标准输出（命令行）上。如果您希望改为写入文件，许多 freqtrade 命令（包括 `trade` 命令）接受 `--logfile` 选项以写入文件。

查看 [FAQ](faq.md#how-do-i-search-the-bot-logs-for-something) 以获取示例。

## 最后说明

算法交易难度很高，由于需要投入大量时间和精力才能使策略在多种场景下实现盈利，大多数公开策略的表现并不理想。

因此，直接采用公开策略并依赖回测来评估性能往往存在问题。不过，Freqtrade 提供了实用工具来协助您做出决策并完成尽职调查。

实现盈利的途径多种多样，并不存在某个单一技巧或配置选项能够彻底扭转表现不佳的策略。

Freqtrade 是一个拥有庞大互助社区的开源平台——欢迎访问我们的 [Discord 频道](https://discord.gg/p7nuUNVfP7)与其他用户交流策略心得！

请始终谨记：只投入您愿意承担损失的资金。

## 结语

在 Freqtrade 中开发策略需要基于技术指标定义入场和离场信号。通过遵循上述框架和方法，您可以创建并测试自己的交易策略。

常见问题解答请参阅 [FAQ](faq.md) 文档。

如需深入探索，请参考更详细的 [Freqtrade 策略定制文档](strategy-customization.md)。