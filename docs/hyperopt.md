# Hyperopt

本页介绍了如何通过寻找最优参数来调整您的策略，这一过程称为超参数优化。该机器人使用 `optuna` 包中包含的算法来实现此目的。
搜索将耗尽您所有的 CPU 核心，让您的笔记本电脑听起来像战斗机一样，并且仍然需要很长时间。

通常，最佳参数的搜索从一些随机组合开始（更多详情见[下文](#reproducible-results)），然后使用 optuna 的采样器算法之一（当前为 NSGAIIISampler）快速找到搜索超空间中最小化[损失函数](#loss-functions)值的参数组合。

Hyperopt 需要历史数据可用，就像回测一样（hyperopt 使用不同的参数多次运行回测）。
要了解如何获取您感兴趣的货币对和交易所的数据，请转到文档的[数据下载](data-download.md)部分。

!!! Bug
    如 [Issue #1133](https://github.com/freqtrade/freqtrade/issues/1133) 中发现，当仅使用 1 个 CPU 核心时，Hyperopt 可能会崩溃。

!!! Note
    自 2021.4 版本起，您不再需要编写单独的 hyperopt 类，而是可以直接在策略中配置参数。
    旧方法支持到 2021.8 版本，并在 2021.9 版本中已被移除。

## 安装 hyperopt 依赖

由于运行机器人本身不需要 Hyperopt 依赖项，且这些依赖项较为庞大，在某些平台（如树莓派）上不易构建，因此默认情况下不会安装。在运行 Hyperopt 之前，您需要按照以下章节所述安装相应的依赖项。

!!! Note
    由于 Hyperopt 是资源密集型进程，不建议也不支持在树莓派上运行。

### Docker 支持

Docker 镜像已包含 Hyperopt 依赖项，无需额外操作。

### 简易安装脚本 (setup.sh) / 手动安装

```bash
source .venv/bin/activate
pip install -r requirements-hyperopt.txt
```

## Hyperopt 命令参考

--8<-- "commands/hyperopt.md"

### Hyperopt 检查清单

Hyperopt 中所有任务/可能性的检查清单

根据您要优化的空间，仅需以下部分内容：

* 使用 `space='buy'` 定义参数 - 用于入场信号优化
* 使用 `space='sell'` 定义参数 - 用于出场信号优化

!!! Note
    `populate_indicators` 需要创建所有空间可能使用的指标，否则 Hyperopt 将无法工作。

极少数情况下，您可能还需要创建一个名为 `HyperOpt` 的[嵌套类](advanced-hyperopt.md#overriding-pre-defined-spaces)并实现

* `roi_space` - 用于自定义 ROI 优化（如果你需要优化超空间中 ROI 参数的范围与默认值不同）
* `generate_roi_table` - 用于自定义 ROI 优化（如果你需要 ROI 表中值的范围与默认值不同，或者 ROI 表中的条目数（步数）与默认的 4 步不同）
* `stoploss_space` - 用于自定义止损优化（如果你需要优化超空间中止损参数的范围与默认值不同）
* `trailing_space` - 用于自定义追踪止损优化（如果你需要优化超空间中追踪止损参数的范围与默认值不同）
* `max_open_trades_space` - 用于自定义最大开仓数优化（如果你需要优化超空间中最大开仓数参数的范围与默认值不同）

!!! Tip "快速优化 ROI、止损和追踪止损"
    你可以在不更改策略任何内容的情况下快速优化 `roi`、`stoploss` 和 `trailing` 空间。

    ``` bash
    # Have a working strategy at hand.
    freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --spaces roi stoploss trailing --strategy MyWorkingStrategy --config config.json -e 100
    ```

### 超参数优化执行逻辑

除非指定了 `--analyze-per-epoch` 参数，否则 Hyperopt 会首先将数据加载到内存中，然后对每个交易对运行一次 `populate_indicators()` 来生成所有指标。

Hyperopt 随后会生成多个进程（处理器数量，或 `-j <n>`），并反复运行回测，同时更改属于已定义 `--spaces` 部分的参数。

对于每一组新参数，freqtrade 将首先运行 `populate_entry_trend()`，接着运行 `populate_exit_trend()`，然后运行常规的回测流程来模拟交易。

回测结束后，结果会传递给 [损失函数](#loss-functions)，该函数将评估此结果是否优于或劣于之前的结果。  
根据损失函数的结果，hyperopt 将确定下一轮回测中要尝试的参数集。

### 配置你的守卫条件和触发条件

你需要在策略文件中修改两个位置来添加新的买入超参数优化测试：

* 在类级别定义 hyperopt 应优化的参数。
* 在 `populate_entry_trend()` 内部 - 使用已定义的参数值代替原始常量。

这里你有两种不同类型的指标：1. `guards`（守卫条件）和 2. `triggers`（触发条件）。

1. 守卫条件是诸如“当 ADX < 10 时绝不买入”，或“当前价格超过 EMA10 时绝不买入”之类的条件。
2. 触发条件则是那些在特定时刻实际触发买入的条件，例如“当 EMA5 上穿 EMA10 时买入”，或“当收盘价触及布林带下轨时买入”。

!!! Hint "Guards and Triggers"
    从技术上讲，Guards（守卫条件）和 Triggers（触发条件）并无区别。  
    但本指南将对此进行区分，以明确信号不应"粘连"。
    粘连信号是指持续多个K线周期的活跃信号。这可能导致延迟入场（恰好在信号消失前入场——此时成功概率远低于信号刚出现时）。

超参数优化将在每个训练轮次中选取一个触发条件，并可能选择多个守卫条件。

#### 出场信号优化

与上述入场信号类似，出场信号同样可以进行优化。
将相应配置填入以下方法：

* 在类级别定义超参数优化所需的参数，可将其命名为`sell_*`，或通过显式定义`space='sell'`实现。
* 在`populate_exit_trend()`方法内——使用已定义的参数值替代原始常量。

配置规则与买入信号相同。

## 破解谜题

假设您存在疑问：应该使用MACD金叉还是布林带下轨作为多头入场触发条件？
同时您还想知道：是否应该使用RSI或ADX来辅助决策？
如果决定使用RSI或ADX，又该为它们设置哪些参数值？

现在让我们运用超参数优化来解开这个谜题。

### 定义要使用的指标

我们首先计算策略将要使用的指标。

``` python
class MyAwesomeStrategy(IStrategy):

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """
        Generate all indicators used by the strategy
        """
        dataframe['adx'] = ta.ADX(dataframe)
        dataframe['rsi'] = ta.RSI(dataframe)
        macd = ta.MACD(dataframe)
        dataframe['macd'] = macd['macd']
        dataframe['macdsignal'] = macd['macdsignal']
        dataframe['macdhist'] = macd['macdhist']

        bollinger = ta.BBANDS(dataframe, timeperiod=20, nbdevup=2.0, nbdevdn=2.0)
        dataframe['bb_lowerband'] = bollinger['lowerband']
        dataframe['bb_middleband'] = bollinger['middleband']
        dataframe['bb_upperband'] = bollinger['upperband']
        return dataframe
```

### 可优化参数

我们继续定义可优化参数：

```python
class MyAwesomeStrategy(IStrategy):
    buy_adx = DecimalParameter(20, 40, decimals=1, default=30.1, space="buy")
    buy_rsi = IntParameter(20, 40, default=30, space="buy")
    buy_adx_enabled = BooleanParameter(default=True, space="buy")
    buy_rsi_enabled = CategoricalParameter([True, False], default=False, space="buy")
    buy_trigger = CategoricalParameter(["bb_lower", "macd_cross_signal"], default="bb_lower", space="buy")
```

上述定义表明：我们有五个参数需要随机组合以寻找最佳组合。  
`buy_rsi` 是一个整数参数，将在 20 到 40 之间进行测试。该参数空间的大小为 20。  
`buy_adx` 是一个小数参数，将在 20 到 40 之间以 1 位小数的精度进行评估（即取值为 20.1、20.2 等）。该参数空间的大小为 200。  
接下来我们还有三个分类变量。前两个是 `True` 或 `False` 的二值选择。
我们使用这些变量来启用或禁用 ADX 和 RSI 保护条件。
最后一个变量名为 `trigger`，用于决定使用哪种买入触发条件。

!!! Note "参数空间分配"
    参数必须被赋值给名为 `buy_*` 或 `sell_*` 的变量——或者包含 `space='buy'` | `space='sell'` 才能正确分配到对应空间。
    如果某个空间没有可用参数，运行 hyperopt 时将收到未找到该空间的错误。  
    空间不明确的参数（例如 `adx_period = IntParameter(4, 24, default=14)`——既无显式也无隐式空间声明）将不会被检测到，因此会被忽略。

现在让我们使用这些数值来编写买入策略：

```python
    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        # GUARDS AND TRENDS
        if self.buy_adx_enabled.value:
            conditions.append(dataframe['adx'] > self.buy_adx.value)
        if self.buy_rsi_enabled.value:
            conditions.append(dataframe['rsi'] < self.buy_rsi.value)

        # TRIGGERS
        if self.buy_trigger.value == 'bb_lower':
            conditions.append(dataframe['close'] < dataframe['bb_lowerband'])
        if self.buy_trigger.value == 'macd_cross_signal':
            conditions.append(qtpylib.crossed_above(
                dataframe['macd'], dataframe['macdsignal']
            ))

        # Check that volume is not 0
        conditions.append(dataframe['volume'] > 0)

        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'enter_long'] = 1

        return dataframe
```

Hyperopt 现在会多次（`epochs`）调用 `populate_entry_trend()` 函数，每次使用不同的参数组合。  
它将使用给定的历史数据，并基于上述函数生成的买入信号模拟买入操作。  
根据结果，hyperopt 会告诉你哪个参数组合产生了最佳结果（基于配置的[损失函数](#loss-functions)）。

!!! Note
    上述设置期望在已填充的指标中找到 ADX、RSI 和布林带指标。  
    当你想测试当前机器人未使用的指标时，请记得将其添加到策略文件或 hyperopt 文件中的 `populate_indicators()` 方法中。

## 参数类型

共有四种参数类型，分别适用于不同用途。

* `IntParameter` - 定义整型参数，包含搜索空间的上界和下界。
* `DecimalParameter` - 定义浮点参数，具有有限的小数位数（默认为3位）。在大多数情况下应优先使用此类型而非 `RealParameter`。
* `RealParameter` - 定义浮点参数，包含上界和下界，无精度限制。由于会创建近乎无限可能性的搜索空间，很少使用。
* `CategoricalParameter` - 定义具有预定数量选项的参数。
* `BooleanParameter` - `CategoricalParameter([True, False])` 的简写形式，非常适合"启用"类参数。

### 参数选项

有两个参数选项可以帮助您快速测试各种想法：

* `optimize` - 当设置为 `False` 时，该参数将不会包含在优化过程中。（默认值：True）
* `load` - 当设置为 `False` 时，先前超参数优化运行的结果（位于策略中的 `buy_params` 和 `sell_params` 或 JSON 输出文件中）将不会用作后续超参数优化的起始值。将改用参数中指定的默认值。（默认值：True）

!!! Tip "`load=False` 对回测的影响"
    请注意，将 `load` 选项设置为 `False` 意味着回测也将使用参数中指定的默认值，而*不是*通过超参数优化找到的值。

!!! Warning
    可超参数优化的参数不能在 `populate_indicators` 中使用 - 因为超参数优化不会为每个周期重新计算指标，所以在这种情况下将使用起始值。

## 优化指标参数

假设您有一个简单的策略想法 - EMA 交叉策略（2 条移动平均线交叉） - 并且您希望找到该策略的理想参数。
默认情况下，我们假设止损为 5% - 止盈（`minimal_roi`）为 10% - 这意味着 freqtrade 将在达到 10% 利润时卖出交易。

``` python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    minimal_roi = {
        "0":  0.10
    }
    # Define the parameter spaces
    buy_ema_short = IntParameter(3, 50, default=5)
    buy_ema_long = IntParameter(15, 200, default=50)


    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        """Generate all indicators used by the strategy"""
        
        # Calculate all ema_short values
        for val in self.buy_ema_short.range:
            dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)
        
        # Calculate all ema_long values
        for val in self.buy_ema_long.range:
            dataframe[f'ema_long_{val}'] = ta.EMA(dataframe, timeperiod=val)
        
        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        conditions.append(qtpylib.crossed_above(
                dataframe[f'ema_short_{self.buy_ema_short.value}'], dataframe[f'ema_long_{self.buy_ema_long.value}']
            ))

        # Check that volume is not 0
        conditions.append(dataframe['volume'] > 0)

        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'enter_long'] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        conditions = []
        conditions.append(qtpylib.crossed_above(
                dataframe[f'ema_long_{self.buy_ema_long.value}'], dataframe[f'ema_short_{self.buy_ema_short.value}']
            ))

        # Check that volume is not 0
        conditions.append(dataframe['volume'] > 0)

        if conditions:
            dataframe.loc[
                reduce(lambda x, y: x & y, conditions),
                'exit_long'] = 1
        return dataframe
```

分解说明：

使用 `self.buy_ema_short.range` 将返回一个范围对象，包含参数低值和高值之间的所有条目。
在此示例中（`IntParameter(3, 50, default=5)`），循环将遍历3到50之间的所有数字（`[3, 4, 5, ... 49, 50]`）。
通过在循环中使用此方法，hyperopt 将生成48个新列（`['buy_ema_3', 'buy_ema_4', ... , 'buy_ema_50']`）。

Hyperopt 随后将使用选定的数值来生成买入和卖出信号。

虽然该策略很可能过于简单而无法实现持续盈利，但它可以作为优化指标参数的示例。

!!! Note
    `self.buy_ema_short.range` 在 hyperopt 模式与其他模式下的行为有所不同。对于 hyperopt，上述示例可能生成48个新列，但在其他所有模式（回测、模拟/实盘）下，它仅会为选定值生成对应列。因此应避免使用显式数值（除 `self.buy_ema_short.value` 外的值）来调用结果列。

!!! Note
    `range` 属性也可与 `DecimalParameter` 和 `CategoricalParameter` 配合使用。由于无限搜索空间的特性，`RealParameter` 不提供此属性。

??? Hint "性能提示"
    在常规超参数优化过程中，指标会被计算一次并供给每个迭代周期使用，这会随着核心数增加而线性提升内存占用。鉴于其对性能的影响，有两种降低内存占用的替代方案：

    * Move `ema_short` and `ema_long` calculations from `populate_indicators()` to `populate_entry_trend()`. Since `populate_entry_trend()` will be calculated every epoch, you don't need to use `.range` functionality.
    * hyperopt provides `--analyze-per-epoch` which will move the execution of `populate_indicators()` to the epoch process, calculating a single value per parameter per epoch instead of using the `.range` functionality. In this case, `.range` functionality will only return the actually used value.

    These alternatives will reduce RAM usage, but increase CPU usage. However, your hyperopting run will be less likely to fail due to Out Of Memory (OOM) issues.

    Whether you are using `.range` functionality or the alternatives above, you should try to use space ranges as small as possible since this will improve CPU/RAM usage.

## 优化保护机制

Freqtrade 同样支持优化保护机制。具体优化方式取决于您的需求，以下内容仅作为示例参考。

策略只需将 "protections" 条目定义为返回保护配置列表的属性。

``` python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'
    # Define the parameter spaces
    cooldown_lookback = IntParameter(2, 48, default=5, space="protection", optimize=True)
    stop_duration = IntParameter(12, 200, default=5, space="protection", optimize=True)
    use_stop_protection = BooleanParameter(default=True, space="protection", optimize=True)


    @property
    def protections(self):
        prot = []

        prot.append({
            "method": "CooldownPeriod",
            "stop_duration_candles": self.cooldown_lookback.value
        })
        if self.use_stop_protection.value:
            prot.append({
                "method": "StoplossGuard",
                "lookback_period_candles": 24 * 3,
                "trade_limit": 4,
                "stop_duration_candles": self.stop_duration.value,
                "only_per_pair": False
            })

        return prot

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # ...
        
```

随后可通过以下命令运行超参数优化：
`freqtrade hyperopt --hyperopt-loss SharpeHyperOptLossDaily --strategy MyAwesomeStrategy --spaces protection`

!!! Note
    保护空间不属于默认空间，仅适用于参数化超参数优化接口，不兼容旧版超参数优化接口（需独立超参数优化文件）。
    若选择保护空间，Freqtrade 将自动切换 "--enable-protections" 标志。

!!! Warning
    若保护机制已定义为属性，配置中的条目将被忽略。
    因此建议不要在配置文件中定义保护机制。

### 从旧版属性设置迁移

从旧版设置迁移非常简单，只需将保护条目转换为属性即可。
简而言之，以下配置将转换为如下形式。

``` python
class MyAwesomeStrategy(IStrategy):
    protections = [
        {
            "method": "CooldownPeriod",
            "stop_duration_candles": 4
        }
    ]
```

结果

``` python
class MyAwesomeStrategy(IStrategy):
    
    @property
    def protections(self):
        return [
            {
                "method": "CooldownPeriod",
                "stop_duration_candles": 4
            }
        ]
```

随后您可将潜在的重要条目改为参数，以便进行超参数优化。

### 优化 `max_entry_position_adjustment`

虽然 `max_entry_position_adjustment` 不是一个独立的空间，但通过使用上述属性方法，仍然可以在超参数优化中使用它。

``` python
from pandas import DataFrame
from functools import reduce

import talib.abstract as ta

from freqtrade.strategy import (BooleanParameter, CategoricalParameter, DecimalParameter, 
                                IStrategy, IntParameter)
import freqtrade.vendor.qtpylib.indicators as qtpylib

class MyAwesomeStrategy(IStrategy):
    stoploss = -0.05
    timeframe = '15m'

    # Define the parameter spaces
    max_epa = CategoricalParameter([-1, 0, 1, 3, 5, 10], default=1, space="buy", optimize=True)

    @property
    def max_entry_position_adjustment(self):
        return self.max_epa.value
        

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # ...
```

??? Tip "使用 `IntParameter`"
    你也可以使用 `IntParameter` 进行此优化，但必须显式返回整数值：
    ``` python
    max_epa = IntParameter(-1, 10, default=1, space="buy", optimize=True)

    @property
    def max_entry_position_adjustment(self):
        return int(self.max_epa.value)
    ```

## 损失函数

每次超参数调优都需要一个目标。这通常被定义为一个损失函数（有时也称为目标函数），对于更理想的结果该函数值应减小，对于不良结果则增大。

必须通过 `--hyperopt-loss <类名>` 参数（或可选地通过配置中的 `"hyperopt_loss"` 键）指定损失函数。
该类应位于 `user_data/hyperopts/` 目录下的独立文件中。

目前内置了以下损失函数：

* `ShortTradeDurHyperOptLoss` - (默认的 Freqtrade 传统超优化损失函数) - 主要用于短期交易时长和避免亏损。
* `OnlyProfitHyperOptLoss` - 仅考虑利润金额。
* `SharpeHyperOptLoss` - 优化基于交易回报相对于标准差计算的夏普比率。
* `SharpeHyperOptLossDaily` - 优化基于**每日**交易回报相对于标准差计算的夏普比率。
* `SortinoHyperOptLoss` - 优化基于交易回报相对于**下行**标准差计算的索提诺比率。
* `SortinoHyperOptLossDaily` - 优化基于**每日**交易回报相对于**下行**标准差计算的索提诺比率。
* `MaxDrawDownHyperOptLoss` - 优化最大绝对回撤。
* `MaxDrawDownRelativeHyperOptLoss` - 优化最大绝对回撤，同时调整最大相对回撤。
* `MaxDrawDownPerPairHyperOptLoss` - 计算每对交易品种的利润/回撤比率，并将最差结果作为目标，强制超优化为交易对列表中的所有交易对优化参数。这样，我们可以防止一个或多个结果良好的交易对夸大指标，而结果较差的交易对未被代表因此未被优化。
* `CalmarHyperOptLoss` - 优化基于交易回报相对于最大回撤计算的卡尔玛比率。
* `ProfitDrawDownHyperOptLoss` - 以最大利润和最小回撤为目标进行优化。超优化损失文件中的 `DRAWDOWN_MULT` 变量可以调整，以在回撤目的上更严格或更灵活。
* `MultiMetricHyperOptLoss` - 通过多个关键指标进行优化以实现平衡的性能。主要重点是最大化利润和最小化回撤，同时考虑其他指标，如盈利因子、期望比率和胜率。此外，它对交易次数较少的周期施加惩罚，鼓励具有足够交易频率的策略。

自定义损失函数的创建在文档的[高级超参数优化](advanced-hyperopt.md)部分有所涵盖。

## 执行超参数优化

更新超参数优化配置后即可运行。
由于超参数优化会尝试大量组合以寻找最佳参数，因此需要时间才能获得良好结果。

我们强烈建议使用 `screen` 或 `tmux` 来防止任何连接中断。

```bash
freqtrade hyperopt --config config.json --hyperopt-loss <hyperoptlossname> --strategy <strategyname> -e 500 --spaces all
```

`-e` 选项用于设置超参数优化的评估次数。由于超参数优化采用贝叶斯搜索，一次性运行过多轮次可能不会产生更好的结果。经验表明，通常在500-1000轮次后结果改善不大。  
`--early-stop` 选项用于设置在多少轮次无改进后停止优化。建议值为总轮次的20-30%。任何大于0且小于20的值将被替换为20。默认情况下提前停止功能是禁用的（`--early-stop=0`）。

使用不同随机状态进行多次运行（执行），每次运行几千轮次，很可能会产生不同的结果。

`--spaces all` 选项表示应优化所有可能的参数。具体可能性如下所列。

!!! Note
    Hyperopt 将使用超参数优化开始时间的时间戳来存储结果。
    读取命令（`hyperopt-list`、`hyperopt-show`）可以使用 `--hyperopt-filename <filename>` 来读取和显示较早的超参数优化结果。
    您可以通过 `ls -l user_data/hyperopt_results/` 命令查看文件名列表。

### 使用不同的历史数据源执行超参数优化

如果您希望使用磁盘上备用的历史数据集进行超参数优化，请使用 `--datadir PATH` 选项。默认情况下，超参数优化使用 `user_data/data` 目录中的数据。

### 使用较小的测试集运行超参数优化

使用 `--timerange` 参数来更改您希望使用的测试集范围。
例如，要使用一个月的数据，请在超参数优化调用中传入 `--timerange 20210101-20210201`（从 2021 年 1 月到 2021 年 2 月）。

完整命令：

```bash
freqtrade hyperopt --strategy <strategyname> --timerange 20210101-20210201
```

### 使用较小的搜索空间运行超参数优化

使用 `--spaces` 选项来限制超参数优化使用的搜索空间。
让 Hyperopt 优化所有参数是一个极其庞大的搜索空间。
通常，更合理的做法可能是从仅搜索初始买入算法开始。
或者，您可能只想为您那个很棒的新买入策略优化止损或 ROI 表格。

合法值为：

* `all`: 优化所有参数
* `buy`: 仅搜索新的买入策略
* `sell`: 仅搜索新的卖出策略
* `roi`: 仅优化策略的最小盈利表
* `stoploss`: 搜索最佳止损值
* `trailing`: 搜索最佳追踪止损值
* `trades`: 搜索最佳最大开仓交易数
* `protection`: 搜索最佳保护参数（请阅读[保护措施章节](#optimizing-protections)了解如何正确定义这些参数）
* `default`: `all` 但不包括 `trailing`、`trades` 和 `protection`
* 支持以上值的空格分隔列表，例如 `--spaces roi stoploss`

当未指定 `--space` 命令行选项时，默认使用的 Hyperopt 搜索空间不包含 `trailing` 超空间。我们建议在找到、验证并将其他超空间的最佳参数粘贴到您的自定义策略中后，单独运行 `trailing` 超空间的优化。

## 理解 Hyperopt 结果

Hyperopt 完成后，您可以使用结果来更新策略。
给定以下来自 hyperopt 的结果：

```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }
```

您应该这样理解该结果：

* 效果最好的买入触发条件是 `bb_lower`。
* 您不应使用 ADX，因为 `'buy_adx_enabled': False`。
* 您应该**考虑**使用 RSI 指标（`'buy_rsi_enabled': True`），其最佳值为 `29.0`（`'buy_rsi': 29.0`）

### 策略参数自动应用

使用超参优化参数时，超参优化运行结果将写入策略同目录的json文件中（例如`MyAwesomeStrategy.py`对应的文件为`MyAwesomeStrategy.json`）。  
使用`hyperopt-show`子命令时也会更新该文件，除非在这两个命令中提供`--disable-param-export`参数。

您的策略类也可以显式包含这些结果。只需复制超参优化结果块并粘贴到类级别，替换旧参数（如有）。下次执行策略时将自动加载新参数。

将完整超参优化结果转移到策略中的操作示例如下：

```python
class MyAwesomeStrategy(IStrategy):
    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }
```

!!! Note
    配置文件中的值将覆盖参数文件级别的参数——而这两者都会覆盖策略内部的参数。
    因此优先级顺序为：配置文件 > 参数文件 > 策略`*_params` > 参数默认值

### 理解超参优化ROI结果

如果正在优化ROI（即优化搜索空间包含'all'、'default'或'roi'），您的结果将如下所示并包含ROI表格：

```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # ROI table:
    minimal_roi = {
        0: 0.10674,
        21: 0.09158,
        78: 0.03634,
        118: 0
    }
```

若要在回测及实盘交易/模拟运行中使用超参优化找到的最佳ROI表格，请将其复制粘贴作为自定义策略的`minimal_roi`属性值：

```
    # Minimal ROI designed for the strategy.
    # This attribute will be overridden if the config file contains "minimal_roi"
    minimal_roi = {
        0: 0.10674,
        21: 0.09158,
        78: 0.03634,
        118: 0
    }
```

如注释中所述，您也可以将其用作配置文件中 `minimal_roi` 设置的值。

#### 默认 ROI 搜索空间

如果您正在优化 ROI，Freqtrade 会为您创建 'roi' 优化超空间——这是 ROI 表组件的超空间。默认情况下，由 Freqtrade 生成的每个 ROI 表包含 4 行（步骤）。Hyperopt 为 ROI 表实现了自适应范围，其中 ROI 步骤中的值范围取决于所使用的时间框架。默认情况下，值在以下范围内变化（对于一些最常用的时间框架，值四舍五入到小数点后 3 位）：

| # 步骤 | 1m     |               | 5m       |             | 1h         |               | 1d           |               |
| ------ | ------ | ------------- | -------- | ----------- | ---------- | ------------- | ------------ | ------------- |
| 1      | 0      | 0.011...0.119 | 0        | 0.03...0.31 | 0          | 0.068...0.711 | 0            | 0.121...1.258 |
| 2      | 2...8  | 0.007...0.042 | 10...40  | 0.02...0.11 | 120...480  | 0.045...0.252 | 2880...11520 | 0.081...0.446 |
| 3      | 4...20 | 0.003...0.015 | 20...100 | 0.01...0.04 | 240...1200 | 0.022...0.091 | 5760...28800 | 0.040...0.162 |
| 4      | 6...44 | 0.0           | 30...220 | 0.0         | 360...2640 | 0.0           | 8640...63360 | 0.0           |

在大多数情况下，这些范围应该是足够的。步骤中的分钟数（ROI 字典的键）会根据使用的时间框架进行线性缩放。步骤中的 ROI 值（ROI 字典的值）会根据使用的时间框架进行对数缩放。

如果你的自定义 hyperopt 中有 `generate_roi_table()` 和 `roi_space()` 方法，请移除它们，以便默认使用 Freqtrade 生成的这些自适应 ROI 表和 ROI 超参数优化空间。

如果你需要 ROI 表的组成部分在其他范围内变化，请重写 `roi_space()` 方法。如果你需要不同的 ROI 表结构或其他数量的行（步骤），请重写 `generate_roi_table()` 和 `roi_space()` 方法，并在超参数优化期间实现你自己的自定义方法来生成 ROI 表。

这些方法的示例可以在[重写预定义空间部分](advanced-hyperopt.md#overriding-pre-defined-spaces)找到。

!!! Note "Reduced search space"
    为了进一步限制搜索空间，Decimal 类型被限制为 3 位小数（精度为 0.001）。这通常是足够的，任何比这更精确的值通常会导致过拟合的结果。但是，你可以通过[重写预定义空间](advanced-hyperopt.md#overriding-pre-defined-spaces)来根据需要更改此设置。

### 理解 Hyperopt 止损结果

如果您正在优化止损值（即优化搜索空间包含 'all'、'default' 或 'stoploss'），您的结果将如下所示并包含止损值：

```
Best result:

    44/100:    135 trades. Avg profit  0.57%. Total profit  0.03871918 BTC (0.7722%). Avg duration 180.4 mins. Objective: 1.94367

    # Buy hyperspace params:
    buy_params = {
        'buy_adx': 44,
        'buy_rsi': 29,
        'buy_adx_enabled': False,
        'buy_rsi_enabled': True,
        'buy_trigger': 'bb_lower'
    }

    stoploss: -0.27996
```

为了在回测和实盘交易/模拟运行中使用 Hyperopt 找到的最佳止损值，请将其复制粘贴作为您自定义策略的 `stoploss` 属性值：

``` python
    # Optimal stoploss designed for the strategy
    # This attribute will be overridden if the config file contains "stoploss"
    stoploss = -0.27996
```

如注释所述，您也可以将其用作配置文件中 `stoploss` 设置的值。

#### 默认止损搜索空间

如果您正在优化止损值，Freqtrade 会为您创建 'stoploss' 优化超空间。默认情况下，该超空间中的止损值在 -0.35...-0.02 范围内变化，这在大多数情况下已经足够。

如果您的自定义超优化文件中存在 `stoploss_space()` 方法，请将其移除以便使用 Freqtrade 默认生成的止损超优化空间。

如果您需要止损值在超优化期间在其他范围内变化，请重写 `stoploss_space()` 方法并在其中定义所需范围。该方法的示例可在[重写预定义空间部分](advanced-hyperopt.md#overriding-pre-defined-spaces)找到。

!!! Note "缩小搜索空间"
    为了进一步限制搜索空间，Decimal 数值被限制为 3 位小数（精度为 0.001）。这通常已经足够，任何比这更精确的值通常会导致过拟合的结果。但您可以通过[覆盖预定义空间](advanced-hyperopt.md#overriding-pre-defined-spaces)来根据需求调整此设置。

### 理解 Hyperopt 追踪止损结果

如果您正在优化追踪止损值（即优化搜索空间包含 'all' 或 'trailing'），您的结果将如下所示并包含追踪止损参数：

```
Best result:

    45/100:    606 trades. Avg profit  1.04%. Total profit  0.31555614 BTC ( 630.48%). Avg duration 150.3 mins. Objective: -1.10161

    # Trailing stop:
    trailing_stop = True
    trailing_stop_positive = 0.02001
    trailing_stop_positive_offset = 0.06038
    trailing_only_offset_is_reached = True
```

为了在回测和实盘交易/模拟运行中使用 Hyperopt 找到的这些最佳追踪止损参数，请将它们复制粘贴作为您自定义策略中相应属性的值：

``` python
    # Trailing stop
    # These attributes will be overridden if the config file contains corresponding values.
    trailing_stop = True
    trailing_stop_positive = 0.02001
    trailing_stop_positive_offset = 0.06038
    trailing_only_offset_is_reached = True
```

如注释所述，您也可以将其用作配置文件中相应设置的值。

#### 默认追踪止损搜索空间

如果您正在优化追踪止损值，Freqtrade 会为您创建 'trailing' 优化超空间。默认情况下，该超空间中的 `trailing_stop` 参数始终设置为 True，`trailing_only_offset_is_reached` 的值在 True 和 False 之间变化，`trailing_stop_positive` 和 `trailing_stop_positive_offset` 参数的值分别在 0.02...0.35 和 0.01...0.1 的范围内变化，这在大多数情况下已经足够。

如果您需要在超参数优化期间让追踪止损参数的值在其他范围内变化，请重写 `trailing_space()` 方法并在其中定义所需的范围。此方法的示例可以在[重写预定义空间部分](advanced-hyperopt.md#overriding-pre-defined-spaces)找到。

!!! Note "缩减的搜索空间"
    为了进一步限制搜索空间，Decimal 类型被限制为 3 位小数（精度为 0.001）。这通常已经足够，任何比这更精确的值通常会导致过拟合的结果。但您可以根据需要[重写预定义空间](advanced-hyperopt.md#overriding-pre-defined-spaces)来更改此设置。

### 可重现的结果

寻找最优参数的搜索始于在参数的超空间中进行少量（当前为 30 次）随机组合，即随机的 Hyperopt 周期。这些随机周期在 Hyperopt 输出的第一列中用星号（`*`）标记。

生成这些随机值的初始状态（随机状态）由 `--random-state` 命令行选项的值控制。您可以将其设置为任意选择的值以获得可重现的结果。

如果您未在命令行选项中显式设置此值，Hyperopt 会使用某个随机值为您设定随机状态。每次 Hyperopt 运行的随机状态值都会在日志中显示，因此您可以将其复制并粘贴到 `--random-state` 命令行选项中，以重复使用相同的初始随机周期集。

如果您未更改命令行选项、配置、时间范围、策略和 Hyperopt 类、历史数据以及损失函数——在使用相同随机状态值的情况下，您应该获得相同的超优化结果。

## 输出格式

默认情况下，hyperopt 会输出彩色化的结果——盈利为正的周期以绿色显示。这种高亮显示有助于您发现后续分析可能感兴趣的周期。总利润为零或为负（亏损）的周期以正常颜色显示。如果您不需要结果着色（例如，当您将 hyperopt 输出重定向到文件时），可以通过在命令行中指定 `--no-color` 选项来关闭着色功能。

如果您希望在 hyperopt 输出中查看所有结果（而不仅是最佳结果），可以使用 `--print-all` 命令行选项。当使用 `--print-all` 时，当前最佳结果默认也会进行彩色化——它们以粗体（高亮）样式显示。这也可以通过 `--no-color` 命令行选项关闭。

!!! Note "Windows and color output"
    Windows 系统本身不支持彩色输出，因此该功能会自动禁用。若要在 Windows 下运行 hyperopt 时获得彩色输出，请考虑使用 WSL。

## 仓位叠加与禁用最大市场仓位

在某些情况下，您可能需要使用 `--eps`/`--enable-position-staking` 参数运行 Hyperopt（及回测），或者可能需要将 `max_open_trades` 设置为一个非常大的数值以禁用开仓交易的数量限制。

默认情况下，hyperopt 模拟 Freqtrade 实盘运行/模拟运行的行为，即每个交易对只允许一个未平仓交易。所有交易对的总开仓交易数也受 `max_open_trades` 设置的限制。在 Hyperopt/回测期间，这可能导致潜在交易被已开仓交易隐藏（或屏蔽）。

`--eps`/`--enable-position-stacking` 参数允许模拟多次买入同一交易对的行为。使用一个非常大的数值设置 `--max-open-trades` 将禁用开仓交易的数量限制。

!!! Note
    模拟/实盘运行 **不会** 使用仓位叠加功能——因此在不启用此功能的情况下验证策略也很有意义，因为这更接近实际情况。

您还可以通过在配置文件中显式设置 `"position_stacking"=true` 来启用仓位叠加功能。

## 内存不足错误

由于 hyperopt 消耗大量内存（每个并行回测进程都需要将完整数据加载到内存中），您很可能会遇到“内存不足”错误。
为解决这些问题，您有多种选择：

* 减少交易对数量。
* 缩小使用的时间范围（`--timerange <timerange>`）。
* 避免使用 `--timeframe-detail`（这会加载大量额外数据到内存中）。
* 减少并行进程数量（`-j <n>`）。
* 增加机器的内存容量。
* 如果您使用了大量具有 `.range` 功能的参数，请使用 `--analyze-per-epoch`。

## 目标函数在此点已被评估过

如果您看到 `The objective has been evaluated at this point before.` 提示，这表示您的参数空间已耗尽或接近耗尽。
基本上，参数空间中的所有点都已被尝试过（或已达到局部最小值）——hyperopt 无法再找到多维空间中尚未尝试的点。
此时，Freqtrade 会通过使用新的随机点来应对“局部最小值”问题。

示例：

``` python
buy_ema_short = IntParameter(5, 20, default=10, space="buy", optimize=True)
# This is the only parameter in the buy space
```

`buy_ema_short` 参数空间有 15 个可能取值（`5, 6, ... 19, 20`）。如果您仅针对买入空间运行 hyperopt，hyperopt 在尝试完这 15 个值后便会无参数可试。
因此，您的迭代次数应与可能取值数量相匹配——或者当您发现大量 `The objective has been evaluated at this point before.` 警告时，应准备好中断运行。

## 查看 Hyperopt 结果详情

在运行 Hyperopt 达到所需的周期数后，您可以随后列出所有结果进行分析，仅选择最佳或盈利的结果，并查看之前评估的任何周期的详细信息。这可以通过 `hyperopt-list` 和 `hyperopt-show` 子命令来完成。这些子命令的用法在 [Utils](utils.md#list-hyperopt-results) 章节中有描述。

## 从策略输出调试消息

如果您想从策略输出调试消息，可以使用 `logging` 模块。默认情况下，Freqtrade 将输出所有级别为 `INFO` 或更高的消息。

``` python
import logging


logger = logging.getLogger(__name__)


class MyAwesomeStrategy(IStrategy):
    ...

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        logger.info("This is a debug message")
        ...

```

!!! Note "使用 print"
    除非禁用并行处理 (`-j 1`)，否则通过 `print()` 打印的消息不会显示在 hyperopt 输出中。
    建议改用 `logging` 模块。

## 验证回测结果

一旦优化后的策略已实施到您的策略中，您应该回测该策略以确保一切按预期工作。

为了获得与 Hyperopt 期间相同的结果（交易数量、持续时间、利润等），请在回测中使用与 hyperopt 相同的配置和参数（时间范围、时间框架等）。

### 为什么我的回测结果与 hyperopt 结果不匹配？

如果结果不匹配，请检查以下因素：

* 您可能在 `populate_indicators()` 中添加了超参数优化参数，这些参数将**针对所有周期**仅计算一次。例如，如果您尝试优化多个SMA时间周期值，可优化的时间周期参数应放置在 `populate_entry_trend()` 中，该函数会在每个周期重新计算。请参阅[优化指标参数](https://www.freqtrade.io/en/stable/hyperopt/#optimizing-an-indicator-parameter)。
* 如果您禁用了超参数优化参数自动导出到JSON参数文件的功能，请仔细检查确保将所有优化后的值正确转移到您的策略中。
* 检查日志以确认正在设置的参数及使用的具体数值。
* 请特别关注止损、最大开仓数和移动止损参数，这些参数通常在配置文件中设置，会覆盖对策略的更改。检查回测日志确保没有因配置而意外设置的参数（如 `stoploss`、`max_open_trades` 或 `trailing_stop`）。
* 确认没有意外的参数JSON文件覆盖策略中的参数或默认超参数优化设置。
* 确认在回测中启用的所有保护机制在超参数优化时也处于启用状态，反之亦然。使用 `--space protection` 时，超参数优化会自动启用保护机制。