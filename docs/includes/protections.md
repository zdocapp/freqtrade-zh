## 保护机制

保护机制通过临时停止单个交易对或所有交易对的交易，来保护您的策略免受意外事件和市场条件的影响。
所有保护的结束时间都会向上取整到下一个K线周期，以避免突然、意外的K线内买入。

!!! Tip "使用提示"
    并非所有保护机制都适用于所有策略，您需要根据策略调整参数以提高性能。

    Each Protection can be configured multiple times with different parameters, to allow different levels of protection (short-term / long-term).

!!! Note "回测"
    回测和超参数优化支持保护机制，但必须使用 `--enable-protections` 参数显式启用。

### 可用保护机制

* [`StoplossGuard`](#stoploss-guard) 在特定时间窗口内达到一定数量的止损时停止交易。
* [`MaxDrawdown`](#maxdrawdown) 达到最大回撤时停止交易。
* [`LowProfitPairs`](#low-profit-pairs) 锁定低收益交易对。
* [`CooldownPeriod`](#cooldown-period) 在卖出交易后不立即进入新交易。

### 所有保护机制的通用设置

|  参数| 描述 |
|------------|-------------|
| `method` | 使用的保护名称。<br> **数据类型:** 字符串，从[可用保护](#available-protections)中选择
| `stop_duration_candles` | 锁定应持续多少根K线？<br> **数据类型:** 正整数（以K线数为单位）
| `stop_duration` | 保护应锁定多少分钟。<br>不能与 `stop_duration_candles` 同时使用。<br> **数据类型:** 浮点数（以分钟为单位）
| `lookback_period_candles` | 仅考虑在过去 `lookback_period_candles` 根K线内完成的交易。某些保护可能会忽略此设置。<br> **数据类型:** 正整数（以K线数为单位）
| `lookback_period` | 仅考虑在 `当前时间 - lookback_period` 之后完成的交易。<br>不能与 `lookback_period_candles` 同时使用。<br>某些保护可能会忽略此设置。<br> **数据类型:** 浮点数（以分钟为单位）
| `trade_limit` | 至少需要的交易数量（并非所有保护都使用）。<br> **数据类型:** 正整数
| `unlock_at` | 交易将定期解锁的时间（并非所有保护都使用）。<br> **数据类型:** 字符串 <br>**输入格式:** "HH:MM"（24小时制）

!!! Note "持续时间"
    持续时间（`stop_duration*` 和 `lookback_period*` 可以以分钟或K线数定义）。
    为了在测试不同时间框架时获得更大灵活性，以下所有示例将使用"K线"定义。

#### Stoploss Guard

`StoplossGuard` 会选择 `lookback_period` 分钟内（或使用 `lookback_period_candles` 时的蜡烛数内）的所有交易。
如果 `trade_limit` 笔或更多交易触发了止损，交易将停止 `stop_duration` 分钟（或使用 `stop_duration_candles` 时的蜡烛数，或使用 `unlock_at` 时直到设定时间）。

这适用于所有交易对，除非将 `only_per_pair` 设置为 true，此时将仅针对单个交易对进行评估。

类似地，该保护默认会考虑所有交易（多单和空单）。对于期货机器人，设置 `only_per_side` 将使机器人仅考虑单侧交易，并仅锁定该侧交易，例如在一系列多单止损后允许空单继续交易。

`required_profit` 将决定触发止损考虑的所需相对盈利（或亏损）。通常不应设置此参数，默认为 0.0——这意味着所有亏损的止损都会触发锁定。

以下示例显示：如果机器人在过去 24 根蜡烛内触发了 4 次止损，则在最后一笔交易后所有交易对将停止交易 4 根蜡烛。

``` python
@property
def protections(self):
    return [
        {
            "method": "StoplossGuard",
            "lookback_period_candles": 24,
            "trade_limit": 4,
            "stop_duration_candles": 4,
            "required_profit": 0.0,
            "only_per_pair": False,
            "only_per_side": False
        }
    ]
```

!!! Note
    `StoplossGuard` 会考虑所有结果为 `"stop_loss"`、`"stoploss_on_exchange"` 和 `"trailing_stop_loss"` 且最终利润为负的交易。
    需要根据您的策略调整 `trade_limit` 和 `lookback_period` 参数。

#### MaxDrawdown

`MaxDrawdown` 使用 `lookback_period` 分钟内（或使用 `lookback_period_candles` 时的蜡烛数）的所有交易来计算最大回撤。如果回撤低于 `max_allowed_drawdown`，交易将在最后一笔交易后停止 `stop_duration` 分钟（或使用 `stop_duration_candles` 时的蜡烛数）——假设机器人需要一些时间让市场恢复。

以下示例会在过去48根蜡烛内（至少需要 `trade_limit` 笔交易）所有交易对的最大回撤超过20%时，停止交易12根蜡烛。如果需要，可以使用 `lookback_period` 和/或 `stop_duration`。

``` python
@property
def protections(self):
    return  [
        {
            "method": "MaxDrawdown",
            "lookback_period_candles": 48,
            "trade_limit": 20,
            "stop_duration_candles": 12,
            "max_allowed_drawdown": 0.2
        },
    ]
```

#### 低收益交易对

`LowProfitPairs` 使用某个交易对在 `lookback_period` 分钟内（或使用 `lookback_period_candles` 时的蜡烛数）的所有交易来计算总收益率。
如果该比率低于 `required_profit`，该交易对将被锁定 `stop_duration` 分钟（或使用 `stop_duration_candles` 时的蜡烛数，或使用 `unlock_at` 时直到设定时间）。

对于期货机器人，设置 `only_per_side` 将使机器人仅考虑单边交易，然后仅锁定该边，例如在连续多头亏损后允许空头继续交易。

以下示例会在过去6根蜡烛内某个交易对的收益率未达到2%（且至少2笔交易）时，停止交易该交易对60分钟。

``` python
@property
def protections(self):
    return [
        {
            "method": "LowProfitPairs",
            "lookback_period_candles": 6,
            "trade_limit": 2,
            "stop_duration": 60,
            "required_profit": 0.02,
            "only_per_pair": False,
        }
    ]
```

#### 冷却期

`CooldownPeriod` 会在退出交易后将交易对锁定 `stop_duration` 分钟（或使用 `stop_duration_candles` 时锁定指定K线数量，或使用 `unlock_at` 时锁定到设定时间），在此 `stop_duration` 分钟内避免该交易对重新入场。

以下示例将在平仓后停止交易该交易对2根K线时间，让该交易对进行"冷却"。

``` python
@property
def protections(self):
    return  [
        {
            "method": "CooldownPeriod",
            "stop_duration_candles": 2
        }
    ]
```

!!! Note
    该保护机制仅作用于交易对级别，永远不会全局锁定所有交易对。
    此保护不考虑 `lookback_period`，因为它仅关注最新交易。

### 保护机制完整示例

所有保护机制都可以自由组合，也可使用不同参数，为表现不佳的交易对构建递增的防护墙。
所有保护机制按照定义顺序依次评估。

以下示例假设时间框架为1小时：

* 每对货币在卖出后锁定额外5根K线时间（`冷却期`），给予其他货币对成交机会。
* 若过去2天（`48根1小时K线`）内发生20笔交易且导致最大回撤超过20%（`最大回撤`），则停止交易4小时（`4根1小时K线`）。
* 若所有货币对在1天内（`24根1小时K线`）触发超过4次止损（`止损防护`），则停止交易。
* 锁定过去6小时内（`6根1小时K线`）发生2笔交易且累计收益率低于0.02（<2%）的所有货币对（`低收益货币对`）。
* 锁定过去24小时内（`24根1小时K线`）收益率低于0.01（<1%）且至少完成4笔交易的所有货币对，持续2根K线时间。

``` python
from freqtrade.strategy import IStrategy

class AwesomeStrategy(IStrategy)
    timeframe = '1h'
    
    @property
    def protections(self):
        return [
            {
                "method": "CooldownPeriod",
                "stop_duration_candles": 5
            },
            {
                "method": "MaxDrawdown",
                "lookback_period_candles": 48,
                "trade_limit": 20,
                "stop_duration_candles": 4,
                "max_allowed_drawdown": 0.2
            },
            {
                "method": "StoplossGuard",
                "lookback_period_candles": 24,
                "trade_limit": 4,
                "stop_duration_candles": 2,
                "only_per_pair": False
            },
            {
                "method": "LowProfitPairs",
                "lookback_period_candles": 6,
                "trade_limit": 2,
                "stop_duration_candles": 60,
                "required_profit": 0.02
            },
            {
                "method": "LowProfitPairs",
                "lookback_period_candles": 24,
                "trade_limit": 4,
                "stop_duration_candles": 2,
                "required_profit": 0.01
            }
        ]
    # ...
```