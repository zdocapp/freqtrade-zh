# 配置机器人

Freqtrade 拥有许多可配置的功能和可能性。
默认情况下，这些设置通过配置文件进行配置（见下文）。

## Freqtrade 配置文件

机器人在运行过程中使用一组配置参数，这些参数共同构成了机器人的配置。它通常从文件（Freqtrade 配置文件）中读取其配置。

默认情况下，机器人从当前工作目录中的 `config.json` 文件加载配置。

您可以使用 `-c/--config` 命令行选项指定机器人使用的不同配置文件。

如果您使用 [快速开始](docker_quickstart.md#docker-quick-start) 方法安装机器人，安装脚本应该已经为您创建了默认配置文件 (`config.json`)。

如果默认配置文件未创建，我们建议使用 `freqtrade new-config --config user_data/config.json` 来生成一个基础配置文件。

Freqtrade 配置文件需采用 JSON 格式编写。

除了标准的 JSON 语法外，您还可以在配置文件中使用单行 `// ...` 和多行 `/* ... */` 注释，以及在参数列表中使用尾随逗号。

如果不熟悉 JSON 格式也无需担心——只需用任意编辑器打开配置文件，修改所需参数后保存更改，最后重启机器人；若机器人之前已停止运行，则直接使用修改后的配置重新启动即可。机器人会在启动时验证配置文件的语法，如果编辑过程中出现错误，它会发出警告并指出有问题的行。

### 环境变量

通过环境变量设置 Freqtrade 配置选项。
此方式优先级高于配置文件或策略中的对应值。

环境变量必须添加 `FREQTRADE__` 前缀才能被加载到 freqtrade 配置中。

`__` 作为层级分隔符，因此格式应遵循 `FREQTRADE__{section}__{key}`。
例如，定义环境变量 `export FREQTRADE__STAKE_AMOUNT=200` 将生成 `{stake_amount: 200}`。

更复杂的示例：使用 `export FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>` 来保护交易所密钥。该值将被移至配置文件的 `exchange.key` 部分。
通过此方案，所有配置设置均可通过环境变量进行设置。

请注意：环境变量会覆盖配置文件中的对应设置，但命令行参数的优先级始终最高。

常见示例：

``` bash
FREQTRADE__TELEGRAM__CHAT_ID=<telegramchatid>
FREQTRADE__TELEGRAM__TOKEN=<telegramToken>
FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>
FREQTRADE__EXCHANGE__SECRET=<yourExchangeSecret>
```

Json 列表会被解析为 json 格式 - 因此您可以使用以下方式设置配对列表：

``` bash
export FREQTRADE__EXCHANGE__PAIR_WHITELIST='["BTC/USDT", "ETH/USDT"]'
```

!!! Note
    检测到的环境变量会在启动时记录 - 因此如果您发现某个配置值不符合预期，请确保该值不是从环境变量加载的。

!!! Tip "验证合并结果"
    您可以使用 [show-config 子命令](utils.md#show-config) 查看最终的合并配置。

??? Warning "加载顺序"
    环境变量在初始配置之后加载。因此，您无法通过环境变量提供配置文件路径。请使用 `--config path/to/config.json` 来实现。
    这在某种程度上也适用于 `user_dir`。虽然可以通过环境变量设置用户目录 - 但配置**不会**从该位置加载。

### 多配置文件

机器人可以指定和使用多个配置文件，或者从进程标准输入流读取配置参数。

您可以在 `add_config_files` 中指定其他配置文件。此参数中指定的文件将被加载并与初始配置文件合并。文件路径相对于初始配置文件解析。
这类似于使用多个 `--config` 参数，但在使用上更简单，因为您无需为所有命令指定所有文件。

!!! Tip "验证合并结果"
    您可以使用 [show-config 子命令](utils.md#show-config) 查看最终的合并配置。

!!! Tip "使用多个配置文件保护密钥安全"
    您可以使用第二个包含密钥的配置文件。这样您就可以共享"主"配置文件，同时将 API 密钥保留给自己。
    第二个文件应仅指定您打算覆盖的配置。
    如果某个键出现在多个配置中，则"最后指定的配置"优先（在上面的示例中为 `config-private.json`）。

    For one-off commands, you can also use the below syntax by specifying multiple "--config" parameters.

    ``` bash
    freqtrade trade --config user_data/config1.json --config user_data/config-private.json <...>
    ```

    The below is equivalent to the example above - but having 2 configuration files in the configuration, for easier reuse.

    ``` json title="user_data/config.json"
    "add_config_files": [
        "config1.json",
        "config-private.json"
    ]
    ```

    ``` bash
    freqtrade trade --config user_data/config.json <...>
    ```

??? Note "配置冲突处理"
    如果相同的配置设置同时出现在 `config.json` 和 `config-import.json` 中，则父配置优先。
    在以下情况下，合并后的 `max_open_trades` 将为 3 - 因为可重用的"导入"配置覆盖了此键。

    ``` json title="user_data/config.json"
    {
        "max_open_trades": 3,
        "stake_currency": "USDT",
        "add_config_files": [
            "config-import.json"
        ]
    }
    ```

    ``` json title="user_data/config-import.json"
    {
        "max_open_trades": 10,
        "stake_amount": "unlimited",
    }
    ```

    Resulting combined configuration:

    ``` json title="Result"
    {
        "max_open_trades": 3,
        "stake_currency": "USDT",
        "stake_amount": "unlimited"
    }
    ```

    If multiple files are in the `add_config_files` section, then they will be assumed to be at identical levels, having the last occurrence override the earlier config (unless a parent already defined such a key).

## 编辑器自动补全与验证

如果您使用的编辑器支持 JSON 模式，可以通过在配置文件顶部添加以下行，使用 Freqtrade 提供的模式来获得配置文件的自动补全和验证功能：

``` json
{
    "$schema": "https://schema.freqtrade.io/schema.json",
}
```

??? Note "开发版本"
    开发版本的模式可通过 `https://schema.freqtrade.io/schema_dev.json` 获取 - 但我们建议坚持使用稳定版本以获得最佳体验。

## 配置参数

下表将列出所有可用的配置参数。

Freqtrade 也可以通过命令行（CLI）参数加载许多选项（详情请查看命令的 `--help` 输出）。

### 配置选项优先级

所有选项的优先级如下：

* CLI 参数会覆盖任何其他选项
* [环境变量](#environment-variables)
* 配置文件按顺序使用（最后一个文件生效）并覆盖策略配置。
* 仅当未通过配置或命令行参数设置时，才会使用策略配置。这些选项在下表中标记为[策略覆盖](#parameters-in-the-strategy)。

### 参数表

必需参数标记为 **必填**，这意味着必须通过其中一种可能的方式进行设置。

| 参数 | 描述 |
|------------|-------------|
| `max_open_trades` | **必需。** 您的机器人允许持有的未平仓交易数量。每个交易对只能有一个未平仓交易，因此您的配对列表长度是另一个可能适用的限制。如果设置为 -1，则忽略此限制（即可能无限数量的未平仓交易，受配对列表限制）。[更多信息如下](#configuring-amount-per-trade)。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 正整数或 -1。
| `stake_currency` | **必需。** 用于交易的加密货币。<br> **数据类型：** 字符串
| `stake_amount` | **必需。** 您的机器人每次交易将使用的加密货币数量。设置为 `"unlimited"` 允许机器人使用所有可用余额。[更多信息如下](#configuring-amount-per-trade)。<br> **数据类型：** 正浮点数或 `"unlimited"`。
| `tradable_balance_ratio` | 机器人允许交易的总账户余额比率。[更多信息如下](#configuring-amount-per-trade)。<br>*默认值为 `0.99` (99%)。*<br> **数据类型：** 介于 `0.1` 和 `1.0` 之间的正浮点数。
| `available_capital` | 机器人的可用起始资金。在同一交易所账户上运行多个机器人时很有用。[更多信息如下](#configuring-amount-per-trade)。<br> **数据类型：** 正浮点数。
| `amend_last_stake_amount` | 必要时使用减少的最后持仓金额。[更多信息如下](#configuring-amount-per-trade)。<br>*默认值为 `false`。* <br> **数据类型：** 布尔值
| `last_stake_amount_min_ratio` | 定义必须保留和执行的最小持仓金额。仅适用于当最后持仓金额被修正为减少值时（即如果 `amend_last_stake_amount` 设置为 `true`）。[更多信息如下](#configuring-amount-per-trade)。<br>*默认值为 `0.5`。* <br> **数据类型：** 浮点数（比率）
| `amount_reserve_percent` | 在最小交易对持仓金额中保留一定金额。机器人在计算最小交易对持仓金额时将保留 `amount_reserve_percent` + 止损值，以避免可能的交易拒绝。<br>*默认值为 `0.05` (5%)。* <br> **数据类型：** 正浮点数（比率）。
| `timeframe` | 要使用的时间框架（例如 `1m`、`5m`、`15m`、`30m`、`1h` ...）。通常在配置中缺失，并在策略中指定。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 字符串
| `fiat_display_currency` | 用于显示利润的法币。[更多信息如下](#what-values-can-be-used-for-fiat_display_currency)。<br> **数据类型：** 字符串
| `dry_run` | **必需。** 定义机器人必须处于模拟运行模式还是生产模式。<br>*默认值为 `true`。* <br> **数据类型：** 布尔值
| `dry_run_wallet` | 定义在模拟运行模式下机器人使用的模拟钱包的起始金额（以持仓货币计）。[更多信息如下](#dry-run-wallet)<br>*默认值为 `1000`。* <br> **数据类型：** 浮点数或字典
| `cancel_open_orders_on_exit` | 当发出 `/stop` RPC 命令、按下 `Ctrl+C` 或机器人意外死亡时取消未成交订单。设置为 `true` 时，这允许您使用 `/stop` 在市场崩盘时取消未成交和部分成交的订单。不影响未平仓头寸。<br>*默认值为 `false`。* <br> **数据类型：** 布尔值
| `process_only_new_candles` | 仅当新 K 线到达时启用指标处理。如果为 false，每个循环都会填充指标，这意味着同一根 K 线会被处理多次，增加系统负载，但如果您的策略依赖于即时数据而不仅仅是 K 线数据，则可能有用。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `true`。*  <br> **数据类型：** 布尔值
| `minimal_roi` | **必需。** 设置机器人用于退出交易的比率阈值。[更多信息如下](#understand-minimal_roi)。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 字典
| `stoploss` |  **必需。** 机器人使用的止损比率值。更多细节请参阅[止损文档](stoploss.md)。[策略覆盖](#parameters-in-the-strategy)。  <br> **数据类型：** 浮点数（比率）
| `trailing_stop` | 启用追踪止损（基于配置文件或策略文件中的 `stoploss`）。更多细节请参阅[止损文档](stoploss.md#trailing-stop-loss)。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 布尔值
| `trailing_stop_positive` | 一旦达到利润后更改止损。更多细节请参阅[止损文档](stoploss.md#trailing-stop-loss-different-positive-loss)。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 浮点数
| `trailing_stop_positive_offset` | 何时应用 `trailing_stop_positive` 的偏移量。应为正值的百分比。更多细节请参阅[止损文档](stoploss.md#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset)。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `0.0`（无偏移）。* <br> **数据类型：** 浮点数
| `trailing_only_offset_is_reached` | 仅当达到偏移量时应用追踪止损。[止损文档](stoploss.md)。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。*  <br> **数据类型：** 布尔值
| `fee` | 回测/模拟运行期间使用的费用。通常不应配置，这将使 freqtrade 回退到交易所的默认费用。设置为比率（例如 0.001 = 0.1%）。每笔交易费用应用两次，一次在买入时，一次在卖出时。<br> **数据类型：** 浮点数（比率）
| `futures_funding_rate` | 当交易所无法提供历史资金费率时使用的用户指定资金费率。这不会覆盖真实的历史费率。除非您正在测试特定币种并且了解资金费率将如何影响 freqtrade 的利润计算，否则建议将其设置为 0。[更多信息在此](leverage.md#unavailable-funding-rates) <br>*默认值为 `None`。*<br> **数据类型：** 浮点数
| `trading_mode` | 指定您是要常规交易、杠杆交易还是交易价格源自匹配加密货币价格的合约。[杠杆文档](leverage.md)。<br>*默认值为 `"spot"`。*  <br> **数据类型：** 字符串
| `margin_mode` | 当使用杠杆交易时，这决定交易者拥有的抵押品是共享还是隔离到每个交易对[杠杆文档](leverage.md)。<br> **数据类型：** 字符串
| `liquidation_buffer` | 一个比率，指定在清算价格和止损之间放置多大的安全网，以防止头寸达到清算价格[杠杆文档](leverage.md)。<br>*默认值为 `0.05`。*  <br> **数据类型：** 浮点数
| | **未成交超时**
| `unfilledtimeout.entry` | **必需。** 机器人将等待未成交的入场订单完成的时长（以分钟或秒计），之后订单将被取消。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 整数
| `unfilledtimeout.exit` | **必需。** 机器人将等待未成交的出场订单完成的时长（以分钟或秒计），之后订单将被取消并在当前（新）价格重复，只要有信号。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 整数
| `unfilledtimeout.unit` | 在 unfilledtimeout 设置中使用的单位。注意：如果您将 unfilledtimeout.unit 设置为 "seconds"，则 "internals.process_throttle_secs" 必须小于或等于超时时间[策略覆盖](#parameters-in-the-strategy)。<br> *默认值为 `"minutes"`。* <br> **数据类型：** 字符串
| `unfilledtimeout.exit_timeout_count` | 出场订单可以超时多少次。一旦达到此超时次数，将触发紧急退出。0 表示禁用并允许无限次订单取消。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `0`。* <br> **数据类型：** 整数
| | **定价**
| `entry_pricing.price_side` | 选择机器人应查看的价差侧以获取入场价格。[更多信息如下](#entry-price)。<br> *默认值为 `"same"`。* <br> **数据类型：** 字符串（`ask`、`bid`、`same` 或 `other` 之一）。
| `entry_pricing.price_last_balance` | **必需。** 插值买入价格。更多信息[如下](#entry-price-without-orderbook-enabled)。
| `entry_pricing.use_order_book` | 启用使用[订单簿入场](#entry-price-with-orderbook-enabled)中的价格入场。<br> *默认值为 `true`。*<br> **数据类型：** 布尔值
| `entry_pricing.order_book_top` | 机器人将使用订单簿"price_side"中的前 N 个价格入场。例如，值为 2 将允许机器人选择[订单簿入场](#entry-price-with-orderbook-enabled)中的第二个条目。<br>*默认值为 `1`。*  <br> **数据类型：** 正整数
| `entry_pricing. check_depth_of_market.enabled` | 如果订单簿中买单和卖单的差异满足条件，则不入场。[检查市场深度](#check-depth-of-market)。<br>*默认值为 `false`。* <br> **数据类型：** 布尔值
| `entry_pricing. check_depth_of_market.bids_to_ask_delta` | 订单簿中发现的买单和卖单的差异比率。低于 1 的值表示卖单规模更大，而大于 1 的值表示买单规模更高。[检查市场深度](#check-depth-of-market) <br> *默认值为 `0`。*  <br> **数据类型：** 浮点数（比率）
| `exit_pricing.price_side` | 选择机器人应查看的价差侧以获取出场价格。[更多信息如下](#exit-price-side)。<br> *默认值为 `"same"`。* <br> **数据类型：** 字符串（`ask`、`bid`、`same` 或 `other` 之一）。
| `exit_pricing.price_last_balance` | 插值出场价格。更多信息[如下](#exit-price-without-orderbook-enabled)。
| `exit_pricing.use_order_book` | 启用使用[订单簿出场](#exit-price-with-orderbook-enabled)退出未平仓交易。<br> *默认值为 `true`。*<br> **数据类型：** 布尔值
| `exit_pricing.order_book_top` | 机器人将使用订单簿"price_side"中的前 N 个价格出场。例如，值为 2 将允许机器人选择[订单簿出场](#exit-price-with-orderbook-enabled)中的第二个卖出价格。<br>*默认值为 `1`。* <br> **数据类型：** 正整数
| `custom_price_max_distance_ratio` | 配置当前价格与自定义入场或出场价格之间的最大距离比率。<br>*默认值为 `0.02` (2%)。*<br> **数据类型：** 正浮点数
| | **订单/信号处理**
| `use_exit_signal` | 使用策略产生的出场信号以及 `minimal_roi`。<br>将此设置为 false 将禁用 `"exit_long"` 和 `"exit_short"` 列的使用。不影响其他退出方法（止损、ROI、回调）。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `true`。* <br> **数据类型：** 布尔值
| `exit_profit_only` | 在做出出场决定之前，等待机器人达到 `exit_profit_offset`。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。* <br> **数据类型：** 布尔值
| `exit_profit_offset` | 出场信号仅在此值之上激活。仅在与 `exit_profit_only=True` 结合使用时激活。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `0.0`。* <br> **数据类型：** 浮点数（比率）
| `ignore_roi_if_entry_signal` | 如果入场信号仍然活跃，则不退出。此设置优先于 `minimal_roi` 和 `use_exit_signal`。[策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。* <br> **数据类型：** 布尔值
| `ignore_buying_expired_candle_after` | 指定买入信号不再使用的秒数。<br> **数据类型：** 整数
| `order_types` | 根据操作（`"entry"`、`"exit"`、`"stoploss"`、`"stoploss_on_exchange"`）配置订单类型。[更多信息如下](#understand-order_types)。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 字典
| `order_time_in_force` | 配置入场和出场订单的时效。[更多信息如下](#understand-order_time_in_force)。[策略覆盖](#parameters-in-the-strategy)。<br> **数据类型：** 字典
| `position_adjustment_enable` | 启用策略使用头寸调整（额外买入或卖出）。[更多信息在此](strategy-callbacks.md#adjust-trade-position)。<br> [策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `false`。*<br> **数据类型：** 布尔值
| `max_entry_position_adjustment` | 每个未平仓交易在第一个入场订单之上的最大额外订单数。设置为 `-1` 表示无限额外订单。[更多信息在此](strategy-callbacks.md#adjust-trade-position)。<br> [策略覆盖](#parameters-in-the-strategy)。<br>*默认值为 `-1`。*<br> **数据类型：** 正整数或 -1
| | **交易所**
| `exchange.name` | **必需。** 要使用的交易所类名称。<br> **数据类型：** 字符串
| `exchange.key` | 用于交易所的 API 密钥。仅在生产模式下需要。<br>**保密，请勿公开披露。** <br> **数据类型：** 字符串
| `exchange.secret` | 用于交易所的 API 密钥。仅在生产模式下需要。<br>**保密，请勿公开披露。** <br> **数据类型：** 字符串
| `exchange.password` | 用于交易所的 API 密码。仅在生产模式下需要，并且适用于在 API 请求中使用密码的交易所。<br>**保密，请勿公开披露。** <br> **数据类型：** 字符串
| `exchange.uid` | 用于交易所的 API uid。仅在生产模式下需要，并且适用于在 API 请求中使用 uid 的交易所。<br>**保密，请勿公开披露。** <br> **数据类型：** 字符串
| `exchange.pair_whitelist` | 机器人用于交易和在回测期间检查潜在交易的交易对列表。支持正则表达式交易对，如 `.*/BTC`。不被 VolumePairList 使用。[更多信息](plugins.md#pairlists-and-pairlist-handlers)。<br> **数据类型：** 列表
| `exchange.pair_blacklist` | 机器人必须绝对避免用于交易和回测的交易对列表。[更多信息](plugins.md#pairlists-and-pairlist-handlers)。<br> **数据类型：** 列表
| `exchange.ccxt_config` | 传递给两个 ccxt 实例（同步和异步）的附加 CCXT 参数。这通常是附加 ccxt 配置的正确位置。参数可能因交易所而异，并在 [ccxt 文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation) 中记录。请避免在此处添加交易所密钥（使用专用字段代替），因为它们可能包含在日志中。<br> **数据类型：** 字典
| `exchange.ccxt_sync_config` | 传递给常规（同步）ccxt 实例的附加 CCXT 参数。参数可能因交易所而异，并在 [ccxt 文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation) 中记录。<br> **数据类型：** 字典
| `exchange.ccxt_async_config` | 传递给异步 ccxt 实例的附加 CCXT 参数。参数可能因交易所而异，并在 [ccxt 文档](https://docs.ccxt.com/#/README?id=overriding-exchange-properties-upon-instantiation) 中记录。<br> **数据类型：** 字典
| `exchange.enable_ws` | 启用交易所的 WebSocket 使用。<br>[更多信息](#consuming-exchange-websockets)。<br>*默认值为 `true`。* <br> **数据类型：** 布尔值
| `exchange.markets_refresh_interval` | 重新加载市场的间隔时间（以分钟计）。<br>*默认值为 `60` 分钟。* <br> **数据类型：** 正整数
| `exchange.skip_open_order_update` | 如果交易所导致问题，则在启动时跳过未成交订单更新。仅在实际交易条件下相关。<br>*默认值为 `false`*<br> **数据类型：** 布尔值
| `exchange.unknown_fee_rate` | 计算交易费用时使用的回退值。这对于费用以非交易货币计价的交易所可能有用。此处提供的值将与"费用成本"相乘。<br>*默认值为 `None`<br> **数据类型：** 浮点数
| `exchange.log_responses` | 记录相关的交易所响应。仅用于调试模式 - 请谨慎使用。<br>*默认值为 `false`*<br> **数据类型：** 布尔值
| `exchange.only_from_ccxt` | 防止从 data.binance.vision 下载数据。将此保留为 false 可以大大加快下载速度，但如果站点不可用可能会出现问题。<br>*默认值为 `false`*<br> **数据类型：** 布尔值
| `experimental.block_bad_exchanges` | 阻止已知不与 freqtrade 兼容的交易所。除非您想测试该交易所现在是否工作，否则保留默认值。<br>*默认值为 `true`。* <br> **数据类型：** 布尔值
| | **插件**
| `pairlists` | 定义一个或多个要使用的配对列表。[更多信息](plugins.md#pairlists-and-pairlist-handlers)。<br>*默认值为 `StaticPairList`。*  <br> **数据类型：** 字典列表
| | **Telegram**
| `telegram.enabled` | 启用 Telegram 使用。<br> **数据类型：** 布尔值
| `telegram.token` | 您的 Telegram 机器人令牌。仅当 `telegram.enabled` 为 `true` 时需要。<br>**保密，请勿公开披露。** <br> **数据类型：** 字符串
| `telegram.chat_id` | 您的个人 Telegram 账户 ID。仅当 `telegram.enabled` 为 `true` 时需要。<br>**保密，请勿公开披露。** <br> **数据类型：** 字符串
| `telegram.balance_dust_level` | 粉尘级别（以持仓货币计）- 余额低于此值的货币将不会在 `/balance` 中显示。<br> **数据类型：** 浮点数
| `telegram.reload` | 允许在 telegram 消息上显示"重新加载"按钮。<br>*默认值为 `true`。<br> **数据类型：** 布尔值
| `telegram.notification_settings.*` | 详细的通知设置。有关详细信息，请参阅 [telegram 文档](telegram-usage.md)。<br> **数据类型：** 字典
| `telegram.allow_custom_messages` | 启用通过 dataprovider.send_msg() 函数从策略发送 Telegram 消息。<br> **数据类型：** 布尔值
| | **Webhook**
| `webhook.enabled` | 启用 Webhook 通知使用<br> **数据类型：** 布尔值
| `webhook.url` | Webhook 的 URL。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.entry` | 入场时发送的有效载荷。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.entry_cancel` | 入场订单取消时发送的有效载荷。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.entry_fill` | 入场订单成交时发送的有效载荷。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.exit` | 出场时发送的有效载荷。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.exit_cancel` | 出场订单取消时发送的有效载荷。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.exit_fill` | 出场订单成交时发送的有效载荷。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.status` | 状态调用时发送的有效载荷。仅当 `webhook.enabled` 为 `true` 时需要。有关更多详细信息，请参阅 [webhook 文档](webhook-config.md)。<br> **数据类型：** 字符串
| `webhook.allow_custom_messages` | 启用通过 dataprovider.send_msg() 函数从策略发送 Webhook 消息。<br> **数据类型：** 布尔值
| | **REST API / FreqUI / 生产者-消费者**
| `api_server.enabled` | 启用 API 服务器使用。有关更多详细信息，请参阅 [API 服务器文档](rest-api.md)。<br> **数据类型：** 布尔值
| `api_server.listen_ip_address` | 绑定 IP 地址。有关更多详细信息，请参阅 [API 服务器文档](rest-api.md)。<br> **数据类型：** IPv4
| `api_server.listen_port` | 绑定端口。有关更多详细信息，请参阅 [API 服务器文档](rest-api.md)。<br>**数据类型：** 1024 到 65535 之间的整数
| `api_server.verbosity` | 日志详细程度。`info` 将打印所有 RPC 调用，而 "error" 仅显示错误。<br>**数据类型：** 枚举，`info` 或 `error`。默认为 `info`。
| `api_server.username` | API 服务器的用户名。有关更多详细信息，请参阅 [API 服务器文档](rest-api.md)。<br>**保密，请勿公开披露。**<br> **数据类型：** 字符串
| `api_server.password` | API 服务器的密码。有关更多详细信息，请参阅 [API 服务器文档](rest-api.md)。<br>**保密，请勿公开披露。**<br> **数据类型：** 字符串
| `api_server.ws_token` | 消息 WebSocket 的 API 令牌。有关更多详细信息，请参阅 [API 服务器文档](rest-api.md)。<br>**保密，请勿公开披露。** <br> **数据类型：** 字符串
| `bot_name` | 机器人名称。通过 API 传递给客户端 - 可用于区分/命名机器人。<br> *默认值为 `freqtrade`*<br> **数据类型：** 字符串
| `external_message_consumer` | 启用[生产者/消费者模式](producer-consumer.md)以获取更多详细信息。<br> **数据类型：** 字典
| | **其他**
| `initial_state` | 定义初始应用程序状态。如果设置为 stopped，则必须通过 `/start` RPC 命令明确启动机器人。<br>*默认值为 `stopped`。* <br> **数据类型：** 枚举，`running`、`paused` 或 `stopped` 之一
| `force_entry_enable` | 启用强制交易入场的 RPC 命令。更多信息如下。<br> **数据类型：** 布尔值
| `disable_dataframe_checks` | 禁用对从策略方法返回的 OHLCV 数据帧的正确性检查。仅在有意更改数据帧并了解自己在做什么时使用。[策略覆盖](#parameters-in-the-strategy)。<br> *默认值为 `False`*。 <br> **数据类型：** 布尔值
| `internals.process_throttle_secs` | 设置进程节流，或一次机器人迭代循环的最小循环持续时间。值以秒为单位。<br>*默认值为 `5` 秒。* <br> **数据类型：** 正整数
| `internals.heartbeat_interval` | 每 N 秒打印一次心跳消息。设置为 0 以禁用心跳消息。<br>*默认值为 `60` 秒。* <br> **数据类型：** 正整数或 0
| `internals.sd_notify` | 启用使用 sd_notify 协议来告知 systemd 服务管理器有关机器人状态的变化并发出保活 ping。有关更多详细信息，请参阅[此处](advanced-setup.md#configure-the-bot-running-as-a-systemd-service)。<br> **数据类型：** 布尔值
| `strategy` | **必需** 定义要使用的策略类。建议通过 `--strategy NAME` 设置。<br> **数据类型：** 类名
| `strategy_path` | 添加额外的策略查找路径（必须是目录）。<br> **数据类型：** 字符串
| `recursive_strategy_search` | 设置为 `true` 以递归搜索 `user_data/strategies` 内的子目录以查找策略。<br> **数据类型：** 布尔值
| `user_data_dir` | 包含用户数据的目录。<br> *默认值为 `./user_data/`*。 <br> **数据类型：** 字符串
| `db_url` | 声明要使用的数据库 URL。注意：如果 `dry_run` 为 `true`，则默认为 `sqlite:///tradesv3.dryrun.sqlite`，对于生产实例默认为 `sqlite:///tradesv3.sqlite`。<br> **数据类型：** 字符串，SQLAlchemy 连接字符串
| `logfile` | 指定日志文件名。使用滚动策略进行日志文件轮换，最多 10 个文件，每个文件限制为 1MB。<br> **数据类型：** 字符串
| `add_config_files` | 附加配置文件。这些文件将被加载并与当前配置文件合并。文件相对于初始文件解析。<br> *默认值为 `[]`*。 <br> **数据类型：** 字符串列表
| `dataformat_ohlcv` | 用于存储历史 K 线（OHLCV）数据的数据格式。<br> *默认值为 `feather`*。 <br> **数据类型：** 字符串
| `dataformat_trades` | 用于存储历史交易数据的数据格式。<br> *默认值为 `feather`*。 <br> **数据类型：** 字符串
| `reduce_df_footprint` | 将所有数值列重新转换为 float32/int32，目的是减少 ram/磁盘使用量（并减少回测/超参数优化和 FreqAI 中的训练/推理时间）。<br> **数据类型：** 布尔值。<br> 默认值：`False`。
| `log_config` | 包含 python 日志记录日志配置的字典。[更多信息](advanced-setup.md#advanced-logging) <br> **数据类型：** 字典。<br> 默认值：`FtRichHandler`

### 策略中的参数

以下参数可在配置文件或策略中设置。
配置文件中的设置值始终会覆盖策略中的设置值。

* `minimal_roi`
* `timeframe`
* `stoploss`
* `max_open_trades`
* `trailing_stop`
* `trailing_stop_positive`
* `trailing_stop_positive_offset`
* `trailing_only_offset_is_reached`
* `use_custom_stoploss`
* `process_only_new_candles`
* `order_types`
* `order_time_in_force`
* `unfilledtimeout`
* `disable_dataframe_checks`
* `use_exit_signal`
* `exit_profit_only`
* `exit_profit_offset`
* `ignore_roi_if_entry_signal`
* `ignore_buying_expired_candle_after`
* `position_adjustment_enable`
* `max_entry_position_adjustment`

### 配置每笔交易金额

有多种方法可以配置机器人将使用多少权益货币来进入交易。所有方法都遵循如下所述的[可用余额配置](#tradable-balance)。

#### 最小交易金额

最小交易金额取决于交易所和交易对，通常会在交易所的支持页面中列出。

假设 XRP/USD 的最小可交易量为 20 XRP（由交易所规定），且价格为 0.6\$，则购买该交易对的最小权益金额为 `20 * 0.6 ~= 12`。
该交易所还对美元有限制——所有订单必须大于 10\$——但在本例中不适用。

为确保安全执行，freqtrade 不会允许以 10.1 美元的交易金额进行买入，而是会确保有足够空间在该交易对下方设置止损位（外加一个由 `amount_reserve_percent` 定义的偏移量，默认为 5%）。

在保留 5% 的情况下，最低交易金额约为 12.6 美元（`12 * (1 + 0.05)`）。如果在此基础上再考虑 10% 的止损，最终金额将约为 14 美元（`12.6 / (1 - 0.1)`）。

为限制大止损值情况下的计算，计算得出的最低交易金额限制绝不会超过实际限制的 50%。

!!! Warning
    由于交易所的限制通常稳定且不常更新，某些交易对可能显示相当高的最低限制，这仅仅是因为自交易所上次调整限制以来价格大幅上涨。Freqtrade 会将交易金额调整至该值，除非它比计算/期望的交易金额高出 30% 以上——在这种情况下交易将被拒绝。

#### 模拟钱包

在模拟运行模式下，机器人将使用模拟钱包来执行交易。该钱包的初始余额由 `dry_run_wallet` 定义（默认为 1000）。对于更复杂的场景，您还可以为 `dry_run_wallet` 分配一个字典来定义每种货币的初始余额。

```json
"dry_run_wallet": {
    "BTC": 0.01,
    "ETH": 2,
    "USDT": 1000
}
```

命令行选项（`--dry-run-wallet`）可用于覆盖配置值，但仅适用于浮动值，不适用于字典。如需使用字典，请调整配置文件。

!!! Note
    非质押货币的余额不会用于交易，但会显示为钱包余额的一部分。
    在交叉保证金交易所中，钱包余额可能用于计算可用交易保证金。

#### 可交易余额

默认情况下，机器人假定`总金额 - 1%`可供其支配，当使用[动态持仓金额](#dynamic-stake-amount)时，它会将总余额按每笔交易分成`max_open_trades`个份额。
Freqtrade 会预留 1% 作为开仓时可能产生的手续费，因此默认不会动用这部分资金。

您可以通过`tradable_balance_ratio`设置来配置"不动用"的金额比例。

例如，如果您在交易所钱包中有 10 ETH 可用，且`tradable_balance_ratio=0.5`（即 50%），则机器人将最多使用 5 ETH 进行交易，并将其视为可用余额。其余钱包余额不会被交易动用。

!!! Danger
    在同一个账户上运行多个机器人时**不应**使用此设置。请改用[机器人可用资金](#assign-available-capital)。

!!! Warning
    `tradable_balance_ratio` 设置适用于当前余额（可用余额 + 交易占用资金）。因此，假设初始余额为 1000，配置 `tradable_balance_ratio=0.99` 并不能保证交易所始终保留 10 个货币单位的可用资金。例如，若总余额因连续亏损或提现降至 500，可用金额可能减少至 5 个单位。

#### 分配可用资金

若要在同一交易所账户上运行多个机器人并充分利用复利收益，您需要将每个机器人的初始余额限制在特定金额。
这可以通过设置 `available_capital` 为期望的初始余额来实现。

假设您的账户有 10000 USDT，并希望在该交易所运行 2 种不同策略。
您可设置 `available_capital=5000`——使每个机器人获得 5000 USDT 的初始资金。
随后机器人会将此初始余额平均分配到 `max_open_trades` 个资金池中。
盈利交易将提升该机器人的持仓规模——而不会影响其他机器人的持仓规模。

调整 `available_capital` 需要重新加载配置才能生效。调整 `available_capital` 会添加先前 `available_capital` 与新 `available_capital` 之间的差额。在有交易未平仓时减少可用资金不会退出这些交易。差额将在交易结束时返还至钱包。此操作的结果取决于调整资金与平仓期间的价格变动情况。

!!! Warning "与 `tradable_balance_ratio` 不兼容"
    设置此选项将覆盖任何 `tradable_balance_ratio` 的配置。

#### 调整最后仓位金额

假设我们拥有 1000 USDT 的可交易余额，`stake_amount=400`，且 `max_open_trades=3`。
机器人将开设 2 笔交易，但无法填满最后一个交易仓位，因为所需的 400 USDT 已不可用（已有 800 USDT 被其他交易占用）。

为解决此问题，可将选项 `amend_last_stake_amount` 设置为 `True`，这将允许机器人将仓位金额减少至可用余额以填满最后一个交易仓位。

在上例中这意味着：

* 交易1：400 USDT
* 交易2：400 USDT
* 交易3：200 USDT

!!! Note
    此选项仅适用于[固定仓位金额](#固定仓位金额)——因为[动态仓位金额](#动态仓位金额)会平均分配余额。

!!! Note
    最小最终持仓金额可通过 `last_stake_amount_min_ratio` 参数进行配置，默认值为 0.5（50%）。这意味着实际使用的最小持仓金额为 `stake_amount * 0.5`。该设置可避免持仓金额过低，接近交易对的最小可交易量而被交易所拒绝。

#### 固定持仓金额

`stake_amount` 配置项用于静态设置机器人每笔交易使用的标价货币金额。

最小配置值为 0.0001，但请根据您使用的标价货币核对交易所的最小交易限额以避免问题。

该设置需与 `max_open_trades` 配合使用。最大交易占用资金为 `stake_amount * max_open_trades`。
例如，假设配置为 `max_open_trades=3` 和 `stake_amount=0.05`，机器人最多将使用（0.05 BTC × 3）= 0.15 BTC。

!!! Note
    此设置遵循[可用余额配置](#tradable-balance)。

#### 动态持仓金额

您也可以使用动态持仓金额模式，该模式将利用交易所可用余额，并平均分配到允许的交易数量（`max_open_trades`）上。

配置此模式需设置 `stake_amount="unlimited"`。同时建议设置 `tradable_balance_ratio=0.99`（99%）以保留最低余额应对可能产生的手续费。

此时单笔交易金额计算公式为：

```python
currency_balance / (max_open_trades - current_open_trades)
```

为了让机器人交易您账户中所有可用的 `stake_currency`（减去 `tradable_balance_ratio`），请设置

```json
"stake_amount" : "unlimited",
"tradable_balance_ratio": 0.99,
```

!!! Tip "复利收益"
    此配置将允许根据机器人的表现增加/减少持仓（如果机器人亏损则降低持仓，如果机器人有盈利记录则提高持仓，因为可用余额更高），并将实现收益复利。

!!! Note "使用模拟交易模式时"
    当在模拟交易、回测或超参数优化中结合使用 `"stake_amount" : "unlimited"` 时，将模拟从 `dry_run_wallet` 的初始金额开始演变的余额。
    因此，务必将 `dry_run_wallet` 设置为合理值（例如 BTC 设为 0.05 或 0.01，USDT 设为 1000 或 100），否则可能会模拟一次性使用 100 BTC（或更多）或 0.05 USDT（或更少）进行交易——这可能与您的实际可用余额不符，或者低于交易所对标的货币订单金额的最低限制。

#### 带仓位调整的动态持仓金额

当您想要在无限持仓模式下使用仓位调整时，还必须实现 `custom_stake_amount` 以根据您的策略返回一个值。
典型值范围应为建议持仓的 25% - 50%，但很大程度上取决于您的策略以及您希望留作仓位调整缓冲的资金量。

例如，如果你的仓位调整假设可以使用相同的持仓金额进行2次额外买入，那么你的缓冲金额应为初始提议的无限制持仓金额的66.6667%。

或者另一个例子，如果你的仓位调整假设可以使用3倍原始持仓金额进行1次额外买入，那么`custom_stake_amount`应返回提议持仓金额的25%，并为后续可能的仓位调整保留75%。

--8<-- "includes/pricing.md"

## 进一步配置详情

### 理解 minimal_roi

`minimal_roi`配置参数是一个JSON对象，其中键表示以分钟为单位的持续时间，值表示最小投资回报率比率。
参见以下示例：

```json
"minimal_roi": {
    "40": 0.0,    # Exit after 40 minutes if the profit is not negative
    "30": 0.01,   # Exit after 30 minutes if there is at least 1% profit
    "20": 0.02,   # Exit after 20 minutes if there is at least 2% profit
    "0":  0.04    # Exit immediately if there is at least 4% profit
},
```

大多数策略文件已包含最优的`minimal_roi`值。
该参数可在策略文件或配置文件中设置。如果在配置文件中使用，它将覆盖策略文件中的`minimal_roi`值。
如果在策略文件或配置文件中均未设置，则默认使用1000% `{"0": 10}`，且除非你的交易产生1000%的利润，否则最小投资回报率功能将被禁用。

!!! Note "在特定时间后强制退出的特殊情况"
    使用`"<N>": -1`作为投资回报率时会出现特殊情况。这将强制机器人在N分钟后退出交易，无论盈亏如何，因此代表有时限的强制退出。

### 理解 force_entry_enable

`force_entry_enable` 配置参数允许通过 Telegram 和 REST API 使用强制入场（`/forcelong`、`/forceshort`）命令。
出于安全考虑，该功能默认处于禁用状态，如果启用，freqtrade 将在启动时显示警告信息。
例如，您可以向机器人发送 `/forceenter ETH/BTC`，这将导致 freqtrade 买入该交易对并持有，直到出现常规退出信号（ROI、止损、`/forceexit`）。

这对某些策略可能具有风险，请谨慎使用。

有关使用详情，请参阅 [Telegram 文档](telegram-usage.md)。

### 忽略过期蜡烛线

当使用较大时间框架（例如 1 小时或更长）且 `max_open_trades` 值较低时，一旦有交易仓位可用，系统会立即处理最新的蜡烛线。在处理最新蜡烛线时，可能会导致不希望在该蜡烛线上使用买入信号的情况。例如，当策略中使用交叉条件时，交叉点可能已过去太久，不适合基于它开仓。

在此类情况下，您可以通过将 `ignore_buying_expired_candle_after` 设置为正数来启用忽略超时蜡烛线的功能，该数值表示买入信号过期后的秒数。

例如，如果您的策略使用1小时时间框架，并且只想在新K线出现的前5分钟内进行买入，您可以在策略中添加以下配置：

``` json
  {
    //...
    "ignore_buying_expired_candle_after": 300,
    // ...
  }
```

!!! Note
    此设置会在每个新K线时重置，因此不会阻止粘性信号在它们活跃的第2或第3根K线上执行。最好对买入信号使用"触发器"选择器，这类信号仅在一根K线内有效。

### 理解 order_types

`order_types` 配置参数将操作（`entry`、`exit`、`stoploss`、`emergency_exit`、`force_exit`、`force_entry`）映射到订单类型（`market`、`limit`等），同时配置交易所上的止损单并定义交易所止损单的更新间隔（以秒为单位）。

这允许使用限价单进行入场，使用限价单进行离场，以及使用市价单创建止损单。
它还允许设置
"交易所上的止损"，这意味着一旦买单成交，止损单将立即被放置。

配置文件中设置的 `order_types` 会整体覆盖策略中设置的值，因此您需要在一个地方配置整个 `order_types` 字典。

如果配置了此参数，则必须包含以下4个值（`entry`、`exit`、`stoploss` 和 `stoploss_on_exchange`），否则机器人将无法启动。

有关 (`emergency_exit`,`force_exit`, `force_entry`, `stoploss_on_exchange`,`stoploss_on_exchange_interval`,`stoploss_on_exchange_limit_ratio`) 的详细信息，请参阅止损文档 [交易所止损](stoploss.md)

策略语法：

```python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": False,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99,
}
```

配置：

```json
"order_types": {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": false,
    "stoploss_on_exchange_interval": 60
}
```

!!! Note "市价单支持"
    并非所有交易所都支持"市价"订单。
    如果你的交易所不支持市价订单，将显示以下消息：
    `"交易所 <yourexchange> 不支持市价订单。"` 并且机器人将拒绝启动。

!!! Warning "使用市价单"
    使用市价单时，请仔细阅读 [市价单定价](#market-order-pricing) 部分。

!!! Note "交易所止损"
    `order_types.stoploss_on_exchange_interval` 不是强制性的。如果你不确定自己在做什么，请不要更改其值。有关止损如何工作的更多信息，请参阅 [止损文档](stoploss.md)。

    If `order_types.stoploss_on_exchange` is enabled and the stoploss is cancelled manually on the exchange, then the bot will create a new stoploss order.

!!! Warning "警告：order_types.stoploss_on_exchange 失败"
    如果由于某种原因交易所止损创建失败，则会启动"紧急退出"。默认情况下，这将使用市价单退出交易。可以通过在 `order_types` 字典中设置 `emergency_exit` 值来更改紧急退出的订单类型——但不建议这样做。

### 理解 order_time_in_force

`order_time_in_force` 配置参数定义了订单在交易所执行的策略。  
常用的有效时间策略包括：

**GTC（Good Till Canceled，直至取消有效）：**

这通常是默认的有效时间策略。它意味着订单将保留在交易所，直到用户取消。订单可以被全部或部分成交。
如果部分成交，剩余部分将保留在交易所直至被取消。

**FOK（Fill Or Kill，全部成交或立即取消）：**

这意味着如果订单未能立即且全部成交，则会被交易所自动取消。

**IOC（Immediate Or Canceled，立即成交或取消）：**

它与 FOK（见上）类似，但允许部分成交。未成交的部分会被交易所自动取消。

**PO（Post only，仅做市）：**

仅做市订单。该订单要么作为做市方订单被挂单，要么被取消。
这意味着订单必须在未成交状态下在订单簿上停留至少一段时间。

请查阅[交易所文档](exchanges.md)以了解您的交易所支持的有效时间值。

#### time_in_force 配置

`order_time_in_force` 参数包含一个字典，其中定义了入场和离场订单的有效时间策略值。
这可以在配置文件中或策略中进行设置。
配置文件中设置的值会覆盖策略中的值，遵循常规的[配置选项优先级规则](#configuration-option-prevalence)。

可能的取值为：`GTC`（默认）、`FOK` 或 `IOC`。

``` python
"order_time_in_force": {
    "entry": "GTC",
    "exit": "GTC"
},
```

!!! Warning
    除非您清楚自己在做什么，并且已研究过针对特定交易所使用不同取值的影响，否则请不要更改默认值。

### 法币换算

Freqtrade 使用 Coingecko API 将代币价值转换为对应的法币价值，用于 Telegram 报告。
可在配置文件中通过 `fiat_display_currency` 参数设置法币种类。

若从配置中完全移除 `fiat_display_currency` 参数，则将跳过 coingecko 初始化，且不会显示任何法币换算信息。这对机器人的正常运行没有影响。

#### fiat_display_currency 可使用哪些取值？

`fiat_display_currency` 配置参数用于设置机器人 Telegram 报告中代币转法币换算的基准货币。

有效取值包括：

```json
"AUD", "BRL", "CAD", "CHF", "CLP", "CNY", "CZK", "DKK", "EUR", "GBP", "HKD", "HUF", "IDR", "ILS", "INR", "JPY", "KRW", "MXN", "MYR", "NOK", "NZD", "PHP", "PKR", "PLN", "RUB", "SEK", "SGD", "THB", "TRY", "TWD", "ZAR", "USD"
```

除法定货币外，还支持多种加密货币。

有效取值包括：

```json
"BTC", "ETH", "XRP", "LTC", "BCH", "BNB"
```

#### Coingecko 速率限制问题

在某些 IP 段，coingecko 会实施严格的速率限制。
遇到这种情况时，您可能需要将 coingecko API 密钥添加到配置中。

``` json
{
    "fiat_display_currency": "USD",
    "coingecko": {
        "api_key": "your-api",
        "is_demo": true
    }
}
```

Freqtrade 同时支持演示版和专业版 coingecko API 密钥。

Coingecko API 密钥并非机器人正常运行所必需的。
它仅用于 Telegram 报告中的加密货币与法币的转换功能，而该功能通常在没有 API 密钥的情况下也能正常工作。

## 使用交易所 WebSocket

Freqtrade 可以通过 ccxt.pro 使用 WebSocket。

Freqtrade 致力于确保数据始终可用。
如果 WebSocket 连接失败（或被禁用），机器人将回退到 REST API 调用。

如果您遇到怀疑是由 WebSocket 引起的问题，可以通过设置 `exchange.enable_ws` 来禁用它，该设置默认为 true。

```jsonc
"exchange": {
    // ...
    "enable_ws": false,
    // ...
}
```

如果需要使用代理，请参考[代理使用章节](#using-a-proxy-with-freqtrade)获取更多信息。

!!! Info "逐步推出"
    我们正在逐步实施此功能，以确保您机器人的稳定性。
    目前，使用范围仅限于 ohlcv 数据流。
    同时仅支持少数交易所，并正在持续添加新的交易所支持。

## 使用模拟模式

我们建议在模拟模式下启动机器人，以观察机器人的行为表现和策略的性能。在模拟模式下，机器人不会动用您的真实资金。它仅运行实时模拟，不会在交易所实际创建交易。

1.  编辑您的 `config.json` 配置文件。
2.  将 `dry-run` 切换为 `true` 并为持久化数据库指定 `db_url`。

```json
"dry_run": true,
"db_url": "sqlite:///tradesv3.dryrun.sqlite",
```

3. 移除您的交易所 API 密钥和密钥（将其更改为空值或模拟凭证）：

```json
"exchange": {
    "name": "binance",
    "key": "key",
    "secret": "secret",
    ...
}
```

当您对机器人在模拟运行模式下的表现感到满意后，可以将其切换至生产模式。

!!! Note
    模拟运行模式下可使用模拟钱包，初始资金将设定为 `dry_run_wallet`（默认为 1000）。

### 模拟运行注意事项

* API 密钥可选择是否提供。模拟运行模式下仅执行交易所的只读操作（即不会改变账户状态的操作）。
* 钱包余额（`/balance`）基于 `dry_run_wallet` 进行模拟。
* 订单为模拟执行，不会实际提交至交易所。
* 市价单将根据下单时刻的订单簿量进行成交，最大滑点不超过 5%。
* 限价单在价格达到设定水平时成交，或根据 `unfilledtimeout` 设置超时取消。
* 若限价单与当前价格偏差超过 1%，将自动转为市价单，并立即根据常规市价单规则成交（参见上文市价单说明）。
* 结合 `stoploss_on_exchange` 功能时，止损价将被视为已成交。
* 未成交订单（非数据库中存储的交易记录）在机器人重启后保持开放状态，默认离线期间未被成交。

## 切换到生产模式

在生产模式下，该机器人将动用您的真实资金。请务必谨慎操作，因为错误的策略可能导致全部资金损失。
请确保您完全了解在生产模式下运行机器人所带来的后果。

切换到生产模式时，请务必使用不同的/全新的数据库，以避免模拟交易干扰您的交易所资金并最终影响统计数据准确性。

### 设置您的交易所账户

您需要在交易所网站上创建API密钥（通常包含`key`和`secret`，部分交易所还需额外提供`password`），并将其填入配置文件的相应字段，或在执行`freqtrade new-config`命令时根据提示输入。
API密钥通常仅适用于实盘交易（真实资金交易，机器人运行于"生产模式"，在交易所执行真实订单），在模拟交易（交易模拟）模式下运行的机器人则不需要。当您在模拟模式下设置机器人时，这些字段可以留空。

### 将机器人切换至生产模式

**编辑您的`config.json`文件。**

**将dry-run设置为false，如果已设置数据库URL请记得相应调整：**

```json
"dry_run": false,
```

**填入您的交易所API密钥（请勿使用示例密钥）：**

```json
{
    "exchange": {
        "name": "binance",
        "key": "af8ddd35195e9dc500b9a6f799f6f5c93d89193b",
        "secret": "08a9dc6db3d7b53e1acebd9275677f4b0a04f1a5",
        //"password": "", // Optional, not needed by all exchanges)
        // ...
    }
    //...
}
```

同时建议您仔细阅读文档的[交易所](exchanges.md)章节，以了解特定交易所可能存在的配置细节。

!!! Hint "保护你的密钥安全"
    为了保护你的密钥安全，我们建议为你的API密钥使用第二套配置文件。
    只需将上述代码片段放入新的配置文件（例如 `config-private.json`）中，并将你的设置保存在此文件中。
    然后你可以使用 `freqtrade trade --config user_data/config.json --config user_data/config-private.json <...>` 启动机器人，以加载你的密钥。

    **NEVER** share your private configuration file or your exchange keys with anyone!

## 在 Freqtrade 中使用代理

要在 freqtrade 中使用代理，请使用设置为适当值的变量 `"HTTP_PROXY"` 和 `"HTTPS_PROXY"` 导出你的代理设置。
这将使代理设置应用于所有内容（电报、CoinGecko 等）**除了**交易所请求。

``` bash
export HTTP_PROXY="http://addr:port"
export HTTPS_PROXY="http://addr:port"
freqtrade
```

### 代理交易所请求

要对交易所连接使用代理 - 你必须将代理定义为 ccxt 配置的一部分。

``` json
{ 
  "exchange": {
    "ccxt_config": {
      "httpsProxy": "http://addr:port",
      "wsProxy": "http://addr:port",
    }
  }
}
```

有关可用代理类型的更多信息，请查阅 [ccxt 代理文档](https://docs.ccxt.com/#/README?id=proxy)。

## 下一步

现在你已经配置好了 config.json，下一步是[启动你的机器人](bot-usage.md)。