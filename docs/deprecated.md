# 已弃用功能

本页包含已被开发团队声明为弃用且不再支持的命令行参数、配置参数和机器人功能的说明。请避免在配置中使用这些功能。

## 已移除功能

### `--refresh-pairs-cached` 命令行选项

在回测、超参数优化和边缘功能中，`--refresh-pairs-cached` 允许刷新回测用的蜡烛图数据。由于这会导致很多混淆，并且会减慢回测速度（而这并非回测的一部分），该功能已被独立为单独的 freqtrade 子命令 `freqtrade download-data`。

该命令行选项在 2019.7-dev（开发分支）中被弃用，并在 2019.9 版本中移除。

### **--dynamic-whitelist** 命令行选项

该命令行选项在 2018 年被弃用，并在 freqtrade 2019.6-dev（开发分支）和 freqtrade 2019.7 版本中移除。请参考 [交易对列表](plugins.md#pairlists-and-pairlist-handlers) 替代方案。

### `--live` 命令行选项

在回测场景中，`--live` 允许下载最新的 tick 数据进行回测。但该功能仅下载最新的 500 根蜡烛图，因此无法有效获取高质量的回测数据。该功能在 2019-7-dev（开发分支）和 freqtrade 2019.8 版本中被移除。

### `ticker_interval`（现为 `timeframe`）

对 `ticker_interval` 术语的支持已于 2020.6 版本弃用，改用 `timeframe`——兼容性代码已于 2022.3 版本移除。

### 允许按顺序运行多个交易对列表

配置中原来的 `"pairlist"` 部分已被移除，替换为 `"pairlists"`——这是一个用于指定交易对列表序列的列表。

旧的配置参数部分（`"pairlist"`）已于 2019.11 版本弃用，并于 2020.4 版本移除。

### 弃用 volume-pairlist 中的 bidVolume 和 askVolume

由于只有 quoteVolume 可以在资产之间进行比较，其他选项（bidVolume、askVolume）已于 2020.4 版本弃用，并于 2020.9 版本移除。

### 使用订单簿步进设置退出价格

使用 `order_book_min` 和 `order_book_max` 曾允许步进订单簿并尝试寻找下一个 ROI 位置——试图提前放置卖出订单。
然而，由于这会增加风险且无实际益处，出于可维护性考虑，该功能已于 2021.7 版本移除。

### 传统超参数优化模式

使用独立的超参数优化文件已于 2021.4 版本弃用，并于 2021.9 版本移除。
请切换到新的 [参数化策略](hyperopt.md) 以受益于新的超参数优化界面。

## V2 与 V3 版本间的策略变更

隔离期货/做空交易于 2022.4 版本引入。这需要对配置设置、策略接口等进行重大更改。

我们投入了大量精力来保持与现有策略的兼容性，因此如果您只想继续在现货市场使用 freqtrade，无需进行任何更改。
虽然我们可能在未来某个时候停止对当前接口的支持，但我们会另行公告并设置适当的过渡期。

请遵循[策略迁移指南](strategy_migration.md)将您的策略迁移至新格式，以开始使用新功能。

### webhooks - 2022.4 版本变更

#### `buy_tag` 已重命名为 `enter_tag`

这仅适用于您的策略以及可能的 webhooks。
我们将保留 1-2 个版本的兼容层（因此 `buy_tag` 和 `enter_tag` 仍将有效），但在此之后 webhooks 中对这一命名的支持将会消失。

#### 命名变更

Webhook 术语从 "sell" 改为 "exit"，从 "buy" 改为 "entry"，并在此过程中移除了 "webhook" 字样。

* `webhookbuy`, `webhookentry` -> `entry`
* `webhookbuyfill`, `webhookentryfill` -> `entry_fill`
* `webhookbuycancel`, `webhookentrycancel` -> `entry_cancel`
* `webhooksell`, `webhookexit` -> `exit`
* `webhooksellfill`, `webhookexitfill` -> `exit_fill`
* `webhooksellcancel`, `webhookexitcancel` -> `exit_cancel`

## 移除 `populate_any_indicators`

2023.3 版本移除了 `populate_any_indicators`，转而采用拆分方法进行特征工程和目标设置。请阅读 [迁移文档](strategy_migration.md#freqai-strategy) 了解完整详情。

## 从配置中移除 `protections`

通过 `"protections": [],` 在配置中设置保护的功能已在 2024.10 版本中移除，此前该功能已发出弃用警告超过 3 年。

## hdf5 数据存储

使用 hdf5 作为数据存储的功能在 2024.12 版本中已被弃用，并已于 2025.1 版本中移除。我们建议切换到 feather 数据格式。

请在更新前使用 [`convert-data` 子命令](data-download.md#sub-command-convert-data) 将现有数据转换为支持的格式之一。

## 通过配置设置高级日志记录

分别通过 `--logfile systemd` 和 `--logfile journald` 配置 syslog 和 journald 的功能已在 2025.3 版本中弃用。
请改用基于配置的 [日志设置](advanced-setup.md#advanced-logging)。

## 移除 edge 模块

edge 模块已于 2023.9 版本弃用，并已于 2025.6 版本移除。
edge 的所有功能已被移除，配置 edge 将导致错误。