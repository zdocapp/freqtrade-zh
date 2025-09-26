## 订单使用的价格

常规订单的价格可通过参数结构 `entry_pricing`（交易入场）和 `exit_pricing`（交易出场）进行控制。
价格总是在下单前即时获取，可通过查询交易所行情或使用订单簿数据实现。

!!! Note
    Freqtrade 使用的订单簿数据是通过 ccxt 的 `fetch_order_book()` 函数从交易所获取的数据，通常来自 L2 聚合订单簿；而行情数据则是 ccxt 的 `fetch_ticker()`/`fetch_tickers()` 函数返回的结构。更多详情请参阅 ccxt 库的[文档](https://github.com/ccxt/ccxt/wiki/Manual#market-data)。

!!! Warning "使用市价单"
    使用市价单时请阅读[市价单定价](#market-order-pricing)章节。

### 入场价格

#### 入场价格侧

配置项 `entry_pricing.price_side` 定义了机器人买入时查询订单簿的侧向。

下图展示了一个订单簿示例。

``` explanation
...
103
102
101  # ask
-------------Current spread
99   # bid
98
97
...
```

若 `entry_pricing.price_side` 设置为 `"bid"`，则机器人将使用 99 作为入场价格。  
同理，若设置为 `"ask"`，则机器人将使用 101 作为入场价格。

根据订单方向（_做多_/_做空_），这将导致不同的结果。因此我们建议在此配置中使用 `"same"` 或 `"other"`。
这将产生以下定价矩阵：

| 方向 | 订单 | 设置 | 价格 | 是否跨越价差 |
|------ |--------|-----|-----|-----|
| 做多  | 买入  | 卖出价   | 101 | 是 |
| 做多  | 买入  | 买入价   | 99  | 否  |
| 做多  | 买入  | 同侧  | 99  | 否  |
| 做多  | 买入  | 对侧 | 101 | 是 |
| 做空 | 卖出 | 卖出价   | 101 | 否  |
| 做空 | 卖出 | 买入价   | 99  | 是 |
| 做空 | 卖出 | 同侧  | 101 | 否  |
| 做空 | 卖出 | 对侧 | 99  | 是 |

使用订单簿的另一侧通常能保证更快的成交速度，但机器人最终也可能支付比必要金额更高的价格。
即使使用限价买单，也很可能适用吃单费而非挂单费。
此外，价差"对侧"的价格高于订单簿中"买入价"侧的价格，因此该订单的行为类似于市价单（但带有最高价格限制）。

#### 启用订单簿时的入场价格

当启用订单簿进行交易时（`entry_pricing.use_order_book=True`），Freqtrade 会从订单簿中获取 `entry_pricing.order_book_top` 条记录，并使用配置侧（`entry_pricing.price_side`）上指定为 `entry_pricing.order_book_top` 的条目。1 表示订单簿中最顶部的条目，而 2 将使用订单簿中的第二个条目，依此类推。

#### 未启用订单簿时的入场价格

以下部分使用 `side` 作为配置的 `entry_pricing.price_side`（默认为 `"same"`）。

当不使用订单簿时（`entry_pricing.use_order_book=False`），如果报价器的最佳 `side` 价格低于报价器的最后交易价格，Freqtrade 将使用该价格。否则（当 `side` 价格高于 `last` 价格时），它会根据 `entry_pricing.price_last_balance` 计算 `side` 和 `last` 价格之间的比率。

配置参数 `entry_pricing.price_last_balance` 控制此行为。值为 `0.0` 时将使用 `side` 价格，而 `1.0` 将使用 `last` 价格，介于两者之间的值则在卖价和最后价格之间进行插值。

#### 检查市场深度

当启用检查市场深度时（`entry_pricing.check_depth_of_market.enabled=True`），入场信号会根据订单簿每侧的深度（所有数量的总和）进行过滤。

订单簿的 `bid`（买入）侧深度随后除以订单簿的 `ask`（卖出）侧深度，并将得到的 delta 与 `entry_pricing.check_depth_of_market.bids_to_ask_delta` 参数的值进行比较。只有当订单簿 delta 大于或等于配置的 delta 值时，入场订单才会被执行。

!!! Note
    低于 1 的 delta 值表示 `ask`（卖出）订单簿侧深度大于 `bid`（买入）订单簿侧深度，而大于 1 的值则表示相反情况（买入侧深度高于卖出侧深度）。

### 出场价格

#### 出场价格侧

配置项 `exit_pricing.price_side` 定义了机器人平仓时寻找的价差侧。

以下展示一个订单簿：

``` explanation
...
103
102
101  # ask
-------------Current spread
99   # bid
98
97
...
```

如果 `exit_pricing.price_side` 设置为 `"ask"`，那么机器人将使用 101 作为出场价格。  
相应地，如果 `exit_pricing.price_side` 设置为 `"bid"`，那么机器人将使用 99 作为出场价格。

根据订单方向（_做多_/_做空_），这会导致不同的结果。因此，我们建议在此配置中使用 `"same"` 或 `"other"`。
这将产生以下价格矩阵：

| 方向 | 订单 | 设置 | 价格 | 是否跨越价差 |
|------ |--------|-----|-----|-----|
| 做多  | 卖出 | 卖一价   | 101 | 否  |
| 做多  | 卖出 | 买一价   | 99  | 是 |
| 做多  | 卖出 | 同边   | 101 | 否  |
| 做多  | 卖出 | 对边 | 99  | 是 |
| 做空 | 买入  | 卖一价   | 101 | 是 |
| 做空 | 买入  | 买一价   | 99  | 否  |
| 做空 | 买入  | 同边  | 99  | 否  |
| 做空 | 买入  | 对边 | 101 | 是 |

#### 启用订单簿时的退出价格

当启用订单簿退出时（`exit_pricing.use_order_book=True`），Freqtrade 会获取订单簿中的 `exit_pricing.order_book_top` 条目，并使用配置侧（`exit_pricing.price_side`）指定的 `exit_pricing.order_book_top` 条目作为交易退出价格。

1 表示订单簿中最顶层的条目，而 2 将使用订单簿中的第二个条目，依此类推。

#### 未启用订单簿时的退出价格

以下部分使用 `side` 作为配置的 `exit_pricing.price_side`（默认为 `"ask"`）。

当不使用订单簿时（`exit_pricing.use_order_book=False`），如果行情中的最佳 `side` 价格高于行情中的最后成交价（`last`），Freqtrade 将使用该价格。否则（当 `side` 价格低于 `last` 价格时），它会根据 `exit_pricing.price_last_balance` 计算 `side` 和 `last` 价格之间的比率。

`exit_pricing.price_last_balance` 配置参数控制此行为。值为 `0.0` 时将使用 `side` 价格，值为 `1.0` 时将使用最新价格，介于两者之间的值将在 `side` 价格和最新价格之间进行插值计算。

### 市价单定价

使用市价单时，应配置价格使用订单簿的"正确"方向，以实现真实的定价检测。
假设入场和离场均使用市价单，则必须使用类似以下的配置：

``` jsonc
  "order_types": {
    "entry": "market",
    "exit": "market"
    // ...
  },
  "entry_pricing": {
    "price_side": "other",
    // ...
  },
  "exit_pricing":{
    "price_side": "other",
    // ...
  },
```

显然，如果只有一方使用限价单，则可以采用不同的定价组合。