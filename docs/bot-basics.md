# Freqtrade 基础

本页为您介绍 Freqtrade 工作原理和操作方式的一些基本概念。

## Freqtrade 术语

* **策略**：您的交易策略，指示机器人执行操作。
* **交易**：未平仓头寸。
* **未完成订单**：当前已下达到交易所但尚未完成的订单。
* **交易对**：可交易货币对，通常采用基础货币/计价货币格式（例如现货交易为 `XRP/USDT`，期货交易为 `XRP/USDT:USDT`）。
* **时间框架**：使用的K线长度（例如 `"5m"`、`"1h"` 等）。
* **指标**：技术指标（SMA、EMA、RSI 等）。
* **限价单**：以指定限价或更优价格执行的订单。
* **市价单**：保证成交，可能因订单规模影响价格。
* **当前利润**：该交易当前待实现（未实现）利润。主要在机器人和用户界面中使用。
* **已实现利润**：已实现的利润。仅与[部分平仓](strategy-callbacks.md#adjust-trade-position)结合使用时相关——这也解释了其计算逻辑。
* **总利润**：已实现和未实现利润的总和。相对数值（%）是根据该交易的总投资额计算的。

## 手续费处理

Freqtrade 的所有利润计算均包含手续费。对于回测/超参优化/模拟运行模式，使用交易所默认费率（交易所最低档位）。对于实盘操作，使用交易所实际收取的手续费（包括 BNB 返佣等）。

## 交易对命名规则

Freqtrade 遵循 [ccxt 命名规范](https://docs.ccxt.com/#/README?id=consistency-of-base-and-quote-currencies) 进行货币命名。
在错误的市场使用错误的命名规范通常会导致机器人无法识别交易对，一般会出现"该交易对不可用"等错误。

### 现货交易对命名

现货交易对命名格式为 `基础货币/计价货币`（例如 `ETH/USDT`）。

### 期货交易对命名

期货交易对命名格式为 `基础货币/计价货币:结算货币`（例如 `ETH/USDT:USDT`）。

## 机器人执行逻辑

以模拟运行或实盘模式启动 freqtrade（使用 `freqtrade trade` 命令）将启动机器人并开始迭代循环。
同时会运行 `bot_start()` 回调函数。

默认情况下，机器人循环每几秒运行一次（`internals.process_throttle_secs`），并执行以下操作：

* 从持久化存储中获取未平仓交易。
* 计算当前可交易货币对列表。
* 下载配对列表的OHLCV数据，包括所有[信息货币对](strategy-customization.md#get-data-for-non-tradeable-pairs)  
  此步骤每个K线周期仅执行一次，以避免不必要的网络流量。
* 调用`bot_loop_start()`策略回调函数。
* 按货币对分析策略。
  * 调用`populate_indicators()`
  * 调用`populate_entry_trend()`
  * 调用`populate_exit_trend()`
* 从交易所更新未平仓订单状态。
  * 对已成交订单调用`order_filled()`策略回调函数。
  * 检查未平仓订单的超时情况。
    * 对未平仓入场订单调用`check_entry_timeout()`策略回调函数。
    * 对未平仓出场订单调用`check_exit_timeout()`策略回调函数。
    * 对未平仓订单调用`adjust_order_price()`策略回调函数。
      * 对未平仓入场订单调用`adjust_entry_price()`策略回调函数。*仅在未实现`adjust_order_price()`时调用*
      * 对未平仓出场订单调用`adjust_exit_price()`策略回调函数。*仅在未实现`adjust_order_price()`时调用*
* 验证现有持仓并最终放置出场订单。
  * 考虑止损、ROI和出场信号，`custom_exit()`和`custom_stoploss()`。
  * 根据`exit_pricing`配置设置或使用`custom_exit_price()`回调函数确定出场价格。
  * 在放置出场订单之前，调用`confirm_trade_exit()`策略回调函数。
* 如果启用了持仓调整功能，则通过调用`adjust_trade_position()`检查未平仓交易的持仓调整情况，并在需要时放置额外订单。
* 检查交易槽位是否仍可用（如果达到`max_open_trades`限制）。
* 验证入场信号以尝试建立新持仓。
  * 根据`entry_pricing`配置设置或使用`custom_entry_price()`回调函数确定入场价格。
  * 在保证金和期货模式下，调用`leverage()`策略回调函数以确定期望的杠杆率。
  * 通过调用`custom_stake_amount()`回调函数确定投资金额。
  * 在放置入场订单之前，调用`confirm_trade_entry()`策略回调函数。

这个循环将一遍又一遍地重复，直到机器人停止。

## 回测 / 超参数优化执行逻辑

[回测](backtesting.md) 或 [超参数优化](hyperopt.md) 仅执行上述逻辑的部分内容，因为大部分交易操作都是完全模拟的。

* 为配置的交易对列表加载历史数据。
* 调用一次 `bot_start()`。
* 计算指标（对每个交易对调用一次 `populate_indicators()`）。
* 计算入场/出场信号（对每个交易对调用一次 `populate_entry_trend()` 和 `populate_exit_trend()`）。
* 按K线循环模拟入场和出场点。
  * 调用策略回调函数 `bot_loop_start()`。
  * 检查订单超时，可通过 `unfilledtimeout` 配置或 `check_entry_timeout()` / `check_exit_timeout()` 策略回调实现。
  * 为未成交订单调用策略回调函数 `adjust_order_price()`。
    * 为未成交的入场订单调用策略回调函数 `adjust_entry_price()`。*仅在未实现 `adjust_order_price()` 时调用！*
    * 为未成交的出场订单调用策略回调函数 `adjust_exit_price()`。*仅在未实现 `adjust_order_price()` 时调用！*
  * 检查交易入场信号（`enter_long` / `enter_short` 列）。
  * 确认交易入场/出场（如果策略中实现了 `confirm_trade_entry()` 和 `confirm_trade_exit()` 则调用）。
  * 调用 `custom_entry_price()`（如果策略中已实现）以确定入场价格（价格会被调整至开盘K线范围内）。
  * 在保证金和期货模式下，调用 `leverage()` 策略回调以确定期望的杠杆率。
  * 通过调用 `custom_stake_amount()` 回调确定仓位大小。
  * 如果启用且存在未平仓交易，则检查仓位调整并调用 `adjust_trade_position()` 以判断是否需要追加订单。
  * 为已成交的入场订单调用 `order_filled()` 策略回调。
  * 调用 `custom_stoploss()` 和 `custom_exit()` 以查找自定义出场点。
  * 对于基于出场信号、自定义出场和部分出场的操作：调用 `custom_exit_price()` 以确定出场价格（价格会被调整至收盘K线范围内）。
  * 为已成交的出场订单调用 `order_filled()` 策略回调。
* 生成回测报告输出

!!! Note
    回测和超参数优化在计算中都包含交易所的默认费用。可以通过指定 `--fee` 参数将自定义费用传递给回测/超参数优化。

!!! Warning "回调函数调用频率"
    回测将最多在每个蜡烛图周期调用一次每个回调函数（`--timeframe-detail` 会修改此行为为每个详细蜡烛图调用一次）。
    在实盘交易中，大多数回调函数会在每次迭代时被调用（通常每约5秒一次）——这可能导致回测结果不匹配。