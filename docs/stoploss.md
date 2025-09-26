# 止损

`stoploss` 配置参数是以比率表示的损失阈值，达到该阈值将触发卖出。
例如，值 `-0.10` 会在特定交易的利润跌破 -10% 时立即卖出。此参数为可选配置。
止损计算已包含手续费，因此 -10% 的止损点会精确设置在入场点下方 10% 的位置。

多数策略文件已包含最优化的 `stoploss` 值。

!!! Info
    本文档提及的所有止损属性均可在策略或配置中设置。  
    <ins>配置值将覆盖策略值。</ins>

## 交易所/Freqtrade 止损

这些止损模式可分为*交易所止损*和*非交易所止损*。

可通过以下值配置这些模式：

``` python
    'emergency_exit': 'market',
    'stoploss_on_exchange': False
    'stoploss_on_exchange_interval': 60,
    'stoploss_on_exchange_limit_ratio': 0.99
```

交易所止损仅适用于以下交易所，且并非所有交易所都同时支持止损限价和止损市价。
若仅有一种模式可用，订单类型设置将被忽略。

| 交易所 | 止损类型 |
|----------|-------------|
| Binance  | limit |
| Binance Futures  | market, limit |
| Bingx    | market, limit |
| Bitget   | market, limit |
| HTX      | limit |
| kraken   | market, limit |
| Gate     | limit |
| Okx      | limit |
| Kucoin   | stop-limit, stop-market|
| Hyperliquid (仅期货)   | limit |

!!! Note "紧密止损"
    <ins>在交易所使用止损时，不要设置过低/过紧的止损值！</ins>  
    如果设置得过低/过紧，订单成交失败的风险会更高，导致止损功能失效。

### stoploss_on_exchange 和 stoploss_on_exchange_limit_ratio

启用或禁用交易所止损功能。  
如果止损设置在*交易所*，意味着买入订单成交后，系统会立即在交易所下一个止损限价单。这能保护您免受市场突然暴跌的影响，因为订单执行完全在交易所内部进行，没有潜在的网络延迟风险。

如果 `stoploss_on_exchange` 使用限价单，交易所需要两个价格：止损价和限价。  
`stoploss` 定义了挂出限价单的止损触发价——而限价应略低于此价格。  
如果交易所同时支持限价和市价止损单，则 `stoploss` 的值将用于决定止损单类型。

计算示例：我们以 100\$ 的价格买入资产。  
止损触发价为 95\$，则限价应为 `95 * 0.99 = 94.05\$`——因此限价单的成交价格区间在 95\$ 到 94.05\$ 之间。

例如，假设已启用交易所止损和移动止损功能，且市场处于上涨趋势，机器人会自动取消之前的止损单，并以高于原止损单的触发值重新挂出新的止损单。

!!! Note
    如果启用了 `stoploss_on_exchange` 并且在交易所手动取消了止损单，那么机器人将创建一个新的止损订单。

### stoploss_on_exchange_interval

对于交易所止损，还有另一个名为 `stoploss_on_exchange_interval` 的参数。该参数配置机器人检查止损并在必要时更新止损单的时间间隔（以秒为单位）。
机器人无法每 5 秒（每次迭代）执行这些操作，否则会被交易所封禁。
因此，此参数将告知机器人应多久更新一次止损订单。默认值为 60（1 分钟）。
如果您意外取消了止损单，同样的逻辑将在交易所重新应用止损订单。

### stoploss_price_type

!!! Warning "仅适用于期货"
    `stoploss_price_type` 仅适用于期货市场（在支持该功能的交易所上）。
    Freqtrade 将在启动时验证此设置，如果为您的交易所选择了无效的设置，将无法启动。
    不同交易所支持的价位类型会有所不同。请向您的交易所核实其支持的价位类型。

期货市场上的交易所止损可以根据不同的价位类型触发。
这些价位在交易所术语中的命名通常各不相同，但通常围绕"最新价"（或"合约价格"）、"标记价格"和"指数价格"等。

此设置的可接受值为 `"last"`、`"mark"` 和 `"index"` - freqtrade 将自动将其转换为相应的 API 类型，并相应地下达[交易所止损](#stoploss_on_exchange-and-stoploss_on_exchange_limit_ratio)订单。

### force_exit

`force_exit` 是一个可选值，默认与 `exit` 值相同，在通过 Telegram 或 Rest API 发送 `/forceexit` 命令时使用。

### force_entry

`force_entry` 是一个可选值，默认与 `entry` 值相同，在通过 Telegram 或 Rest API 发送 `/forceentry` 命令时使用。

### emergency_exit

`emergency_exit` 是一个可选值，默认为 `market`，在创建交易所止损订单失败时使用。
以下是默认值，如果在策略或配置文件中未更改则使用此设置。

策略文件示例：

``` python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": True,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99
}
```

## 止损类型

目前机器人包含以下止损支持模式：

1. 静态止损
2. 追踪止损
3. 自定义正向损失的追踪止损
4. 仅在交易达到特定偏移量后启动的追踪止损
5. [自定义止损函数](strategy-callbacks.md#custom-stoploss)

### 静态止损

这非常简单，您需要定义一个止损值 x（作为价格的比例，即价格的 x * 100%）。一旦亏损超过设定的止损值，系统将尝试卖出该资产。

止损示例：

``` python
    stoploss = -0.10
```

例如，简化计算：

* 机器人以 100 美元的价格买入资产
* 止损设定为 -10%
* 当资产价格跌破 90 美元时，止损将被触发

### 追踪止损

其初始值为 `stoploss`，就像您定义静态止损一样。
启用追踪止损：

``` python
    stoploss = -0.10
    trailing_stop = True
```

这将激活一个算法，每当您的资产价格上涨时，该算法会自动向上移动止损位。

例如，简化计算：

* 机器人以 100 美元的价格买入资产
* 止损设定为 -10%
* 当资产价格跌破 90 美元时，止损将被触发
* 假设资产现在上涨至 102 美元
* 止损将变为 102 美元的 -10% = 91.8 美元
* 现在资产价值跌至 101 美元，止损位仍为 91.8 美元，并将在 91.8 美元触发。

总结：止损将调整为始终为观察到的最高价格的 -10%。

### 追踪止损，不同的正向亏损

你也可以在买入处于亏损状态（买入价 - 手续费）时设置一个默认止损，但一旦达到正收益（或你定义的偏移值），系统将采用一个具有不同值的新止损。
例如，你的默认止损是 -10%，但一旦达到盈利（例如 0.1%），将使用不同的追踪止损。

!!! Note
    如果你希望止损仅在达到盈亏平衡或盈利时改变（大多数用户的需求），请参阅下一节关于[启用偏移](#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset)的内容。

这两个值都需要将 `trailing_stop` 设置为 true，并且 `trailing_stop_positive` 需要设定一个值。

``` python
    stoploss = -0.10
    trailing_stop = True
    trailing_stop_positive = 0.02
    trailing_stop_positive_offset = 0.0
    trailing_only_offset_is_reached = False  # Default - not necessary for this example
```

例如，简化计算如下：

* 机器人以 100 美元的价格买入资产
* 止损设定为 -10%
* 当资产价格跌破 90 美元时，止损将被触发
* 假设资产现在上涨到 102 美元
* 止损现在将是 102 美元的 -2% = 99.96 美元（99.96 美元的止损将被锁定，并随着资产价格上涨以 -2% 的幅度跟进）
* 现在资产价格跌至 101 美元，止损仍为 99.96 美元，并将在 99.96 美元触发

这里的 0.02 对应的是 -2% 的止损。
在此之前，`stoploss` 用于追踪止损。

!!! Tip "使用偏移量调整止损"
    通过将 `trailing_stop_positive_offset` 设置为高于 `trailing_stop_positive` 的值，可以确保新的追踪止损位处于盈利状态。这样设置后，首个新止损值将立即锁定利润。

    Example with simplified math:

    ``` python
        stoploss = -0.10
        trailing_stop = True
        trailing_stop_positive = 0.02
        trailing_stop_positive_offset = 0.03
    ```

    * the bot buys an asset at a price of 100$
    * the stop loss is defined at -10%, so the stop loss would get triggered once the asset drops below 90$
    * assuming the asset now increases to 102$
    * the stoploss will now be at 91.8$ - 10% below the highest observed rate
    * assuming the asset now increases to 103.5$ (above the offset configured)
    * the stop loss will now be -2% of 103.5$ = 101.43$
    * now the asset drops in value to 102\$, the stop loss will still be 101.43$ and would trigger once price breaks below 101.43$

### 仅在交易达到特定偏移量后启动追踪止损

您还可以在达到偏移量前保持固定止损，待市场转向时通过追踪止损来获取利润。

若设置 `trailing_only_offset_is_reached = True`，则追踪止损仅在达到偏移量后激活。在此之前，止损将维持在配置的 `stoploss` 水平且不会追踪。
保持 `trailing_only_offset_is_reached=False` 将允许资产价格突破初始入场价后立即启动追踪止损。

该选项可独立使用或与 `trailing_stop_positive` 配合使用，但偏移量基准采用 `trailing_stop_positive_offset`。

配置示例（偏移量为买入价+3%）：

``` python
    stoploss = -0.10
    trailing_stop = True
    trailing_stop_positive = 0.02
    trailing_stop_positive_offset = 0.03
    trailing_only_offset_is_reached = True
```

简化计算示例：

* 机器人以100美元的价格买入一项资产
* 止损设定为-10%
* 一旦资产价格跌破90美元，止损将被触发
* 除非资产价格上涨至配置的偏移量或更高，否则止损将保持在90美元
* 假设资产现在上涨到103美元（我们在此配置了偏移量）
* 止损现在将是103美元的-2% = 100.94美元
* 当资产价值下跌到101美元时，止损仍为100.94美元，并会在100.94美元触发

!!! Tip
    请确保该值（`trailing_stop_positive_offset`）低于最小投资回报率，否则最小投资回报率将优先适用并卖出交易。

## 止损与杠杆

止损应被视为"此交易的风险"——因此，在100美元的交易上设置10%的止损意味着您愿意在此交易中损失10美元（10%）——当价格向下波动10%时将触发止损。

使用杠杆时，适用相同的原则——止损定义了交易的风险（您愿意损失的金额）。

因此，在10倍杠杆交易中设置10%的止损，将在价格波动1%时触发。
如果您的保证金金额（自有资金）为100美元——这笔交易在10倍杠杆后将是1000美元。
如果价格波动1%——您将损失10美元的自有资金——因此在这种情况下止损将被触发。

请务必注意这一点，并避免使用过紧的止损（在10倍杠杆下，10%的风险可能太小，无法让交易有"喘息"的空间）。

## 修改持仓交易的止损

可以通过修改配置文件或策略中的数值，并使用 `/reload_config` 命令来更改持仓交易的止损（或者完全停止并重新启动机器人也可行）。

新的止损值将应用于持仓交易（并生成相应的日志消息）。

### 限制条件

如果启用了 `trailing_stop` 且止损已被调整，则无法修改止损值。