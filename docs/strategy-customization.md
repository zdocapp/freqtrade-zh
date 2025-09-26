# 策略自定义

本页说明如何自定义策略、添加新指标并设置交易规则。

如果您尚未熟悉，请先了解：

- [Freqtrade 策略入门指南](strategy-101.md)，提供策略开发的快速入门
- [Freqtrade 机器人基础](bot-basics.md)，提供机器人运行方式的整体信息

## 开发自己的策略

机器人包含一个默认策略文件。

此外，[策略库](https://github.com/freqtrade/freqtrade-strategies)中提供了其他几种策略。

但您很可能会有自己的策略想法。

本文档旨在帮助您将想法转化为可运行的策略。

### 生成策略模板

要开始使用，您可以运行以下命令：

```bash
freqtrade new-strategy --strategy AwesomeStrategy
```

这将从模板创建一个名为 `AwesomeStrategy` 的新策略，该策略将使用文件名 `user_data/strategies/AwesomeStrategy.py` 保存。

!!! Note
    策略的*名称*与文件名存在区别。在大多数命令中，Freqtrade 使用策略的*名称*，*而非文件名*。

!!! Note
    `new-strategy` 命令生成的初始示例在开箱即用时不会立即产生盈利。

??? Hint "不同的模板层级"
    `freqtrade new-strategy` 有一个额外参数 `--template`，用于控制创建策略时预置信息的多少。使用 `--template minimal` 可获得不含任何指标示例的空策略，或使用 `--template advanced` 获得定义了更复杂功能的模板。

### 策略结构解析

策略文件包含构建策略逻辑所需的全部信息：

- OHLCV 格式的 K 线数据
- 指标
- 入场逻辑
  - 信号
- 离场逻辑
  - 信号
  - 最小 ROI
  - 回调函数（"自定义功能"）
- 止损设置
  - 固定/绝对值
  - 追踪止损
  - 回调函数（"自定义功能"）
- 定价 [可选]
- 仓位调整 [可选]

机器人内置了一个名为 `SampleStrategy` 的示例策略，您可将其作为基础模板：`user_data/strategies/sample_strategy.py`。
您可以使用参数进行测试：`--strategy SampleStrategy`。请注意此处使用的是策略类名，而非文件名。

此外，还有一个名为 `INTERFACE_VERSION` 的属性，用于定义机器人应使用的策略接口版本。
当前版本为 3——当策略中未显式设置时，该版本也是默认值。

您可能会看到设置为接口版本 2 的旧策略，这些策略需要更新至 v3 术语体系，因为未来版本将要求必须设置此属性。

使用 `trade` 命令可以以模拟或实盘模式启动机器人：

```bash
freqtrade trade --strategy AwesomeStrategy
```

### 机器人模式

Freqtrade 策略可以通过 Freqtrade 机器人以 5 种主要模式进行处理：

- 回测
- 超参数优化
- 模拟（"前向测试"）
- 实盘
- FreqAI（此处不涉及）

关于如何将机器人设置为模拟或实盘模式，请查阅[配置文档](configuration.md)。

**测试时请务必使用模拟模式，这能让您了解策略在实际中的表现，同时避免资金风险。**

## 深入探索

**在接下来的章节中，我们将以 [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py) 文件作为参考。**

!!! Note "策略与回测"
    为避免回测与模拟/实盘模式之间出现问题和意外差异，请注意在回测期间，完整的时间范围会一次性传递给 `populate_*()` 方法。
    因此最佳实践是使用向量化操作（针对整个数据框，而非循环）并避免索引引用（`df.iloc[-1]`），而应使用 `df.shift()` 来获取前一根K线数据。

!!! Warning "Warning: Using future data"
    由于回测将完整时间范围传递给 `populate_*()` 方法，策略开发者需注意避免策略使用未来数据。
    本文档的[常见错误](#common-mistakes-when-developing-strategies)章节列出了一些常见模式。

??? Hint "Lookahead and recursive analysis"
    Freqtrade 包含两个实用命令来帮助评估常见的前视偏差（使用未来数据）和递归偏差（指标值方差）问题。
    在模拟或实盘运行策略前，应始终先使用这些命令。请查阅相关文档了解[前视分析](lookahead-analysis.md)和[递归分析](recursive-analysis.md)。

### 数据框

Freqtrade 使用 [pandas](https://pandas.pydata.org/) 存储/提供K线（OHLCV）数据。
Pandas 是为处理表格形式大量数据而开发的优秀库。

数据框中的每一行对应图表上的一根K线，最新完成的K线始终位于数据框末尾（按日期排序）。

若使用 pandas 的 `head()` 函数查看主数据框的前几行，我们将看到：

```output
> dataframe.head()
                       date      open      high       low     close     volume
0 2021-11-09 23:25:00+00:00  67279.67  67321.84  67255.01  67300.97   44.62253
1 2021-11-09 23:30:00+00:00  67300.97  67301.34  67183.03  67187.01   61.38076
2 2021-11-09 23:35:00+00:00  67187.02  67187.02  67031.93  67123.81  113.42728
3 2021-11-09 23:40:00+00:00  67123.80  67222.40  67080.33  67160.48   78.96008
4 2021-11-09 23:45:00+00:00  67160.48  67160.48  66901.26  66943.37  111.39292
```

数据框是一种表格，其列不是单个值，而是一系列数据值。因此，简单的 Python 比较如下所示将无法正常工作：

``` python
    if dataframe['rsi'] > 30:
        dataframe['enter_long'] = 1
```

上述部分将因 `Series 的真值不明确 [...]` 而失败。

必须改用与 pandas 兼容的方式编写，以便在整个数据框上执行操作，即 `向量化`。

``` python
    dataframe.loc[
        (dataframe['rsi'] > 30)
    , 'enter_long'] = 1
```

通过此部分，您的数据框中将新增一列，当 RSI 高于 30 时该列会被赋值为 `1`。

Freqtrade 将此新列用作入场信号，并假设交易将在下一个开盘蜡烛上随后开启。

Pandas 提供了计算指标的快速方法，即"向量化"。为受益于此速度优势，建议不要使用循环，而应使用向量化方法。

向量化操作在整个数据范围内执行计算，因此在计算指标时，与逐行循环相比速度要快得多。

??? 提示 "信号与交易"
    - 信号由指标在蜡烛收盘时生成，代表入场交易的意图。
    - 交易是已执行的订单（实盘模式下在交易所执行），交易将尽可能在下一个蜡烛开盘时开启。

!!! 警告 "交易订单假设"
    在回测中，信号在蜡烛收盘时生成。交易随后在下一个蜡烛开盘时立即启动。

    In dry and live, this may be delayed due to all pair dataframes needing to be analysed first, then trade processing 
    for each of those pairs happens. This means that in dry/live you need to be mindful of having as low a computation 
    delay as possible, usually by running a low number of pairs and having a CPU with a good clock speed.

#### 为什么我看不到"实时"蜡烛数据？

Freqtrade 不会在数据框中存储不完整/未结束的蜡烛数据。

使用不完整数据制定策略决策的行为被称为"重绘"，您可能会在其他平台上看到允许此类操作的情况。

Freqtrade 不允许这样做。数据帧中仅提供完整/已完成的K线数据。

### 自定义指标

入场和出场信号需要指标支持。您可以通过扩展策略文件中 `populate_indicators()` 方法包含的列表来添加更多指标。

您应该仅添加在 `populate_entry_trend()`、`populate_exit_trend()` 中使用的指标，或用于填充其他指标的指标，否则可能会影响性能。

务必始终从这三个函数返回数据帧，且不要删除/修改 `"open"`、`"high"`、`"low"`、`"close"`、`"volume"` 这些列，否则这些字段将包含意外内容。

示例：

```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Adds several different TA indicators to the given DataFrame

    Performance Note: For the best performance be frugal on the number of indicators
    you are using. Let uncomment only the indicator you are using in your strategies
    or your hyperopt configuration, otherwise you will waste your memory and CPU usage.
    :param dataframe: Dataframe with data from the exchange
    :param metadata: Additional information, like the currently traded pair
    :return: a Dataframe with all mandatory indicators for the strategies
    """
    dataframe['sar'] = ta.SAR(dataframe)
    dataframe['adx'] = ta.ADX(dataframe)
    stoch = ta.STOCHF(dataframe)
    dataframe['fastd'] = stoch['fastd']
    dataframe['fastk'] = stoch['fastk']
    dataframe['bb_lower'] = ta.BBANDS(dataframe, nbdevup=2, nbdevdn=2)['lowerband']
    dataframe['sma'] = ta.SMA(dataframe, timeperiod=40)
    dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)
    dataframe['mfi'] = ta.MFI(dataframe)
    dataframe['rsi'] = ta.RSI(dataframe)
    dataframe['ema5'] = ta.EMA(dataframe, timeperiod=5)
    dataframe['ema10'] = ta.EMA(dataframe, timeperiod=10)
    dataframe['ema50'] = ta.EMA(dataframe, timeperiod=50)
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
    dataframe['ao'] = awesome_oscillator(dataframe)
    macd = ta.MACD(dataframe)
    dataframe['macd'] = macd['macd']
    dataframe['macdsignal'] = macd['macdsignal']
    dataframe['macdhist'] = macd['macdhist']
    hilbert = ta.HT_SINE(dataframe)
    dataframe['htsine'] = hilbert['sine']
    dataframe['htleadsine'] = hilbert['leadsine']
    dataframe['plus_dm'] = ta.PLUS_DM(dataframe)
    dataframe['plus_di'] = ta.PLUS_DI(dataframe)
    dataframe['minus_dm'] = ta.MINUS_DM(dataframe)
    dataframe['minus_di'] = ta.MINUS_DI(dataframe)

    # remember to always return the dataframe
    return dataframe
```

!!! Note "需要更多指标示例？"
    请查看 [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py)。
    然后取消注释您需要的指标。

#### 指标库

Freqtrade 默认安装以下技术分析库：

- [ta-lib](https://ta-lib.github.io/ta-lib-python/)
- [pandas-ta](https://twopirllc.github.io/pandas-ta/)
- [technical](https://technical.freqtrade.io)

可根据需要安装额外的技术库，或者策略作者可以编写/自定义指标。

### 策略启动期

某些指标存在不稳定的启动期，此时没有足够的K线数据来计算任何值（NaN），或者计算不正确。这可能导致不一致性，因为Freqtrade不知道这个不稳定期的时长，会直接使用数据框中现有的指标值。

为解决这个问题，可以为策略设置 `startup_candle_count` 属性。

该值应设置为策略计算稳定指标所需的最大K线数量。当用户包含带信息对的高时间框架时，`startup_candle_count` 不一定需要改变。该值是所有信息时间框架计算稳定指标所需的最大周期（以K线数为单位）。

您可以使用[递归分析](recursive-analysis.md)来检查并找到正确的 `startup_candle_count` 值。当递归分析显示方差为0%时，您可以确信已获得足够的启动期K线数据。

在此示例策略中，该值应设置为400（`startup_candle_count = 400`），因为确保ema100计算值正确所需的最小历史数据是400根K线。

``` python
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
```

通过让机器人知道需要多少历史数据，回测交易可以在回测和超参数优化期间从指定的时间范围开始。

!!! Warning "使用 x 次调用获取 OHLCV 数据"
    如果收到类似 `WARNING - 使用 3 次调用获取 OHLCV 数据。这可能导致机器人操作变慢。请检查您的策略是否真的需要 1500 根 K 线` 的警告 - 您应该考虑是否真的需要这么多历史数据来生成信号。
    设置过大的值将导致 Freqtrade 对同一交易对进行多次调用，这显然会比单次网络请求更慢。
    因此，Freqtrade 刷新 K 线数据的时间会变长 - 所以应尽可能避免这种情况。
    为避免对交易所造成过载或使 freqtrade 运行过慢，总调用次数被限制在 5 次以内。

!!! Warning
    `startup_candle_count` 应低于 `ohlcv_candle_limit * 5`（对于大多数交易所是 500 * 5） - 因为在模拟交易/实盘交易操作期间只有这个数量的 K 线数据可用。

#### 示例

让我们尝试使用上面提到的带有 EMA100 的示例策略，回测 1 个月（2019 年 1 月）的 5 分钟 K 线数据。

``` bash
freqtrade backtesting --timerange 20190101-20190201 --timeframe 5m
```

假设 `startup_candle_count` 设置为 400，回测系统知道需要 400 根 K 线来生成有效的入场信号。它将加载从 `20190101 - (400 * 5m)` 开始的数据 - 即约 2018-12-30 11:40:00。

如果该数据可用，指标将使用这个扩展的时间范围进行计算。在开始回测之前，不稳定的启动周期（截至 2019-01-01 00:00:00）将被移除。

!!! Note "启动蜡烛数据不可用"
    如果启动周期的数据不可用，时间范围将被调整以考虑这个启动周期。在我们的示例中，回测将从 2019-01-02 09:20:00 开始。

### 入场信号规则

编辑策略文件中的 `populate_entry_trend()` 方法来更新您的入场策略。

务必始终返回未删除/修改 `"open"`、`"high"`、`"low"`、`"close"`、`"volume"` 列的数据框，否则这些字段可能包含意外内容。策略可能会产生无效值，或完全停止工作。

此方法还将定义一个新列 `"enter_long"`（对于空头策略为 `"enter_short"`），该列需要为入场信号包含 `1`，为"无操作"包含 `0`。`enter_long` 是一个必须设置的列，即使策略仅做空也是如此。

您可以使用 `"enter_tag"` 列为入场信号命名，这有助于后续调试和评估策略。

来自 `user_data/strategies/sample_strategy.py` 的示例：

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Based on TA indicators, populates the buy signal for the given dataframe
    :param dataframe: DataFrame populated with indicators
    :param metadata: Additional information, like the currently traded pair
    :return: DataFrame with buy column
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

!!! Note "Enter short trades"
    可以通过设置 `enter_short`（对应多头交易的 `enter_long`）来创建空头入场信号。
    `enter_tag` 列保持不变。
    做空需要您的交易所和市场配置支持！
    此外，如果您打算做空，请确保在策略中正确设置 [`can_short`](#can-short)。

    ```python
    # allow both long and short trades
    can_short = True

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
                (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

        dataframe.loc[
            (
                (qtpylib.crossed_below(dataframe['rsi'], 70)) &  # Signal: RSI crosses below 70
                (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['enter_short', 'enter_tag']] = (1, 'rsi_cross')

        return dataframe
    ```

!!! Note
    买入需要有卖方才能成交。因此，成交量必须大于 0（`dataframe['volume'] > 0`），以确保机器人在无交易活动期间不会进行买入/卖出操作。

### 出场信号规则

编辑策略文件中的 `populate_exit_trend()` 方法来更新您的出场策略。

可以通过在配置或策略中将 `use_exit_signal` 设置为 false 来抑制出场信号。

`use_exit_signal` 不会影响[信号冲突规则](#colliding-signals) - 这些规则仍然适用，并可能阻止入场。

务必始终返回不删除/修改 `"open"`、`"high"`、`"low"`、`"close"`、`"volume"` 列的数据框，否则这些字段将包含意外内容。策略可能会产生无效值，或完全停止工作。

此方法还将定义一个新列 `"exit_long"`（空头对应 `"exit_short"`），出场时需要包含 `1`，无操作时包含 `0`。

您可以使用 `"exit_tag"` 列来命名您的退出信号，这有助于后续调试和评估策略。

示例来自 `user_data/strategies/sample_strategy.py`：

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Based on TA indicators, populates the exit signal for the given dataframe
    :param dataframe: DataFrame populated with indicators
    :param metadata: Additional information, like the currently traded pair
    :return: DataFrame with buy column
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
    return dataframe
```

??? Note "做空交易退出"
    可以通过设置 `exit_short`（对应 `exit_long`）来创建做空退出。
    `exit_tag` 列保持不变。
    做空需要您的交易所和市场配置支持！
    此外，如果您打算做空，请确保在策略中正确设置 [`can_short`](#can-short)。

    ```python
    # allow both long and short trades
    can_short = True

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
                (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
        dataframe.loc[
            (
                (qtpylib.crossed_below(dataframe['rsi'], 30)) &  # Signal: RSI crosses below 30
                (dataframe['tema'] < dataframe['bb_middleband']) &  # Guard
                (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
                (dataframe['volume'] > 0)  # Make sure Volume is not 0
            ),
            ['exit_short', 'exit_tag']] = (1, 'rsi_too_low')
        return dataframe
    ```

### 最小投资回报率

`minimal_roi` 策略变量定义了交易在退出前应达到的最小投资回报率（ROI），与退出信号无关。

其格式如下，即一个 Python `dict`，字典键（冒号左侧）为交易开仓后经过的分钟数，值（冒号右侧）为百分比。

```python
minimal_roi = {
    "40": 0.0,
    "30": 0.01,
    "20": 0.02,
    "0": 0.04
}
```

因此，上述配置意味着：

- 当达到 4% 利润时退出
- 当达到 2% 利润时退出（20 分钟后生效）
- 当达到 1% 利润时退出（30 分钟后生效）
- 当交易不亏损时退出（40 分钟后生效）

计算包含手续费。

#### 禁用最小投资回报率

要完全禁用 ROI，请将其设置为空字典：

```python
minimal_roi = {}
```

#### 在最小投资回报率中使用计算

要使用基于蜡烛时长（时间框架）的时间，以下代码片段会很有用。

这将允许您更改策略的时间框架，但最小投资回报率时间仍将设置为蜡烛数，例如3根蜡烛后。

``` python
from freqtrade.exchange import timeframe_to_minutes

class AwesomeStrategy(IStrategy):

    timeframe = "1d"
    timeframe_mins = timeframe_to_minutes(timeframe)
    minimal_roi = {
        "0": 0.05,                      # 5% for the first 3 candles
        str(timeframe_mins * 3): 0.02,  # 2% after 3 candles
        str(timeframe_mins * 6): 0.01,  # 1% After 6 candles
    }
```

??? info "未立即成交的订单"
    `minimal_roi` 将以 `trade.open_date` 作为参考时间，即交易初始化的时间，也就是该交易的首个订单被下达的时间。
    对于未立即成交的限价订单（通常与通过 `custom_entry_price()` 设置的"离场"价格结合使用），以及初始订单价格通过 `adjust_entry_price()` 被替换的情况，这也同样适用。
    使用的时间仍将是最初的 `trade.open_date`（即初始订单首次下达的时间），而非新下达或调整后的订单日期。

### 止损

强烈建议设置止损，以保护您的资金免受对您不利的剧烈波动影响。

设置10%止损的示例：

``` python
stoploss = -0.10
```

有关止损功能的完整文档，请查阅专门的[止损页面](stoploss.md)。

### 时间框架

这是机器人应在策略中使用的蜡烛周期。

常用值为 `"1m"`、`"5m"`、`"15m"`、`"1h"`，但您的交易所支持的所有值都应该有效。

请注意，相同的入场/出场信号可能在某一个时间框架下表现良好，但在其他时间框架下则不然。

此设置可在策略方法中作为 `self.timeframe` 属性访问。

### 允许做空

要在期货市场使用做空信号，您必须设置 `can_short = True`。

启用此功能的策略将无法在现货市场加载。

如果在 `enter_short` 列中有 `1` 值来触发做空信号，设置 `can_short = False`（默认值）将意味着这些做空信号被忽略，即使您在配置中指定了期货市场。

### 元数据字典

`metadata` 字典（可用于 `populate_entry_trend`、`populate_exit_trend`、`populate_indicators`）包含附加信息。
目前这是 `pair`，可以使用 `metadata['pair']` 访问，并将返回格式为 `XRP/BTC` 的交易对（对于期货市场为 `XRP/BTC:BTC`）。

元数据字典不应被修改，并且不会在策略的多个函数之间持久保存信息。

请查阅[存储信息](strategy-advanced.md#storing-information-persistent)章节。

--8<-- "includes/strategy-imports.md"

## 策略文件加载

默认情况下，freqtrade 将尝试从 `userdir`（默认为 `user_data/strategies`）内的所有 `.py` 文件加载策略。

假设您的策略名为 `AwesomeStrategy`，存储在文件 `user_data/strategies/AwesomeStrategy.py` 中，那么您可以通过以下方式以模拟（或实盘，取决于您的配置）模式启动 freqtrade：

```bash
freqtrade trade --strategy AwesomeStrategy
```

请注意，我们使用的是类名，而非文件名。

您可以使用 `freqtrade list-strategies` 查看 Freqtrade 能够加载的所有策略列表（正确文件夹中的所有策略）。
该列表还会包含一个“状态”字段，用于突出显示潜在问题。

??? Hint "自定义策略目录"
    您可以通过使用 `--strategy-path user_data/otherPath` 来指定不同的目录。此参数适用于所有需要策略的命令。

## 信息对

### 获取非交易对的数据

额外的信息对（参考对）数据对于某些策略查看更广泛时间框架的数据可能是有益的。

这些对的 OHLCV 数据将作为常规白名单刷新过程的一部分进行下载，并通过 `DataProvider` 提供，与其他对相同（见下文）。

这些对**不会**被交易，除非它们也被指定在配对白名单中，或已被动态白名单（例如 `VolumePairlist`）选中。

这些对需要以元组形式指定，格式为 `("pair", "timeframe")`，其中 pair 为第一个参数，timeframe 为第二个参数。

示例：

``` python
def informative_pairs(self):
    return [("ETH/USDT", "5m"),
            ("BTC/TUSD", "15m"),
            ]
```

完整示例可在 [DataProvider 部分](#complete-dataprovider-sample) 找到。

!!! Warning
    由于这些交易对将作为常规白名单刷新的一部分进行更新，最好保持此列表简短。
    只要使用的交易所提供（且活跃）的所有时间框架和所有交易对都可以指定。
    但尽可能使用较长时间框架的重采样会更佳，
    以避免向交易所发送过多请求导致被封锁的风险。

??? Note "替代 K 线类型"
    Informative_pairs 还可以提供第三个元组元素来明确定义 K 线类型。
    替代 K 线类型的可用性取决于交易模式和交易所。
    通常，现货交易对不能用于期货市场，而期货 K 线也不能作为现货机器人的信息对。
    具体细节可能有所不同，如有变化可在交易所文档中找到相关信息。

    ``` python
    def informative_pairs(self):
        return [
            ("ETH/USDT", "5m", ""),   # Uses default candletype, depends on trading_mode (recommended)
            ("ETH/USDT", "5m", "spot"),   # Forces usage of spot candles (only valid for bots running on spot markets).
            ("BTC/TUSD", "15m", "futures"),  # Uses futures candles (only bots with `trading_mode=futures`)
            ("BTC/TUSD", "15m", "mark"),  # Uses mark candles (only bots with `trading_mode=futures`)
        ]
    ```

***

### 信息对装饰器 (`@informative()`)

为方便定义信息对，可使用 `@informative` 装饰器。所有被装饰的 `populate_indicators_*` 方法都独立运行，
且无法访问其他信息对的数据。但每个交易对的所有信息数据框会被合并并传递给主 `populate_indicators()` 方法。

!!! Note
    如果需要在生成一个信息对时使用另一个信息对的数据，请不要使用 `@informative` 装饰器。而应按照 [DataProvider 章节](#complete-dataprovider-sample) 中所述手动定义信息对。

进行超参数优化时，不支持使用可优化参数的 `.value` 属性。请使用 `.range` 属性。更多信息请参阅 [优化指标参数](hyperopt.md#optimizing-an-indicator-parameter)。

??? info "完整文档"
    ``` python
    def informative(
        timeframe: str,
        asset: str = "",
        fmt: str | Callable[[Any], str] | None = None,
        *,
        candle_type: CandleType | str | None = None,
        ffill: bool = True,
    ) -> Callable[[PopulateIndicators], PopulateIndicators]:
        """
        用于 populate_indicators_Nn(self, dataframe, metadata) 的装饰器，允许这些函数定义信息指标。

        Example usage:

            @informative('1h')
            def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
                dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
                return dataframe

        :param timeframe: Informative timeframe. Must always be equal or higher than strategy timeframe.
        :param asset: Informative asset, for example BTC, BTC/USDT, ETH/BTC. Do not specify to use
                    current pair. Also supports limited pair format strings (see below)
        :param fmt: Column format (str) or column formatter (callable(name, asset, timeframe)). When not
        specified, defaults to:
        * {base}_{quote}_{column}_{timeframe} if asset is specified.
        * {column}_{timeframe} if asset is not specified.
        Pair format supports these format variables:
        * {base} - base currency in lower case, for example 'eth'.
        * {BASE} - same as {base}, except in upper case.
        * {quote} - quote currency in lower case, for example 'usdt'.
        * {QUOTE} - same as {quote}, except in upper case.
        Format string additionally supports this variables.
        * {asset} - full name of the asset, for example 'BTC/USDT'.
        * {column} - name of dataframe column.
        * {timeframe} - timeframe of informative dataframe.
        :param ffill: ffill dataframe after merging informative pair.
        :param candle_type: '', mark, index, premiumIndex, or funding_rate
        """
    ```

??? Example "定义信息对的快速简便方法"

    Most of the time we do not need power and flexibility offered by `merge_informative_pair()`, therefore we can use a decorator to quickly define informative pairs.

    ``` python

    from datetime import datetime
    from freqtrade.persistence import Trade
    from freqtrade.strategy import IStrategy, informative

    class AwesomeStrategy(IStrategy):
        
        # This method is not required. 
        # def informative_pairs(self): ...

        # Define informative upper timeframe for each pair. Decorators can be stacked on same 
        # method. Available in populate_indicators as 'rsi_30m' and 'rsi_1h'.
        @informative('30m')
        @informative('1h')
        def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

        # Define BTC/STAKE informative pair. Available in populate_indicators and other methods as
        # 'btc_rsi_1h'. Current stake currency should be specified as {stake} format variable 
        # instead of hard-coding actual stake currency. Available in populate_indicators and other 
        # methods as 'btc_usdt_rsi_1h' (when stake currency is USDT).
        @informative('1h', 'BTC/{stake}')
        def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

        # Define BTC/ETH informative pair. You must specify quote currency if it is different from
        # stake currency. Available in populate_indicators and other methods as 'eth_btc_rsi_1h'.
        @informative('1h', 'ETH/BTC')
        def populate_indicators_eth_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe
    
        # Define BTC/STAKE informative pair. A custom formatter may be specified for formatting
        # column names. A callable `fmt(**kwargs) -> str` may be specified, to implement custom
        # formatting. Available in populate_indicators and other methods as 'rsi_upper_1h'.
        @informative('1h', 'BTC/{stake}', '{column}_{timeframe}')
        def populate_indicators_btc_1h_2(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi_upper'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe
    
        def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            # Strategy timeframe indicators for current pair.
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            # Informative pairs are available in this method.
            dataframe['rsi_less'] = dataframe['rsi'] < dataframe['rsi_1h']
            return dataframe

    ```

!!! Note
    访问其他交易对的信息数据框时请使用字符串格式化。这样可以在不调整策略代码的情况下轻松更改配置中的计价货币。

    ``` python
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        stake = self.config['stake_currency']
        dataframe.loc[
            (
                (dataframe[f'btc_{stake}_rsi_1h'] < 35)
                &
                (dataframe['volume'] > 0)
            ),
            ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')
    
        return dataframe
    ```

    Alternatively column renaming may be used to remove stake currency from column names: `@informative('1h', 'BTC/{stake}', fmt='{base}_{column}_{timeframe}')`.

!!! Warning "重复的方法名称"
    使用 `@informative()` 装饰器标记的方法必须始终具有唯一名称！重复使用相同名称（例如复制粘贴已定义的信息方法时）将覆盖先前定义的方法，并且由于 Python 编程语言的限制不会产生任何错误。在这种情况下，您会发现策略文件中较早创建的方法指标在数据框中不可用。请仔细检查方法名称并确保它们是唯一的！

### *merge_informative_pair()*

此方法帮助您安全且一致地将信息对合并到常规主数据框中，避免前视偏差。

选项：

- 重命名列以创建唯一列
- 无前视偏差地合并数据框
- 前向填充（可选）

完整示例请参考下方的[完整数据提供者示例](#complete-dataprovider-sample)。

信息数据框的所有列将以重命名的方式在返回的数据框中可用：

!!! Example "列重命名"
    假设 `inf_tf = '1d'`，生成的列将为：

    ``` python
    'date', 'open', 'high', 'low', 'close', 'rsi'                     # from the original dataframe
    'date_1d', 'open_1d', 'high_1d', 'low_1d', 'close_1d', 'rsi_1d'   # from the informative dataframe
    ```

??? Example "列重命名 - 1小时"
    假设 `inf_tf = '1h'`，生成的列将为：

    ``` python
    'date', 'open', 'high', 'low', 'close', 'rsi'                     # from the original dataframe
    'date_1h', 'open_1h', 'high_1h', 'low_1h', 'close_1h', 'rsi_1h'   # from the informative dataframe
    ```

??? Example "自定义实现"
    可以对此进行自定义实现，操作如下：

    ``` python

    # Shift date by 1 candle
    # This is necessary since the data is always the "open date"
    # and a 15m candle starting at 12:15 should not know the close of the 1h candle from 12:00 to 13:00
    minutes = timeframe_to_minutes(inf_tf)
    # Only do this if the timeframes are different:
    informative['date_merge'] = informative["date"] + pd.to_timedelta(minutes, 'm')

    # Rename columns to be unique
    informative.columns = [f"{col}_{inf_tf}" for col in informative.columns]
    # Assuming inf_tf = '1d' - then the columns will now be:
    # date_1d, open_1d, high_1d, low_1d, close_1d, rsi_1d

    # Combine the 2 dataframes
    # all indicators on the informative sample MUST be calculated before this point
    dataframe = pd.merge(dataframe, informative, left_on='date', right_on=f'date_merge_{inf_tf}', how='left')
    # FFill to have the 1d value available in every row throughout the day.
    # Without this, comparisons would only work once per day.
    dataframe = dataframe.ffill()

    ```

!!! Warning "信息时间框架 < 时间框架"
    不建议在此方法中使用比主数据框架时间框架更小的信息时间框架，因为它不会利用这种设置可能提供的任何额外信息。
    要正确使用更详细的信息，应采用更高级的方法（这超出了本文档的范围）。

## 附加数据（DataProvider）

策略提供对 `DataProvider` 的访问权限。这允许您获取在策略中使用的附加数据。

所有方法在失败时都会返回 `None`，即失败不会引发异常。

请务必检查操作模式以选择正确的数据获取方法（参见以下示例）。

!!! Warning "Hyperopt 限制"
    DataProvider 在 hyperopt 期间可用，但只能在**策略内部**的 `populate_indicators()` 中使用，不能在 hyperopt 类文件中使用。
    它在 `populate_entry_trend()` 和 `populate_exit_trend()` 方法中也不可用。

### DataProvider 的可能选项

- [`available_pairs`](#available_pairs) - 包含缓存交易对及其时间框架元组列表的属性（交易对，时间框架）。
- [`current_whitelist()`](#current_whitelist) - 返回当前白名单交易对列表。适用于访问动态白名单（如 VolumePairlist）。
- [`get_pair_dataframe(pair, timeframe)`](#get_pair_dataframepair-timeframe) - 通用方法，返回历史数据（用于回测）或缓存的实时数据（用于模拟交易和实盘交易模式）。
- [`get_analyzed_dataframe(pair, timeframe)`](#get_analyzed_dataframepair-timeframe) - 返回已分析的数据框（在调用 `populate_indicators()`、`populate_buy()`、`populate_sell()` 之后）及最新分析时间。
- `historic_ohlcv(pair, timeframe)` - 返回存储在磁盘上的历史数据。
- `market(pair)` - 返回交易对的市场数据：费用、限制、精度、活跃标志等。有关市场数据结构的更多详情，请参阅 [ccxt 文档](https://github.com/ccxt/ccxt/wiki/Manual#markets)。
- `ohlcv(pair, timeframe)` - 返回交易对当前缓存的蜡烛图（OHLCV）数据，返回 DataFrame 或空 DataFrame。
- [`orderbook(pair, maximum)`](#orderbookpair-maximum) - 返回交易对的最新订单簿数据，包含总共 `maximum` 个条目的买入/卖出字典。
- [`ticker(pair)`](#tickerpair) - 返回交易对的当前行情数据。有关 Ticker 数据结构的更多详情，请参阅 [ccxt 文档](https://github.com/ccxt/ccxt/wiki/Manual#price-tickers)。
- [`check_delisting(pair)`](#check_delistingpair) - 如果存在交易对下架计划，则返回其日期时间，否则返回 None。
- [`funding_rate(pair)`](#funding_ratepair) - 返回交易对的当前资金费率数据。
- `runmode` - 包含当前运行模式的属性。

### 示例用法

### *available_pairs*

``` python
for pair, timeframe in self.dp.available_pairs:
    print(f"available {pair}, {timeframe}")
```

### *current_whitelist()*

假设您开发了一个策略，该策略使用交易量排名前10的交易对在`1d`时间框架上生成的信号，在`5m`时间框架上进行交易。

策略逻辑可能如下所示：

*每5分钟使用`VolumePairList`扫描交易量前10的交易对，并使用14日RSI指标进行入场和出场。*

由于可用数据有限，将`5m`K线重采样为日线以用于14日RSI计算非常困难。大多数交易所将用户限制在仅500-1000根K线，这实际上只给我们提供了大约1.74根日线。而我们至少需要14天的数据！

既然无法重采样数据，我们将不得不使用信息交易对，而且由于白名单是动态的，我们不知道要使用哪个（些）交易对！我们遇到了问题！

这时调用`self.dp.current_whitelist()`就派上用场了，它可以仅检索白名单中的那些交易对。

```python
    def informative_pairs(self):

        # get access to all pairs available in whitelist.
        pairs = self.dp.current_whitelist()
        # Assign timeframe to each pair so they can be downloaded and cached for strategy.
        informative_pairs = [(pair, '1d') for pair in pairs]
        return informative_pairs
```

!!! Note "使用current_whitelist绘图"
    `plot-dataframe`不支持当前白名单功能，因为该命令通常通过提供显式交易对列表来使用，因此会使此方法的返回值产生误导。
    在[网络服务器模式](utils.md#webserver-mode)下的FreqUI可视化中也不支持该功能，因为网络服务器模式的配置不需要设置交易对列表。

### *get_pair_dataframe(pair, timeframe)*

``` python
# fetch live / historical candle (OHLCV) data for the first informative pair
inf_pair, inf_timeframe = self.informative_pairs()[0]
informative = self.dp.get_pair_dataframe(pair=inf_pair,
                                         timeframe=inf_timeframe)
```

!!! Warning "关于回测的警告"
    在回测中，`dp.get_pair_dataframe()` 的行为会根据调用位置而有所不同。
    在 `populate_*()` 方法内部，`dp.get_pair_dataframe()` 返回完整的时间范围。请确保不要"预见未来"，以避免在干运行/实盘模式下出现意外情况。
    在[回调函数](strategy-callbacks.md)内部，您将获得截至当前（模拟）蜡烛线的完整时间范围。

### *get_analyzed_dataframe(pair, timeframe)*

此方法由 freqtrade 内部用于确定最后的信号。
它也可以在特定的回调函数中使用，以获取触发操作的信号（有关可用回调的更多详细信息，请参阅[高级策略文档](strategy-advanced.md)）。

``` python
# fetch current dataframe
dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=metadata['pair'],
                                                         timeframe=self.timeframe)
```

!!! Note "无可用数据"
    如果请求的交易对未被缓存，则返回一个空的数据框。
    您可以使用 `if dataframe.empty:` 来检查这种情况并进行相应处理。
    在使用白名单交易对时，这种情况不应发生。

### *orderbook(pair, maximum)*

获取指定交易对的当前订单簿。

``` python
if self.dp.runmode.value in ('live', 'dry_run'):
    ob = self.dp.orderbook(metadata['pair'], 1)
    dataframe['best_bid'] = ob['bids'][0][0]
    dataframe['best_ask'] = ob['asks'][0][0]
```

订单簿结构与 [ccxt](https://github.com/ccxt/ccxt/wiki/Manual#order-book-structure) 的订单结构保持一致，因此结果将按以下格式呈现：

``` js
{
    'bids': [
        [ price, amount ], // [ float, float ]
        [ price, amount ],
        ...
    ],
    'asks': [
        [ price, amount ],
        [ price, amount ],
        //...
    ],
    //...
}
```

因此，如上所示使用 `ob['bids'][0][0]` 将使用最佳买价。`ob['bids'][0][1]` 将查看该订单簿位置的数量。

!!! Warning "关于回测的警告"
    订单簿不是历史数据的一部分，这意味着如果使用此方法，回测和超参数优化将无法正常工作，因为该方法将返回最新的值。

### *ticker(pair)*

``` python
if self.dp.runmode.value in ('live', 'dry_run'):
    ticker = self.dp.ticker(metadata['pair'])
    dataframe['last_price'] = ticker['last']
    dataframe['volume24h'] = ticker['quoteVolume']
    dataframe['vwap'] = ticker['vwap']
```

!!! Warning
    尽管行情数据结构是 ccxt 统一接口的一部分，但此方法返回的值可能因不同交易所而异。例如，许多交易所不返回 `vwap` 值，有些交易所并不总是填写 `last` 字段（因此它可能为 None），等等。因此，您需要仔细验证从交易所返回的行情数据，并添加适当的错误处理/默认值。

!!! Warning "关于回测的警告"
    此方法将始终返回最新的/实时值。因此，在回测/超参数优化期间使用而不进行运行模式检查将导致错误的结果，例如，您的整个数据框将在所有行中包含相同的单个值。

### *check_delisting(pair)*

```python
def custom_exit(self, pair: str, trade: Trade, current_time: datetime, current_rate: float, current_profit: float, **kwargs):
    if self.dp.runmode.value in ('live', 'dry_run'):
        delisting_dt = self.dp.check_delisting(pair)
        if delisting_dt is not None:
            return "delist"
```

!!! Note "退市信息的可用性"
    此方法仅适用于某些交易所，并且在不可用或该交易对未计划退市的情况下将返回 `None`。

!!! Warning "关于回测的警告"
    此方法将始终返回最新/实时值。因此，在回测/超参数优化期间使用而不进行运行模式检查将导致错误结果，例如您的整个数据框将在所有行中包含相同的单个值。

### *funding_rate(pair)*

获取交易对的当前资金费率，仅适用于格式为 `base/quote:settle` 的永续合约交易对（例如 `ETH/USDT:USDT`）。

``` python
if self.dp.runmode.value in ('live', 'dry_run'):
    funding_rate = self.dp.funding_rate(metadata['pair'])
    dataframe['current_funding_rate'] = funding_rate['fundingRate']
    dataframe['next_funding_timestamp'] = funding_rate['fundingTimestamp']
    dataframe['next_funding_datetime'] = funding_rate['fundingDatetime']
```

资金费率结构与 [ccxt](https://github.com/ccxt/ccxt/wiki/Manual#funding-rate-structure) 的资金费率结构保持一致，因此结果将按以下格式返回：

``` python
{
    "info": {
        # ... 
    },
    "symbol": "BTC/USDT:USDT",
    "markPrice": 110730.7,
    "indexPrice": 110782.52,
    "interestRate": 0.0001,
    "estimatedSettlePrice": 110822.67200153,
    "timestamp": 1757146321001,
    "datetime": "2025-09-06T08:12:01.001Z",
    "fundingRate": 5.609e-05,
    "fundingTimestamp": 1757174400000,
    "fundingDatetime": "2025-09-06T16:00:00.000Z",
    "nextFundingRate": None,
    "nextFundingTimestamp": None,
    "nextFundingDatetime": None,
    "previousFundingRate": None,
    "previousFundingTimestamp": None,
    "previousFundingDatetime": None,
    "interval": None,
}
```

因此，如上所示使用 `funding_rate['fundingRate']` 将使用当前资金费率。
实际可用数据因交易所而异，因此此代码在不同交易所之间可能无法按预期工作。

!!! Warning "关于回测的警告"
    当前资金费率不是历史数据的一部分，这意味着如果使用此方法，回测和超参数优化将无法正确工作，因为该方法将返回最新值。
    我们建议使用历史上可用的资金费率进行回测（该数据会自动下载，频率与交易所提供的频率一致，通常为4小时或8小时）。
    `self.dp.get_pair_dataframe(pair=metadata['pair'], timeframe='8h', candle_type="funding_rate")`

### 发送通知

数据提供者的 `.send_msg()` 函数允许您从策略发送自定义通知。
相同的通知在每个蜡烛图周期内只会发送一次，除非第二个参数 (`always_send`) 设置为 True。

``` python
    self.dp.send_msg(f"{metadata['pair']} just got hot!")

    # Force send this notification, avoid caching (Please read warning below!)
    self.dp.send_msg(f"{metadata['pair']} just got hot!", always_send=True)
```

通知仅在交易模式（实盘/模拟运行）下发送 - 因此该方法可以在无需考虑回测条件的情况下调用。

!!! Warning "消息轰炸"
    通过在此方法中设置 `always_send=True`，您可能会遭受严重的消息轰炸。请极其谨慎地使用此功能，并仅在您确认整个蜡烛图周期内不会重复触发的条件下使用，以避免每5秒收到一条消息。

### 完整的数据提供者示例

```python
from freqtrade.strategy import IStrategy, merge_informative_pair
from pandas import DataFrame

class SampleStrategy(IStrategy):
    # strategy init stuff...

    timeframe = '5m'

    # more strategy init stuff..

    def informative_pairs(self):

        # get access to all pairs available in whitelist.
        pairs = self.dp.current_whitelist()
        # Assign tf to each pair so they can be downloaded and cached for strategy.
        informative_pairs = [(pair, '1d') for pair in pairs]
        # Optionally Add additional "static" pairs
        informative_pairs += [("ETH/USDT", "5m"),
                              ("BTC/TUSD", "15m"),
                            ]
        return informative_pairs

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        if not self.dp:
            # Don't do anything if DataProvider is not available.
            return dataframe

        inf_tf = '1d'
        # Get the informative pair
        informative = self.dp.get_pair_dataframe(pair=metadata['pair'], timeframe=inf_tf)
        # Get the 14 day rsi
        informative['rsi'] = ta.RSI(informative, timeperiod=14)

        # Use the helper function merge_informative_pair to safely merge the pair
        # Automatically renames the columns and merges a shorter timeframe dataframe and a longer timeframe informative pair
        # use ffill to have the 1d value available in every row throughout the day.
        # Without this, comparisons between columns of the original and the informative pair would only work once per day.
        # Full documentation of this method, see below
        dataframe = merge_informative_pair(dataframe, informative, self.timeframe, inf_tf, ffill=True)

        # Calculate rsi of the original dataframe (5m timeframe)
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        # Do other stuff
        # ...

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
                (dataframe['rsi_1d'] < 30) &                     # Ensure daily RSI is < 30
                (dataframe['volume'] > 0)                        # Ensure this candle had volume (important for backtesting)
            ),
            ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

```

***

## 附加数据（钱包）

策略提供对 `wallets` 对象的访问。该对象包含您在交易所的钱包/账户当前余额。

!!! Note "回测 / 超参数优化"
    钱包的行为根据调用它的函数而有所不同。
    在 `populate_*()` 方法内，它将返回配置的完整钱包。
    在[回调函数](strategy-callbacks.md)内，您将获得模拟过程中对应时间点的实际模拟钱包状态。

请始终检查 `wallets` 是否可用，以避免在回测期间出现故障。

``` python
if self.wallets:
    free_eth = self.wallets.get_free('ETH')
    used_eth = self.wallets.get_used('ETH')
    total_eth = self.wallets.get_total('ETH')
```

### 钱包的可能选项

- `get_free(asset)` - 当前可用于交易的余额
- `get_used(asset)` - 当前被占用的余额（未成交订单）
- `get_total(asset)` - 总可用余额 - 上述两项之和

***

## 附加数据（交易记录）

策略中可以通过查询数据库获取交易历史记录。

在文件顶部导入所需对象：

```python
from freqtrade.persistence import Trade
```

以下示例查询当前交易对（`metadata['pair']`）今日的交易记录。其他筛选条件可以轻松添加。

``` python
trades = Trade.get_trades_proxy(pair=metadata['pair'],
                                open_date=datetime.now(timezone.utc) - timedelta(days=1),
                                is_open=False,
            ]).order_by(Trade.close_date).all()
# Summarize profit for this pair.
curdayprofit = sum(trade.close_profit for trade in trades)
```

完整可用方法列表请查阅[交易对象](trade-object.md)文档。

!!! Warning
    在回测或超参优化期间，`populate_*` 方法中无法获取交易历史记录，查询将返回空结果。

## 阻止特定交易对的交易

当交易对平仓时，Freqtrade 会自动锁定当前K线周期内的该交易对（直到该K线结束），防止立即重新入场。

此举旨在避免单根K线内出现大量频繁交易的"瀑布式"操作。

被锁定的交易对将显示消息 `交易对 <pair> 当前已被锁定。`。

### 在策略中锁定交易对

有时可能需要在特定事件发生后锁定交易对（例如连续多次亏损交易）。

Freqtrade 提供了一种在策略中轻松实现此功能的方法，只需调用 `self.lock_pair(pair, until, [reason])`。
`until` 必须是一个未来的 datetime 对象，在此时间之后该交易对的交易将重新启用；而 `reason` 是一个可选字符串，用于详细说明锁定该交易对的原因。

锁定也可以手动解除，通过调用 `self.unlock_pair(pair)` 或 `self.unlock_reason(<reason>)` 并提供解锁原因。
`self.unlock_reason(<reason>)` 将解锁当前所有使用指定原因锁定的交易对。

要验证某个交易对当前是否被锁定，请使用 `self.is_pair_locked(pair)`。

!!! Note
    被锁定的交易对将始终向上取整到下一个蜡烛图。因此，假设使用 `5m` 时间框架，将 `until` 设置为 10:18 的锁定会将交易对锁定到 10:15-10:20 的蜡烛图结束。

!!! Warning
    手动锁定交易对在回测期间不可用。仅允许通过保护机制（Protections）进行锁定。

#### 交易对锁定示例

``` python
from freqtrade.persistence import Trade
from datetime import timedelta, datetime, timezone
# Put the above lines at the top of the strategy file, next to all the other imports
# --------

# Within populate indicators (or populate_entry_trend):
if self.config['runmode'].value in ('live', 'dry_run'):
    # fetch closed trades for the last 2 days
    trades = Trade.get_trades_proxy(
        pair=metadata['pair'], is_open=False, 
        open_date=datetime.now(timezone.utc) - timedelta(days=2))
    # Analyze the conditions you'd like to lock the pair .... will probably be different for every strategy
    sumprofit = sum(trade.close_profit for trade in trades)
    if sumprofit < 0:
        # Lock pair for 12 hours
        self.lock_pair(metadata['pair'], until=datetime.now(timezone.utc) + timedelta(hours=12))
```

## 打印主数据框

要检查当前的主数据框，您可以在 `populate_entry_trend()` 或 `populate_exit_trend()` 中添加打印语句。
您可能还想打印交易对信息，以便清楚显示当前展示的是哪些数据。

``` python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            #>> whatever condition<<<
        ),
        ['enter_long', 'enter_tag']] = (1, 'somestring')

    # Print the Analyzed pair
    print(f"result for {metadata['pair']}")

    # Inspect the last 5 rows
    print(dataframe.tail())

    return dataframe
```

通过使用 `print(dataframe)` 而非 `print(dataframe.tail())`，也可以打印多行数据。但不推荐这样做，因为这可能导致大量输出（每个交易对每 5 秒约 500 行）。

## 开发策略时的常见错误

### 回测时窥探未来数据

出于性能考虑，回测会一次性分析整个数据框的时间范围。因此，策略作者需要确保策略不会窥探未来数据，即使用在模拟或实盘模式下不可用的数据。

这是一个常见的痛点，可能导致回测与模拟/实盘运行方法之间产生巨大差异。窥探未来数据的策略在回测期间表现良好，通常会有惊人的利润或胜率，但在实际条件下会失败或表现不佳。

以下列表包含了一些应避免的常见模式，以防止挫败感：

- 不要在回测中使用 `shift(-1)` 或其他负值。这会在回测中使用未来的数据，而这些数据在模拟或实盘模式下是不可用的。
- 不要在 `populate_` 函数中使用 `.iloc[-1]` 或任何其他在数据框中的绝对位置，因为这在模拟运行和回测之间会有所不同。然而，在回调函数中使用绝对的 `iloc` 索引是安全的 - 请参阅 [策略回调](strategy-callbacks.md)。
- 不要使用涉及所有数据框或列值的函数，例如 `dataframe['mean_volume'] = dataframe['volume'].mean()`。由于回测使用完整的数据框，在数据框的任何一点，`'mean_volume'` 序列都将包含未来的数据。请改用滚动计算，例如 `dataframe['volume'].rolling(<窗口>).mean()`。
- 不要使用 `.resample('1h')`。这会使用周期间隔的左边界，从而将小时边界的数据移动到小时的开始。请改用 `.resample('1h', label='right')`。
- 不要使用 `.merge()` 将较长时间框架的数据合并到较短时间框架上。请改用 [信息对](#informative-pairs) 辅助工具。（普通的合并可能会隐式导致前视偏差，因为日期指的是开盘日期，而不是收盘日期）。

!!! Tip "识别问题"
    您应始终使用两个辅助命令 [lookahead-analysis](lookahead-analysis.md) 和 [recursive-analysis](recursive-analysis.md)，它们能以不同方式帮助您发现策略中的问题。
    请将它们视为识别最常见问题的辅助工具。每个命令的阴性结果并不能保证不存在上述错误。

### 信号冲突

当冲突信号同时出现时（例如 `'enter_long'` 和 `'exit_long'` 同时设置为 `1`），freqtrade 将不执行任何操作并忽略入场信号。这将避免交易入场后立即离场的情况。显然，这可能导致错过入场机会。

以下规则适用，如果3个信号中有超过一个被设置，入场信号将被忽略：

- `enter_long` -> `exit_long`, `enter_short`
- `enter_short` -> `exit_short`, `enter_long`

## 更多策略思路

要获取更多策略思路，请访问 [策略库](https://github.com/freqtrade/freqtrade-strategies)。可将其作为示例使用，但实际效果取决于当前市场状况、使用的交易对等因素。因此，这些策略应仅用于学习目的，不应用于实盘交易。请先针对您的交易所/目标交易对进行策略回测，然后通过模拟交易仔细评估，使用时风险自负。

欢迎使用它们作为您自己策略的灵感来源。我们很乐意接受包含新策略的 Pull Request 到代码库中。

## 后续步骤

现在您已经有了一个完美的策略，可能想要进行回测。
下一步是学习[如何使用回测功能](backtesting.md)。