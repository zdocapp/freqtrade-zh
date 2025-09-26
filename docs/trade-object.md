# 交易对象

## 交易

freqtrade 开立的仓位存储在 `Trade` 对象中 - 该对象会持久化到数据库。
这是 freqtrade 的核心概念 - 您会在文档的许多部分遇到它，这些部分很可能会指向此处。

它会在许多[策略回调](strategy-callbacks.md)中传递给策略。传递给策略的对象不能直接修改。根据回调结果可能会产生间接修改。

## 交易 - 可用属性

以下属性可用于每个独立交易 - 并可通过 `trade.<属性名>` 使用（例如 `trade.pair`）。

| 属性 | 数据类型 | 描述 |
|------------|-------------|-------------|
| `pair` | string | 该交易对的名称。 |
| `is_open` | boolean | 交易当前是否开放，或已结束。 |
| `open_rate` | float | 交易开仓时的汇率（若存在交易调整，则为平均开仓汇率）。 |
| `close_rate` | float | 平仓汇率 - 仅在 `is_open` = False 时设置。 |
| `stake_amount` | float | 以质押（或报价）货币计量的金额。 |
| `amount` | float | 当前持有的资产/基础货币数量。在初始订单成交前为 0.0。 |
| `open_date` | datetime | 交易开仓的时间戳 **请改用 `open_date_utc`** |
| `open_date_utc` | datetime | 交易开仓的时间戳 - UTC 时间。 |
| `close_date` | datetime | 交易平仓的时间戳 **请改用 `close_date_utc`** |
| `close_date_utc` | datetime | 交易平仓的时间戳 - UTC 时间。 |
| `close_profit` | float | 交易平仓时的相对利润。`0.01` == 1% |
| `close_profit_abs` | float | 交易平仓时的绝对利润（以质押货币计量）。 |
| `realized_profit` | float | 交易仍开放时已实现的绝对利润（以质押货币计量）。 |
| `leverage` | float | 该交易使用的杠杆 - 现货市场默认为 1.0。 |
| `enter_tag` | string | 通过数据框中的 `enter_tag` 列在开仓时提供的标签。 |
| `is_short` | boolean | 空头交易为 True，否则为 False。 |
| `orders` | Order[] | 附加到此交易的订单对象列表（包括已成交和已取消的订单）。 |
| `date_last_filled_utc` | datetime | 最后成交订单的时间。 |
| `entry_side` | "buy" / "sell" | 交易开仓时的订单方向。 |
| `exit_side` | "buy" / "sell" | 将导致交易退出/仓位减少的订单方向。 |
| `trade_direction` | "long" / "short" | 文本形式的交易方向 - 多头或空头。 |
| `nr_of_successful_entries` | int | 成功（已成交）的开仓订单数量。 |
| `nr_of_successful_exits` | int | 成功（已成交）的平仓订单数量。 |
| `has_open_orders` | boolean | 交易是否有未结订单（不包括止损订单）。 |

## 类方法

以下是类方法——它们返回通用信息，通常会对数据库执行显式查询。
可以以 `Trade.<方法>` 的形式使用——例如 `open_trades = Trade.get_open_trade_count()`

!!! Warning "回测/超参优化"
    大多数方法在回测/超参优化和实盘/模拟盘模式下均能工作。
    在回测期间，仅限于在[策略回调函数](strategy-callbacks.md)中使用。不支持在 `populate_*()` 方法中使用，否则会导致错误结果。

### get_trades_proxy

当您的策略需要获取现有（开仓或平仓）交易的相关信息时，最好使用 `Trade.get_trades_proxy()`。

使用方法:

``` python
from freqtrade.persistence import Trade
from datetime import timedelta

# ...
trade_hist = Trade.get_trades_proxy(pair='ETH/USDT', is_open=False, open_date=current_date - timedelta(days=2))

```

`get_trades_proxy()` 支持以下关键字参数。所有参数均为可选——不带参数调用 `get_trades_proxy()` 将返回数据库中的所有交易列表。

* `pair` 例如 `pair='ETH/USDT'`
* `is_open` 例如 `is_open=False`
* `open_date` 例如 `open_date=current_date - timedelta(days=2)`
* `close_date` 例如 `close_date=current_date - timedelta(days=5)`

### get_open_trade_count

获取当前开仓交易的数量

``` python
from freqtrade.persistence import Trade
# ...
open_trades = Trade.get_open_trade_count()
```

### get_total_closed_profit

检索机器人迄今为止产生的总利润。
汇总所有已平仓交易的 `close_profit_abs`。

``` python
from freqtrade.persistence import Trade

# ...
profit = Trade.get_total_closed_profit()
```

### total_open_trades_stakes

检索当前交易中的总投入金额。

``` python
from freqtrade.persistence import Trade

# ...
profit = Trade.total_open_trades_stakes()
```

### get_overall_performance

获取整体性能 - 类似于电报命令 `/performance`。

``` python
from freqtrade.persistence import Trade

# ...
if self.config['runmode'].value in ('live', 'dry_run'):
    performance = Trade.get_overall_performance()
```

示例返回值：ETH/BTC 共有 5 笔交易，总利润为 1.5%（比率为 0.015）。

``` json
{"pair": "ETH/BTC", "profit": 0.015, "count": 5}
```

## 订单对象

`Order` 对象代表交易所上的订单（或模拟运行模式下的模拟订单）。
`Order` 对象始终与其对应的[`Trade`](#trade-object)相关联，并且仅在交易上下文中才有实际意义。

### 订单 - 可用属性

订单对象通常附加到交易上。
由于这些属性依赖于交易所的响应，大多数属性可能为 None。

| 属性 | 数据类型 | 描述 |
|------------|-------------|-------------|
| `trade` | Trade | 此订单关联的交易对象 |
| `ft_pair` | string | 订单对应的交易对 |
| `ft_is_open` | boolean | 订单是否仍处于开放状态？ |
| `order_type` | string | 交易所定义的订单类型 - 通常是市价单、限价单或止损单 |
| `status` | string | 由 [ccxt 订单结构](https://docs.ccxt.com/#/README?id=order-structure) 定义的状态。通常是 open、closed、expired、canceled 或 rejected |
| `side` | string | 买入或卖出 |
| `price` | float | 订单下单价格 |
| `average` | float | 订单成交的平均价格 |
| `amount` | float | 基础货币的数量 |
| `filled` | float | 已成交数量（以基础货币计）（请改用 `safe_filled`） |
| `safe_filled` | float | 已成交数量（以基础货币计）- 保证不为 None |
| `remaining` | float | 剩余数量（请改用 `safe_remaining`） |
| `safe_remaining` | float | 剩余数量 - 从交易所获取或计算得出 |
| `cost` | float | 订单成本 - 通常是 average * filled（*期货交易取决于交易所，可能包含带杠杆或不带杠杆的成本，并且可能以合约计*） |
| `stake_amount` | float | 此订单使用的保证金金额。*2023.7 版本新增* |
| `stake_amount_filled` | float | 此订单已成交的保证金金额。*2024.11 版本新增* |
| `order_date` | datetime | 订单创建日期 **请改用 `order_date_utc`** |
| `order_date_utc` | datetime | 订单创建日期（UTC 时间） |
| `order_fill_date` | datetime | 订单成交日期 **请改用 `order_fill_date_utc`** |
| `order_fill_date_utc` | datetime | 订单成交日期（UTC 时间） |