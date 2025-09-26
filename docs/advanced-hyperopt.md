# 高级超参数优化

本页介绍一些高级超参数优化主题，这些主题可能需要比创建普通超参数优化类更高的编码技能和Python知识。

## 创建和使用自定义损失函数

要使用自定义损失函数类，请确保在自定义超参数优化损失类中定义了函数 `hyperopt_loss_function`。
对于下面的示例，您需要在超参数优化调用中添加命令行参数 `--hyperopt-loss SuperDuperHyperOptLoss`，以便使用此函数。

下面提供了一个示例，该示例与默认超参数优化损失实现完全相同。完整示例可在 [userdata/hyperopts](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_hyperopt_loss.py) 中找到。

``` python
from datetime import datetime
from typing import Any, Dict

from pandas import DataFrame

from freqtrade.constants import Config
from freqtrade.optimize.hyperopt import IHyperOptLoss

TARGET_TRADES = 600
EXPECTED_MAX_PROFIT = 3.0
MAX_ACCEPTED_TRADE_DURATION = 300

class SuperDuperHyperOptLoss(IHyperOptLoss):
    """
    Defines the default loss function for hyperopt
    """

    @staticmethod
    def hyperopt_loss_function(
        *,
        results: DataFrame,
        trade_count: int,
        min_date: datetime,
        max_date: datetime,
        config: Config,
        processed: dict[str, DataFrame],
        backtest_stats: dict[str, Any],
        starting_balance: float,
        **kwargs,
    ) -> float:
        """
        Objective function, returns smaller number for better results
        This is the legacy algorithm (used until now in freqtrade).
        Weights are distributed as follows:
        * 0.4 to trade duration
        * 0.25: Avoiding trade loss
        * 1.0 to total profit, compared to the expected value (`EXPECTED_MAX_PROFIT`) defined above
        """
        total_profit = results['profit_ratio'].sum()
        trade_duration = results['trade_duration'].mean()

        trade_loss = 1 - 0.25 * exp(-(trade_count - TARGET_TRADES) ** 2 / 10 ** 5.8)
        profit_loss = max(0, 1 - total_profit / EXPECTED_MAX_PROFIT)
        duration_loss = 0.4 * min(trade_duration / MAX_ACCEPTED_TRADE_DURATION, 1)
        result = trade_loss + profit_loss + duration_loss
        return result
```

当前的参数包括：

* `results`: 包含结果交易的 DataFrame。  
   results 中提供以下列（与使用 `--export trades` 进行回测时的输出文件相对应）：  
   `pair, profit_ratio, profit_abs, open_date, open_rate, fee_open, close_date, close_rate, fee_close, amount, trade_duration, is_open, exit_reason, stake_amount, min_rate, max_rate, stop_loss_ratio, stop_loss_abs`
* `trade_count`: 交易数量（等同于 `len(results)`）
* `min_date`: 所使用时间范围的开始日期
* `max_date`: 所使用时间范围的结束日期
* `config`: 使用的配置对象（注意：如果策略相关参数是超参优化空间的一部分，则此处不会全部更新）
* `processed`: 以交易对为键的 DataFrame 字典，包含用于回测的数据
* `backtest_stats`: 使用与回测文件 "strategy" 子结构相同格式的回测统计信息。可用字段可见 `optimize_reports.py` 中的 `generate_strategy_stats()` 函数
* `starting_balance`: 用于回测的起始余额

此函数需要返回一个浮点数（`float`）。数值越小表示结果越好。具体的参数设置和平衡由您自行决定。

!!! Note
    此函数每个周期调用一次——因此请务必尽可能优化，以免不必要地拖慢超参优化速度。

!!! Note "`*args` and `**kwargs`"
    请在接口中保留参数 `*args` 和 `**kwargs`，以便我们未来扩展此接口。

## 覆盖预定义空间

要覆盖预定义空间（`roi_space`、`generate_roi_table`、`stoploss_space`、`trailing_space`、`max_open_trades_space`），需定义一个名为 Hyperopt 的嵌套类，并按如下方式定义所需空间：

```python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal

class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        # Define a custom stoploss space.
        def stoploss_space():
            return [SKDecimal(-0.05, -0.01, decimals=3, name='stoploss')]

        # Define custom ROI space
        def roi_space() -> List[Dimension]:
            return [
                Integer(10, 120, name='roi_t1'),
                Integer(10, 60, name='roi_t2'),
                Integer(10, 40, name='roi_t3'),
                SKDecimal(0.01, 0.04, decimals=3, name='roi_p1'),
                SKDecimal(0.01, 0.07, decimals=3, name='roi_p2'),
                SKDecimal(0.01, 0.20, decimals=3, name='roi_p3'),
            ]

        def generate_roi_table(params: Dict) -> dict[int, float]:

            roi_table = {}
            roi_table[0] = params['roi_p1'] + params['roi_p2'] + params['roi_p3']
            roi_table[params['roi_t3']] = params['roi_p1'] + params['roi_p2']
            roi_table[params['roi_t3'] + params['roi_t2']] = params['roi_p1']
            roi_table[params['roi_t3'] + params['roi_t2'] + params['roi_t1']] = 0

            return roi_table

        def trailing_space() -> List[Dimension]:
            # All parameters here are mandatory, you can only modify their type or the range.
            return [
                # Fixed to true, if optimizing trailing_stop we assume to use trailing stop at all times.
                Categorical([True], name='trailing_stop'),

                SKDecimal(0.01, 0.35, decimals=3, name='trailing_stop_positive'),
                # 'trailing_stop_positive_offset' should be greater than 'trailing_stop_positive',
                # so this intermediate parameter is used as the value of the difference between
                # them. The value of the 'trailing_stop_positive_offset' is constructed in the
                # generate_trailing_params() method.
                # This is similar to the hyperspace dimensions used for constructing the ROI tables.
                SKDecimal(0.001, 0.1, decimals=3, name='trailing_stop_positive_offset_p1'),

                Categorical([True, False], name='trailing_only_offset_is_reached'),
        ]

        # Define a custom max_open_trades space
        def max_open_trades_space(self) -> List[Dimension]:
            return [
                Integer(-1, 10, name='max_open_trades'),
            ]
```

!!! Note
    所有覆盖操作均为可选，可根据需要混合/匹配使用。

### 动态参数

参数也可以动态定义，但必须在调用 [`bot_start()` 回调](strategy-callbacks.md#bot-start)后对实例可用。

``` python

class MyAwesomeStrategy(IStrategy):

    def bot_start(self, **kwargs) -> None:
        self.buy_adx = IntParameter(20, 30, default=30, optimize=True)

    # ...
```

!!! Warning
    以此方式创建的参数不会显示在 `list-strategies` 的参数计数中。

### 覆盖基础估计器

您可以通过在 Hyperopt 子类中实现 `generate_estimator()` 来为 Hyperopt 定义自己的 optuna 采样器。

```python
class MyAwesomeStrategy(IStrategy):
    class HyperOpt:
        def generate_estimator(dimensions: List['Dimension'], **kwargs):
            return "NSGAIIISampler"

```

可能的值包括："NSGAIISampler"、"TPESampler"、"GPSampler"、"CmaEsSampler"、"NSGAIIISampler"、"QMCSampler" 之一（详细信息可查阅 [optuna-samplers 文档](https://optuna.readthedocs.io/en/stable/reference/samplers/index.html)），或"继承自 `optuna.samplers.BaseSampler` 的类的实例"。

需要一些研究来寻找其他采样器（例如来自 optunahub）。

!!! Note
    虽然可以提供自定义估计器，但作为用户，您需要自行研究可能的参数，并分析/理解应使用哪些参数。
    如果您对此不确定，最好使用默认值之一（`"NSGAIIISampler"` 已被证明是最通用的），无需额外参数。

??? Example "使用 Optunahub 的 `AutoSampler`"

    [AutoSampler docs](https://hub.optuna.org/samplers/auto_sampler/)
    
    Install the necessary dependencies 
    ``` bash
    pip install optunahub cmaes torch scipy
    ```
    Implement `generate_estimator()`  in your strategy

    ``` python
    # ...
    from freqtrade.strategy.interface import IStrategy
    from typing import List
    import optunahub
    # ... 

    class my_strategy(IStrategy):
        class HyperOpt:
            def generate_estimator(dimensions: List["Dimension"], **kwargs):
                if "random_state" in kwargs.keys():
                    return optunahub.load_module("samplers/auto_sampler").AutoSampler(seed=kwargs["random_state"])
                else:
                    return optunahub.load_module("samplers/auto_sampler").AutoSampler()

    ```

    Obviously the same approach will work for all other Samplers optuna supports.

## 空间选项

对于额外的空间，scikit-optimize（结合 Freqtrade）提供以下空间类型：

* `Categorical` - 从类别列表中选取（例如 `Categorical(['a', 'b', 'c'], name="cat")`）
* `Integer` - 从整数范围内选取（例如 `Integer(1, 10, name='rsi')`）
* `SKDecimal` - 从有限精度的十进制数范围内选取（例如 `SKDecimal(0.1, 0.5, decimals=3, name='adx')`）。*仅适用于 freqtrade*。
* `Real` - 从全精度的十进制数范围内选取（例如 `Real(0.1, 0.5, name='adx')`）

您可以从 `freqtrade.optimize.space` 导入所有这些空间，尽管 `Categorical`、`Integer` 和 `Real` 只是其对应 scikit-optimize 空间的别名。`SKDecimal` 由 freqtrade 提供，用于加速优化过程。

``` python
from freqtrade.optimize.space import Categorical, Dimension, Integer, SKDecimal, Real  # noqa
```

!!! Hint "SKDecimal 与 Real 对比"
    在几乎所有情况下，我们建议使用 `SKDecimal` 而非 `Real` 空间。虽然 Real 空间提供完全精度（最高约16位小数）——但这种精度很少需要，并且会导致不必要的超参数优化时间延长。

    Assuming the definition of a rather small space (`SKDecimal(0.10, 0.15, decimals=2, name='xxx')`) - SKDecimal will have 5 possibilities (`[0.10, 0.11, 0.12, 0.13, 0.14, 0.15]`).

    A corresponding real space `Real(0.10, 0.15 name='xxx')`  on the other hand has an almost unlimited number of possibilities (`[0.10, 0.010000000001, 0.010000000002, ... 0.014999999999, 0.01500000000]`).