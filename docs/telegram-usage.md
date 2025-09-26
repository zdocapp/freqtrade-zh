# Telegram 使用指南

## 设置您的 Telegram 机器人

以下将说明如何创建 Telegram 机器人，以及如何获取您的 Telegram 用户 ID。

### 1. 创建 Telegram 机器人

与 [Telegram BotFather](https://telegram.me/BotFather) 开始对话

发送消息 `/newbot`。

*BotFather 回复：*

> 好的，一个新机器人。我们该如何称呼它？请为您的机器人选择一个名称。

选择您机器人的公开名称（例如 `Freqtrade 机器人`）

*BotFather 回复：*

> 很好。现在为您的机器人选择一个用户名。必须以 `bot` 结尾。例如：TetrisBot 或 tetris_bot。

选择您机器人的名称 ID 并发送给 BotFather（例如 "`My_own_freqtrade_bot`"）

*BotFather 回复：*

> 完成！恭喜您创建了新机器人。您可以在 `t.me/yourbots_name_bot` 找到它。现在可以为您的机器人添加描述、关于章节和头像，请参阅 /help 查看命令列表。顺便一提，当您完成酷炫机器人的创建后，如果想要更好的用户名，可以联系我们的机器人支持。只需确保机器人在此之前完全可操作。

> 使用此令牌访问 HTTP API：`22222222:APITOKEN`

> 有关机器人 API 的描述，请参阅此页面：https://core.telegram.org/bots/api BotFather 将返回令牌（API 密钥）

复制 API 令牌（上例中的 `22222222:APITOKEN`）并将其用于配置参数 `token`。

不要忘记通过点击 `/START` 按钮来启动与机器人的对话

### 2. Telegram 用户 ID

#### 获取您的用户 ID

与 [userinfobot](https://telegram.me/userinfobot) 对话

获取您的 "Id"，您将使用它作为配置参数 `chat_id`。

#### 使用群组 ID

要获取群组 ID，您可以将机器人添加到群组，启动 freqtrade，并执行 `/tg_info` 命令。
这将返回群组 ID 给您，无需使用其他随机机器人。
虽然 "chat_id" 仍然是必需的，但此命令不需要将其设置为特定的群组 ID。

如有必要，响应还将包含 "topic_id"——两者均以可直接复制/粘贴到配置中的格式提供。

``` json
 {
    "enabled": true,
    "token": "********",
    "chat_id": "-1001332619709",
    "topic_id": "122"
}
```

对于 Freqtrade 配置，您可以使用完整值（包括 `-`）作为字符串：

```json
   "chat_id": "-1001332619709"
```

!!! Warning "使用 Telegram 群组"
    使用 Telegram 群组时，您将授予群组中每个成员访问您的 freqtrade 机器人以及通过 Telegram 执行所有命令的权限。请确保您信任 Telegram 群组中的每个人，以避免不愉快的意外。

##### 群组话题 ID

要在群组中使用特定话题，可以通过配置中的 `topic_id` 参数实现。这将允许机器人在群组的特定话题中运作。  
若不设置此参数，当群聊启用话题功能时，机器人将始终在群组的通用频道中响应。

```json
   "chat_id": "-1001332619709",
   "topic_id": "3"
```

与获取群组ID类似 - 您可以在话题/线程中使用 `/tg_info` 命令获取正确的话题ID。

#### 授权用户

对于群组场景，限制可向机器人发送命令的用户范围会非常实用。

若 `"authorized_users": []` 配置项存在且为空列表，则所有用户都将无权操控机器人。
以下示例中，仅用户ID为"1234567"的用户被允许操控机器人——其他所有用户仅能接收消息。

```json
   "chat_id": "-1001332619709",
   "topic_id": "3",
   "authorized_users": ["1234567"]
```

## 控制 Telegram 通知频率

Freqtrade 提供了多种方式来调节机器人通知的详细程度。
每个设置项支持以下可选值：

* `on` - 发送消息并通知用户
* `silent` - 发送消息但通知时无提示音/振动
* `off` - 完全跳过该类消息的发送

展示不同设置的配置示例：

``` json
"telegram": {
    "enabled": true,
    "token": "your_telegram_token",
    "chat_id": "your_telegram_chat_id",
    "allow_custom_messages": true,
    "notification_settings": {
        "status": "silent",
        "warning": "on",
        "startup": "off",
        "entry": "silent",
        "entry_fill": "on",
        "entry_cancel": "silent",
        "exit": {
            "roi": "silent",
            "emergency_exit": "on",
            "force_exit": "on",
            "exit_signal": "silent",
            "trailing_stop_loss": "on",
            "stop_loss": "on",
            "stoploss_on_exchange": "on",
            "custom_exit": "silent",  // custom_exit without specifying an exit reason
            "partial_exit": "on",
            // "custom_exit_message": "silent",  // Disable individual custom exit reasons
            "*": "off"  // Disable all other exit reasons
        },
        // "exit": "off",  // Simplistic configuration to disable all exit messages
        "exit_cancel": "on",
        "exit_fill": "off",
        "protection_trigger": "off",
        "protection_trigger_global": "on",
        "strategy_msg": "off",
        "show_candle": "off"
    },
    "reload": true,
    "balance_dust_level": 0.01
},
```

* `entry` 通知在订单创建时发送，而 `entry_fill` 通知在交易所订单成交时发送。  
* `exit` 通知在订单创建时发送，而 `exit_fill` 通知在交易所订单成交时发送。  
    退出消息（`exit` 和 `exit_fill`）可以在单个退出原因级别进一步控制，以特定退出原因为键。所有退出原因的默认值为 `on` - 但可以通过特殊的 `*` 键进行配置 - 该键将作为所有未明确定义的退出原因的通配符。
* `*_fill` 通知默认关闭，必须显式启用。  
* `protection_trigger` 通知在保护机制触发时发送，而 `protection_trigger_global` 通知在全局保护触发时发送。  
* `strategy_msg` - 接收来自策略的通知，通过策略中的 `self.dp.send_msg()` 发送 [更多详情](strategy-customization.md#send-notification)。  
* `show_candle` - 在入场/出场消息中显示蜡烛图数值。唯一可能的值为 `"ohlc"` 或 `"off"`。  
* `balance_dust_level` 将定义 `/balance` 命令视为"粉尘"的阈值 - 余额低于此值的货币将被显示。  
* `allow_custom_messages` 完全禁用策略消息。  
* `reload` 允许您在选定消息上禁用重新加载按钮。

## 创建自定义键盘（命令快捷按钮）

Telegram 允许我们创建带有命令按钮的自定义键盘。
默认的自定义键盘如下所示。

```python
[
    ["/daily", "/profit", "/balance"], # row 1, 3 commands
    ["/status", "/status table", "/performance"], # row 2, 3 commands
    ["/count", "/start", "/stop", "/help"] # row 3, 4 commands
]
```

### 使用方法

您可以在 `config.json` 中创建自己的键盘：

``` json
"telegram": {
      "enabled": true,
      "token": "your_telegram_token",
      "chat_id": "your_telegram_chat_id",
      "keyboard": [
          ["/daily", "/stats", "/balance", "/profit"],
          ["/status table", "/performance"],
          ["/reload_config", "/count", "/logs"]
      ]
   },
```

!!! Note "支持的命令"
    仅允许以下命令。不支持命令参数！

    `/start`, `/pause`, `/stop`, `/status`, `/status table`, `/trades`, `/profit`, `/performance`, `/daily`, `/stats`, `/count`, `/locks`, `/balance`, `/stopentry`, `/reload_config`, `/show_config`, `/logs`, `/whitelist`, `/blacklist`, `/help`, `/version`, `/marketdir`

## Telegram 命令

默认情况下，Telegram 机器人会显示预定义的命令。某些命令
只能通过发送给机器人来使用。下表列出了
官方命令。您可以随时使用 `/help` 寻求帮助。

|  命令 | 描述 |
|----------|-------------|
| **系统命令** |
| `/start` | 启动交易机器人
| `/pause | /stopentry | /stopbuy` | 暂停交易机器人。根据规则妥善处理未平仓交易。不建立新仓位。
| `/stop` | 停止交易机器人
| `/reload_config` | 重新加载配置文件
| `/show_config` | 显示当前配置中与操作相关的部分设置
| `/logs [limit]` | 显示最新的日志消息。
| `/help` | 显示帮助信息
| `/version` | 显示版本信息
| **状态** |
| `/status` | 列出所有未平仓交易
| `/status <trade_id>` | 列出一个或多个特定交易。多个 <trade_id> 用空格分隔。
| `/status table` | 以表格格式列出所有未平仓交易。挂单的买单用星号 (*) 标记，挂单的卖单用双星号 (**) 标记。
| `/order <trade_id>` | 列出一个或多个特定交易的订单。多个 <trade_id> 用空格分隔。
| `/trades [limit]` | 以表格格式列出所有最近已平仓的交易。
| `/count` | 显示已使用和可用的交易数量
| `/locks` | 显示当前被锁定的交易对。
| `/unlock <pair or lock_id>` | 移除该交易对（或该锁定ID）的锁定。
| `/marketdir [long | short | even | none]` | 更新代表当前市场方向的用户管理变量。如果未提供方向，将显示当前设置的方向。
| `/list_custom_data <trade_id> [key]` | 列出交易ID和键组合的自定义数据。如果未提供键，将列出为该交易ID找到的所有键值对。
| **修改交易状态** |
| `/forceexit <trade_id> | /fx <tradeid>` | 立即平仓给定交易（忽略 `minimum_roi`）。
| `/forceexit all | /fx all` | 立即平仓所有未平仓交易（忽略 `minimum_roi`）。
| `/fx` | `/forceexit` 的别名
| `/forcelong <pair> [rate]` | 立即买入给定交易对。价格是可选的，仅适用于限价单。（`force_entry_enable` 必须设置为 True）
| `/forceshort <pair> [rate]` | 立即做空给定交易对。价格是可选的，仅适用于限价单。这仅适用于非现货市场。（`force_entry_enable` 必须设置为 True）
| `/delete <trade_id>` | 从数据库中删除特定交易。尝试取消未成交订单。需要在交易所手动处理此交易。
| `/reload_trade <trade_id>` | 从交易所重新加载交易。仅适用于实盘交易，可能有助于恢复在交易所手动卖出的交易。
| `/cancel_open_order <trade_id> | /coo <trade_id>` | 取消交易的未成交订单。
| **指标** |
| `/profit [<n>]` | 显示最近 n 天（默认为所有交易）已平仓交易的盈亏摘要以及一些绩效统计数据
| `/profit_[long|short] [<n>]` | 显示最近 n 天（默认为所有交易）某一方向已平仓交易的盈亏摘要以及一些绩效统计数据
| `/performance` | 按交易对分组显示每个已完成交易的绩效
| `/balance` | 显示每种货币由机器人管理的余额
| `/balance full` | 显示每种货币的账户余额
| `/daily <n>` | 显示最近 n 天（n 默认为 7）每天的盈亏情况
| `/weekly <n>` | 显示最近 n 周（n 默认为 8）每周的盈亏情况
| `/monthly <n>` | 显示最近 n 个月（n 默认为 6）每月的盈亏情况
| `/stats` | 按退出原因显示胜率/亏损率以及买入和卖出的平均持仓时间
| `/exits` | 按退出原因显示胜率/亏损率以及买入和卖出的平均持仓时间
| `/entries` | 按退出原因显示胜率/亏损率以及买入和卖出的平均持仓时间
| `/whitelist [sorted] [baseonly]` | 显示当前白名单。可选择按字母顺序显示和/或仅显示每个交易对的基础货币。
| `/blacklist [pair]` | 显示当前黑名单，或将交易对添加到黑名单。

## Telegram 命令实战

以下展示每个命令对应的 Telegram 消息示例。

### /start

> **状态:** `运行中`

### /pause | /stopentry | /stopbuy

> **状态:** `已暂停，此后不再开仓。执行 /start 命令可重新启用开仓功能。`

通过将状态更改为 `已暂停` 来阻止机器人开设新交易。
已开仓的交易将继续按照其常规规则（ROI/退出信号、止损等）进行管理。
请注意，仓位调整功能仍保持活跃，但仅限于退出方向——即当机器人处于 `已暂停` 状态时，只能减少已开仓交易的持仓规模。

此后，请给机器人时间平仓（可通过 `/status table` 查看进度）。
待所有仓位平仓后，执行 `/stop` 命令以完全停止机器人。

使用 `/start` 命令可将机器人恢复至 `运行中` 状态，允许其开设新仓位。

!!! Warning
    暂停/停止开仓信号仅在机器人运行时有效，且不会被持久化保存，因此重启机器人将导致该设置重置。

### /stop

> `正在停止交易程序...`
> **状态:** `已停止`

### /status

对于每个未平仓交易，机器人将向您发送以下消息。
入场标签可通过策略进行配置。

> **交易ID:** `123` `(开仓时间: 1天前)`  
> **当前交易对:** CVC/BTC  
> **方向:** 做多  
> **杠杆:** 1.0  
> **数量:** `26.64180098`  
> **开仓标签:** Awesome Long Signal  
> **开仓价格:** `0.00007489`  
> **当前价格:** `0.00007489`  
> **未实现盈亏:** `12.95%`  
> **止损价:** `0.00007389 (-0.02%)`

### /status 表格

以表格形式返回所有未平仓交易的状态。

```
ID L/S    Pair     Since   Profit
----    --------  -------  --------
  67 L   SC/BTC    1 d      13.33%
 123 S   CVC/BTC   1 h      12.95%
```

### /count

返回已使用和可用的交易数量。

```
current    max
---------  -----
     2     10
```

### /profit

也可使用 `/profit_long` 和 `/profit_short` 分别显示仅做多或仅做空交易的利润。

返回您的盈亏和绩效摘要。

> **投资回报率:** 已平仓交易  
>   ∙ `0.00485701 BTC (2.2%) (15.2 Σ%)`  
>   ∙ `62.968 USD`  
> **投资回报率:** 全部交易  
>   ∙ `0.00255280 BTC (1.5%) (6.43 Σ%)`  
>   ∙ `33.095 EUR`  
>  
> **总交易笔数:** `138`  
> **机器人启动时间:** `2022-07-11 18:40:44`  
> **首笔交易开仓时间:** `3天前`  
> **最近交易开仓时间:** `2分钟前`  
> **平均持仓时长:** `2:33:45`  
> **最佳表现:** `PAY/BTC: 50.23%`  
> **交易量:** `0.5 BTC`  
> **盈利因子:** `1.04`  
> **盈利/亏损交易数:** `102 / 36`  
> **胜率:** `73.91%`  
> **期望值 (比率):** `4.87 (1.66)`  
> **最大回撤:** `9.23% (0.01255 BTC)`

`1.2%` 的相对利润是每笔交易的平均利润。  
`15.2 Σ%` 的相对利润是基于起始资金计算的——因此在这种情况下，起始资金为 `0.00485701 * 1.152 = 0.00738 BTC`。  
**起始资金** 要么取自 `available_capital` 设置，要么通过当前钱包规模减去利润来计算。  
**利润因子** 的计算方式为总利润 / 总亏损，应作为策略的整体衡量指标。  
**期望值** 对应于每单位风险货币的平均回报，即胜率和风险回报比（盈利交易的平均收益与亏损交易的平均损失之比）。  
**期望比率** 是基于所有过往交易表现计算的后续交易的预期利润或损失。  
**最大回撤** 对应于回测指标 `绝对回撤（账户）`——计算方式为 `（绝对回撤）/（回撤高点 + 起始余额）`。  
**机器人启动日期** 将指机器人首次启动的日期。对于较旧的机器人，此日期将默认为第一笔交易的开仓日期。

### /forceexit <trade_id>

> **BINANCE:** 正在以限价 `0.01650000（利润：约 -4.07%，-0.00008168）` 退出 BTC/LTC

!!! Tip
    您可以通过调用不带参数的 `/forceexit` 命令获取所有未平仓交易的列表，该命令将显示一组按钮以便快速平仓。
    此命令有一个别名 `/fx` - 功能完全相同，但在"紧急"情况下输入更快捷。

### /forcelong <交易对> [价格] | /forceshort <交易对> [价格]

`/forcebuy <交易对> [价格]` 也支持做多操作，但应视为已弃用。

> **币安：** 以限价 `0.03400000` 做多 ETH/BTC (`1.000000 ETH`, `225.290 USD`)

省略交易对将弹出查询窗口，要求输入要交易的货币对（基于当前白名单）。
通过 `/forcelong` 创建的交易将带有 `force_entry` 买入标签。

![Telegram 强制买入截图](assets/telegram_forcebuy.png)

请注意，此功能需要将 `force_entry_enable` 设置为 true 才能生效。

[更多详情](configuration.md#understand-force_entry_enable)

### /performance

返回机器人已售出的每种加密货币的收益表现。

> 收益表现：  
> 1. `RCN/BTC 0.003 BTC (57.77%) (1)`  
> 2. `PAY/BTC 0.0012 BTC (56.91%) (1)`  
> 3. `VIB/BTC 0.0011 BTC (47.07%) (1)`  
> 4. `SALT/BTC 0.0010 BTC (30.24%) (1)`  
> 5. `STORJ/BTC 0.0009 BTC (27.24%) (1)`  
> ...

相对收益率是针对该币种的总投资额计算的，汇总了该币种所有已成交的买入记录。

### /balance

返回您在交易所持有的所有加密货币余额。

> **币种:** BTC  
> **可用余额:** 3.05890234  
> **总余额:** 3.05890234  
> **待处理:** 0.0  
>
> **币种:** CVC  
> **可用余额:** 86.64180098  
> **总余额:** 86.64180098  
> **待处理:** 0.0

### /daily <n>

默认情况下 `/daily` 将返回最近7天的数据。以下示例为 `/daily 3` 的结果：

> **最近3天的每日收益：**

```
Day (count)     USDT          USD         Profit %
--------------  ------------  ----------  ----------
2022-06-11 (1)  -0.746 USDT   -0.75 USD   -0.08%
2022-06-10 (0)  0 USDT        0.00 USD    0.00%
2022-06-09 (5)  20 USDT       20.10 USD   5.00%
```

### /weekly <n>

默认情况下 `/weekly` 将返回最近8周的数据（包含当前周）。每周从周一开始计算。以下示例为 `/weekly 3` 的结果：

> **最近3周的每周收益（从周一开始）：**

```
Monday (count)  Profit BTC      Profit USD   Profit %
-------------  --------------  ------------    ----------
2018-01-03 (5)  0.00224175 BTC  29,142 USD   4.98%
2017-12-27 (1)  0.00033131 BTC   4,307 USD   0.00%
2017-12-20 (4)  0.00269130 BTC  34.986 USD   5.12%
```

### /monthly <n>

默认情况下 `/monthly` 将返回最近6个月的数据（包含当前月）。以下示例为 `/monthly 3` 的结果：

> **最近3个月的每月收益：**

```
Month (count)  Profit BTC      Profit USD    Profit %
-------------  --------------  ------------    ----------
2018-01 (20)    0.00224175 BTC  29,142 USD  4.98%
2017-12 (5)    0.00033131 BTC   4,307 USD   0.00%
2017-11 (10)    0.00269130 BTC  34.986 USD  5.10%
```

### /whitelist

显示当前白名单

> 使用包含22个交易对的 `StaticPairList` 白名单  
> `IOTA/BTC, NEO/BTC, TRX/BTC, VET/BTC, ADA/BTC, ETC/BTC, NCASH/BTC, DASH/BTC, XRP/BTC, XVG/BTC, EOS/BTC, LTC/BTC, OMG/BTC, BTG/BTC, LSK/BTC, ZEC/BTC, HOT/BTC, IOTX/BTC, XMR/BTC, AST/BTC, XLM/BTC, NANO/BTC`

### /blacklist [交易对]

显示当前黑名单。
如果设置了交易对，则该交易对将被添加到黑名单中。
支持多个交易对，以空格分隔。  
使用 `/reload_config` 可重置黑名单。

> 使用包含 2 个交易对的静态配对列表黑名单  
> `DODGE/BTC`, `HOT/BTC`。

### /version

> **版本：** `0.14.3`

### /marketdir

如果提供了市场方向，该命令将更新代表当前市场方向的用户管理变量。
该变量在机器人启动时不会设置为任何有效的市场方向，必须由用户设置。以下示例针对 `/marketdir long`：

```
Successfully updated marketdirection from none to long.
```

如果未提供市场方向，该命令将输出当前设置的市场方向。以下示例针对 `/marketdir`：

```
Currently set marketdirection: even
```

您可以通过 `self.market_direction` 在策略中使用市场方向。

!!! Warning "机器人重启"
    请注意，市场方向不会被持久化，在机器人重启/重载后将被重置。

!!! Danger "回测"
    由于此值/变量旨在用于模拟/实盘交易中手动更改，使用 `market_direction` 的策略可能无法产生可靠、可复现的结果（对此变量的更改不会反映在回测中）。使用风险自负。