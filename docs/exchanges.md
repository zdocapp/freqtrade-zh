# 交易所特定说明

本页面汇总了特定于交易所的常见注意事项和信息，这些内容很可能不适用于其他交易所。

## 交易所配置

Freqtrade 基于 [CCXT 库](https://github.com/ccxt/ccxt) 构建，该库支持超过 100 个加密货币交易所市场和交易 API。完整的最新列表可在 [CCXT 代码库主页](https://github.com/ccxt/ccxt/tree/master/python) 找到。但开发团队仅对少数交易所进行了测试，当前列表可在本文档的"首页"部分查看。

欢迎测试其他交易所并提交反馈或 PR 来改进机器人，或确认运行完美的交易所。

部分交易所需要特殊配置，具体如下所示。

### 交易所配置示例

"binance"交易所的配置示例如下：

```json
"exchange": {
    "name": "binance",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "ccxt_config": {},
    "ccxt_async_config": {},
    // ... 
```

### 设置速率限制

通常 CCXT 设置的速率限制是可靠且有效的。若遇到与速率限制相关的问题（通常是日志中的 DDOS 异常），可轻松将 rateLimit 设置更改为其他值。

```json
"exchange": {
    "name": "kraken",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "ccxt_config": {"enableRateLimit": true},
    "ccxt_async_config": {
        "enableRateLimit": true,
        "rateLimit": 3100
    },
```

以下配置启用 kraken 交易所，并通过速率限制避免被交易所封禁。`"rateLimit": 3100` 定义了每次调用间隔 3.1 秒的等待事件。通过将 `"enableRateLimit"` 设置为 false 也可完全禁用此功能。

!!! Note
    限流的最佳设置取决于交易所和白名单的大小，因此理想的参数会因许多其他设置而异。
    我们尽可能为每个交易所提供合理的默认值，如果您遇到封禁，请确保已启用 `"enableRateLimit"` 并逐步增加 `"rateLimit"` 参数。

## Binance

!!! Warning "服务器位置与地理IP限制"
    请注意，Binance 会根据服务器所在国家限制 API 访问。当前被封锁的国家（非详尽列表）包括加拿大、马来西亚、荷兰和美国。请访问 [Binance 条款 > b. 资格](https://www.binance.com/en/terms) 查看最新列表。

Binance 支持 [time_in_force](configuration.md#understand-order_time_in_force)。

!!! Tip "交易所止损"
    Binance 支持 `stoploss_on_exchange` 并使用 `stop-loss-limit` 订单。这提供了巨大优势，因此我们建议通过启用交易所止损来利用此功能。
    在期货交易中，Binance 同时支持 `stop-limit` 和 `stop-market` 订单。您可以在 `order_types.stoploss` 配置设置中使用 `"limit"` 或 `"market"` 来决定使用哪种类型。

### Binance 黑名单建议

对于币安（Binance），建议将 `"BNB/<STAKE>"` 添加到您的黑名单中以避免问题，除非您愿意在账户中维持足够的额外 `BNB`，或者您愿意禁用使用 `BNB` 支付费用。
币安账户可以使用 `BNB` 支付费用，如果交易恰好涉及 `BNB`，后续交易可能会消耗该持仓，导致初始的 BNB 交易因预期数量不足而无法卖出。

如果没有足够的 `BNB` 来支付交易费用，那么费用将不会由 `BNB` 承担，也不会发生费用减免。Freqtrade 永远不会购买 BNB 来支付费用。为此，需要手动购买并监控 BNB。

### 币安站点

币安已拆分为两个部分，用户必须为其交易所使用正确的 ccxt 交易所 ID，否则 API 密钥将无法被识别。

* [binance.com](https://www.binance.com/) - 国际用户。使用交易所 ID：`binance`。
* [binance.us](https://www.binance.us/) - 美国用户。使用交易所 ID：`binanceus`。

### 币安 RSA 密钥

Freqtrade 支持币安 RSA API 密钥。

我们建议将它们作为环境变量使用。

``` bash
export FREQTRADE__EXCHANGE__SECRET="$(cat ./rsa_binance.private)"
```

然而，它们也可以通过配置文件进行配置。由于 json 不支持多行字符串，您必须将所有换行符替换为 `\n` 以生成有效的 json 文件。

``` json
// ...
 "key": "<someapikey>",
 "secret": "-----BEGIN PRIVATE KEY-----\nMIIEvQIBABACAFQA<...>s8KX8=\n-----END PRIVATE KEY-----"
// ...
```

### 币安期货

币安有特定的（不幸的是复杂的）[期货交易量化规则](https://www.binance.com/en/support/faq/4f462ebe6ff445d4a170be7d9e897272)需要遵守，其中包括禁止过多订单的单笔金额过低。
违反这些规则将导致交易限制。

在币安期货市场交易时，必须使用订单簿，因为期货没有价格行情数据。

``` jsonc
  "entry_pricing": {
      "use_order_book": true,
      "order_book_top": 1,
      "check_depth_of_market": {
          "enabled": false,
          "bids_to_ask_delta": 1
      }
  },
  "exit_pricing": {
      "use_order_book": true,
      "order_book_top": 1
  },
```

#### 币安逐仓期货设置

用户还必须将期货设置中的"持仓模式"设置为"单向持仓模式"，并将"资产模式"设置为"单一资产模式"。
这些设置将在启动时检查，如果设置错误，freqtrade 将显示错误。

![币安期货设置](assets/binance_futures_settings.png)

Freqtrade 不会尝试更改这些设置。

#### 币安 BNFCR 期货

BNFCR 模式是币安上一种特殊的期货模式，用于规避欧洲的监管问题。
要使用 BNFCR 期货，您必须使用以下设置组合：

``` jsonc
{
    // ...
    "trading_mode": "futures",
    "margin_mode": "cross",
    "proxy_coin": "BNFCR",
    "stake_currency": "USDT" // or "USDC"
    // ...
}
```

`stake_currency` 设置定义了机器人将操作的市场。这个选择实际上是任意的。

在交易所上，您必须使用"多资产模式" - 并且"持仓模式"设置为"单向持仓模式"。
Freqtrade 将在启动时检查这些设置，但不会尝试更改它们。

## Bingx

BingX 支持 [time_in_force](configuration.md#understand-order_time_in_force) 设置，可选 "GTC"（取消前有效）、"IOC"（立即或取消）和 "PO"（仅限挂单）设置。

!!! Tip "交易所止损"
    Bingx 支持 `stoploss_on_exchange`，并可同时使用限价止损和市价止损订单。这提供了巨大优势，因此我们建议通过启用交易所止损来从中受益。

## Kraken

Kraken 支持 [time_in_force](configuration.md#understand-order_time_in_force) 设置，可选 "GTC"（取消前有效）、"IOC"（立即或取消）和 "PO"（仅限挂单）设置。

!!! Tip "交易所止损"
    Kraken 支持 `stoploss_on_exchange`，并可同时使用市价止损和限价止损订单。这提供了巨大优势，因此我们建议通过启用交易所止损来从中受益。
    您可以在 `order_types.stoploss` 配置设置中使用 `"limit"` 或 `"market"` 来决定使用哪种类型。

### Kraken 历史数据

Kraken API 仅提供 720 根历史 K 线数据，这对于 Freqtrade 的模拟交易和实盘交易模式足够，但对于回测则存在问题。
要为 Kraken 交易所下载数据，必须使用 `--dl-trades` 参数，否则机器人将反复下载相同的 720 根 K 线，您将没有足够的回测数据。

为加速下载，您可下载 Kraken 提供的[交易记录压缩包](https://support.kraken.com/hc/en-us/articles/360047543791-Downloadable-historical-market-data-time-and-sales-)。
这些文件通常每季度更新一次。Freqtrade 要求将这些文件放置在 `user_data/data/kraken/trades_csv` 目录下。

若使用增量文件，采用以下结构较为合理：将"完整"历史数据存放于一个目录，增量文件分置于不同目录。
此模式的前提是数据下载解压后保持原始文件名不变。
重复内容（基于时间戳判断）将被忽略——但前提是数据不存在断档。

这意味着，如果您的"完整"历史数据截止到2022年第四季度，那么2023年第一季度和第二季度的增量更新需同时就位。
若缺少增量文件将导致数据不完整，进而使使用该数据得出的结果无效。

```
└── trades_csv
    ├── Kraken_full_history
    │   ├── BCHEUR.csv
    │   └── XBTEUR.csv
    ├── Kraken_Trading_History_Q1_2023
    │   ├── BCHEUR.csv
    │   └── XBTEUR.csv
    └── Kraken_Trading_History_Q2_2023
        ├── BCHEUR.csv
        └── XBTEUR.csv
```

您可将这些文件转换为 freqtrade 格式：

``` bash
freqtrade convert-trade-data --exchange kraken --format-from kraken_csv --format-to feather
# Convert trade data to different ohlcv timeframes
freqtrade trades-to-ohlcv -p BTC/EUR BCH/EUR --exchange kraken -t 1m 5m 15m 1h
```

转换后的数据还支持增量下载功能，系统将从已载入的最新交易记录之后开始下载。

``` bash
freqtrade download-data --exchange kraken --dl-trades -p BTC/EUR BCH/EUR 
```

!!! Warning "从 Kraken 下载数据"
    下载 Kraken 数据将需要比任何其他交易所显著更多的内存（RAM），因为交易数据需要在您的机器上转换为 K 线数据。
    这也会花费很长时间，因为 freqtrade 需要下载该交易对/时间范围组合在交易所发生的每一笔交易，因此请耐心等待。

!!! Warning "rateLimit 调优"
    请注意，rateLimit 配置项表示请求之间的延迟时间（以毫秒为单位），而不是每秒请求速率。
    因此，为了避免 Kraken API 的"超出速率限制"异常，此配置值应该增加，而不是减少。

## Kucoin

Kucoin 要求每个 API 密钥都附带一个密码短语，因此您需要将此密钥添加到配置中，使您的交易所配置部分如下所示：

```json
"exchange": {
    "name": "kucoin",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

Kucoin 支持 [time_in_force](configuration.md#understand-order_time_in_force)，可设置"GTC"（取消前有效）、"FOK"（全部成交或取消）和"IOC"（立即成交或取消）选项。

!!! Tip "交易所止损"
    Kucoin 支持 `stoploss_on_exchange`，并且可以使用止损市价单和止损限价单。这提供了巨大优势，因此我们建议充分利用此功能。
    您可以在 `order_types.stoploss` 配置设置中使用 `"limit"` 或 `"market"` 来决定使用哪种类型的止损单。

### Kucoin 黑名单

对于 Kucoin，建议将 `"KCS/<STAKE>"` 添加到您的黑名单中以避免问题，除非您愿意在账户中维持足够的额外 `KCS`，或者除非您愿意禁用使用 `KCS` 支付费用。
Kucoin 账户可能使用 `KCS` 支付费用，如果交易恰好涉及 `KCS`，后续交易可能会消耗该头寸，导致初始的 `KCS` 交易无法卖出，因为预期的数量已不存在。

## HTX

!!! Tip "交易所止损"
    HTX 支持 `stoploss_on_exchange` 并使用 `stop-limit` 订单。这提供了巨大优势，因此我们建议通过启用交易所止损来利用这一功能。

## OKX

OKX 要求每个 API 密钥都设置密码，因此您需要将此密钥添加到配置中，使您的交易所配置部分如下所示：

```json
"exchange": {
    "name": "okx",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

如果您在主机 my.okx.com（OKX EAA）上注册了 OKX - 您需要使用 `"myokx"` 作为交易所名称。
使用错误的交易所将导致错误 "OKX Error 50119: API key doesn't exist" - 因为这两个是独立的实体。

!!! Warning
    OKX 每次 API 调用仅提供 100 根 K 线。因此，在回测模式下，策略可用的数据量将相当有限。

!!! Warning "Futures"
    OKX 期货交易存在"持仓模式"概念——可以是"买入/卖出"模式或多空(对冲模式)。
    Freqtrade 同时支持这两种模式（我们推荐使用买入/卖出模式）——但不支持在交易中途切换模式，否则将导致异常和下单失败。
    OKX 仅提供过去约3个月的标记价格K线数据。因此对该日期之前的期货数据进行回测会产生轻微偏差，因为缺少这些数据将无法准确计算资金费率。

## Gate.io

!!! Tip "Stoploss on Exchange"
    Gate.io 支持 `stoploss_on_exchange` 并使用`止损限价`订单。该功能优势显著，因此我们建议通过启用交易所止损来利用这一优势。

Gate.io 支持 [time_in_force](configuration.md#understand-order_time_in_force) 参数，可设置为"GTC"(取消前有效)和"IOC"(立即成交或取消)模式。

Gate.io 允许使用 `POINT` 支付手续费。由于这不是可交易货币（无常规市场），自动手续费计算将失败（并默认手续费为0）。
可通过配置参数 `exchange.unknown_fee_rate` 指定 Point 与交易货币之间的汇率。显然，更改交易货币也需要同步调整此数值。

Gate.io API密钥除所需交易的市场类型外，还需具备以下权限：

* "现货交易" _或_ "永续合约" (读写权限) (可同时选择两者，或选择与您要交易的市场匹配的选项)
* "钱包" (只读权限)
* "账户" (只读权限)

若缺少这些权限，机器人将无法正常启动并显示"权限缺失"等错误信息。

## Bybit

!!! Tip "交易所止损功能"
    Bybit (仅限合约) 支持 `stoploss_on_exchange` 并使用 `止损限价` 订单。这具有显著优势，因此我们建议通过启用交易所止损功能来利用此优势。
    在合约交易中，Bybit 同时支持 `止损限价` 和 `止损市价` 订单。您可以在 `order_types.stoploss` 配置设置中使用 `"limit"` 或 `"market"` 来决定使用哪种类型。

Bybit 支持 [订单时效](configuration.md#understand-order_time_in_force) 设置，包括 "GTC"（取消前有效）、"FOK"（全部成交或取消）、"IOC"（立即成交或取消）和 "PO"（仅限挂单）设置。

Bybit 的合约交易目前支持隔离保证金模式。

启动时，freqtrade 将为整个（子）账户设置持仓模式为"单向持仓模式"。这避免了重复进行此调用（减缓机器人运行速度），但意味着手动更改此设置可能导致异常和错误。

由于 Bybit 不提供资金费率历史数据，实盘交易同样使用模拟计算方式。

用于实盘期货交易的API密钥必须具备以下权限：

* 读写权限
* 合约 - 订单
* 合约 - 持仓

我们强烈建议将所有API密钥限制在您将要使用的IP地址范围内。

!!! Warning "统一账户"
    Freqtrade假定账户专用于机器人交易。
    因此我们建议每个机器人使用一个子账户。在使用统一账户时这一点尤为重要。  
    其他配置（一个账户上运行多个机器人、在机器人账户上进行非机器人手动交易）不受支持，并可能导致意外行为。

## Bitmart

Bitmart要求API密钥备注（您为API密钥设置的名称）与交易所密钥和密钥一起使用。
因此还需要传递UID。

```json
"exchange": {
    "name": "bitmart",
    "uid": "your_bitmart_api_key_memo",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

!!! Warning "必要验证"
    Bitmart要求通过Lvl2验证才能通过API在现货市场成功交易——即使仅通过Lvl1验证在UI界面交易也能正常工作。

## Bitget

Bitget要求每个API密钥设置密码，因此您需要将此密钥添加到配置中，使您的交易所配置部分如下所示：

```json
"exchange": {
    "name": "bitget",
    "key": "your_exchange_key",
    "secret": "your_exchange_secret",
    "password": "your_exchange_api_key_password",
    // ...
}
```

Bitget支持[time_in_force](configuration.md#understand-order_time_in_force)设置，包括"GTC"（取消前有效）、"FOK"（全部成交或取消）、"IOC"（立即成交或取消）和"PO"（仅限挂单）设置。

!!! Tip "Stoploss on Exchange"
    Bitget 支持 `stoploss_on_exchange` 功能，并能同时使用止损市价单和止损限价单。这提供了巨大优势，因此我们建议充分利用该功能。
    您可以在 `order_types.stoploss` 配置设置中使用 `"limit"` 或 `"market"` 来决定使用哪种类型的止损单。

## Hyperliquid

!!! Tip "Stoploss on Exchange"
    Hyperliquid 支持 `stoploss_on_exchange` 功能并使用 `stop-loss-limit` 订单。这提供了巨大优势，因此我们建议充分利用该功能。

Hyperliquid 是一个去中心化交易所（DEX）。去中心化交易所与常规交易所的运作方式略有不同。私有 API 调用无需使用 API 密钥进行身份验证，而是需要使用您钱包的私钥进行签名（我们建议为此使用在 Hyperliquid 或您选择的钱包中生成的 API 钱包）。
需要按如下方式配置：

```json
"exchange": {
    "name": "hyperliquid",
    "walletAddress": "your_eth_wallet_address",  // This should NOT be your API Wallet Address!
    "privateKey": "your_api_private_key",
    // ...
}
```

* 十六进制格式的钱包地址：`0x<40 位十六进制字符>` - 可从您的钱包轻松复制 - 且应是您的主钱包地址，而非 API 钱包地址。
* 十六进制格式的私钥：`0x<64 位十六进制字符>` - 使用 API 钱包创建时显示的密钥。

Hyperliquid 在 Arbitrum One 链上处理存款和提现，这是一个构建在以太坊之上的 Layer 2 扩容解决方案。Hyperliquid 使用 USDC 作为报价/抵押品。在 Hyperliquid 上存入 USDC 的过程需要几个步骤，有关所需步骤的详细信息，请参阅 [如何开始交易](https://hyperliquid.gitbook.io/hyperliquid-docs/onboarding/how-to-start-trading)。

!!! Note "Hyperliquid 通用使用说明"
    Hyperliquid 不支持市价单，但 ccxt 将通过下止损限价单来模拟市价单，最大滑点率为 5%。
    遗憾的是，hyperliquid 仅提供 5000 根历史 K 线，因此回测要么需要历史性地构建 K 线（通过等待并随时间推移逐步下载数据），要么将限于最近 5000 根 K 线。

!!! Info "一些通用最佳实践（非详尽列表）"
    * 警惕供应链攻击，例如 pip 包投毒等。每当使用私钥时，请确保环境安全。
    * 不要使用实际钱包私钥进行交易。使用 Hyperliquid [API 生成器](https://app.hyperliquid.xyz/API)创建独立的 API 钱包。
    * 不要将实际钱包私钥存储在用于 freqtrade 的服务器上。请改用 API 钱包私钥。该密钥不允许提现，仅用于交易。
    * 始终保管好你的助记词和私钥。
    * 不要使用初始化硬件钱包时需要备份的相同助记词，使用相同助记词基本上会消除硬件钱包的安全性。
    * 创建不同的软件钱包，仅将想要用于交易的资金转入该钱包，并使用该钱包在 Hyperliquid 上进行交易。
    * 如果有不想用于交易的资金（例如获利后），请将其转回硬件钱包。

### Hyperliquid 金库 / 子账户

Hyperliquid 允许你创建金库或子账户。  
要在 Freqtrade 中使用这些功能，你需要使用以下配置模式：

``` json
"exchange": {
    "name": "hyperliquid",
    "walletAddress": "your_vault_address",  // Vault or subaccount address
    "privateKey": "your_api_private_key",
    "ccxt_config": {
        "options": {
            "vaultAddress": "your_vault_address" // Optional, only if you want to use a vault or subaccount
        }
    },
    // ...
}
```

现在将从你的金库/子账户使用余额和进行交易，而不再从主账户使用。

### Hyperliquid 历史数据

Hyperliquid API 不提供单次获取当前数据之外的历史数据，因此无法下载数据，因为下载的数据无法构成完整的历史数据。

## Bitvavo

如果您的账户需要使用 operatorId，可以在配置文件中按如下方式设置：

``` json
"exchange": {
        "name": "bitvavo",
        "key": "",
        "secret": "",
        "ccxt_config": {
            "options": {
                "operatorId": "123567"
            }
        },
   }
```

Bitvavo 要求 `operatorId` 为整数类型。

## 所有交易所

若持续遇到 Nonce 相关错误（例如 `InvalidNonce`），最佳解决方案是重新生成 API 密钥。重置 Nonce 较为困难，通常重新生成 API 密钥更为便捷。

## 其他交易所的随机备注

* The Ocean（交易所 ID：`theocean`）使用 Web3 功能，需要安装 `web3` Python 包：

```shell
pip3 install web3
```

### 获取最新价格 / 不完整的 K 线

大多数交易所通过其 OHLCV/K 线 API 接口返回当前未完成的 K 线。
默认情况下，Freqtrade 假定从交易所获取的是不完整的 K 线，并会移除最后一根 K 线（假定其为未完成状态）。

您可以通过[贡献者文档中的辅助脚本](developer.md#incomplete-candles)来验证交易所是否返回不完整的 K 线。

由于存在重新绘图的风险，Freqtrade 不允许使用这类未完成的 K 线。

然而，如果这是基于策略对最新价格的需求——那么这一要求可以通过策略内部的[数据提供器](strategy-customization.md#possible-options-for-dataprovider)来获取。

### 高级 Freqtrade 交易所配置

高级选项可通过 `_ft_has_params` 设置进行配置，该设置将覆盖默认值和特定交易所的行为。

可用选项在交易所类中列为 `_ft_has_default`。

例如，要使用 Kraken 测试 `FOK` 订单类型，并将蜡烛图限制修改为 200（这样每次 API 调用仅获取 200 根蜡烛）：

```json
"exchange": {
    "name": "kraken",
    "_ft_has_params": {
        "order_time_in_force": ["GTC", "FOK"],
        "ohlcv_candle_limit": 200
        }
    //...
}
```

!!! Warning
    请务必在修改这些设置前充分理解其影响。