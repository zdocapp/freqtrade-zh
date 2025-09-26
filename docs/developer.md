# 开发帮助

本页面面向 Freqtrade 的开发者、希望为 Freqtrade 代码库或文档做出贡献的人员，或希望了解所运行应用程序源代码的用户。

我们欢迎所有贡献、错误报告、错误修复、文档改进、功能增强和想法。我们在 [GitHub](https://github.com) 上[跟踪问题](https://github.com/freqtrade/freqtrade/issues)，同时在 [discord](https://discord.gg/p7nuUNVfP7) 设有开发频道供您提问。

## 文档

文档可在 [https://freqtrade.io](https://www.freqtrade.io/) 获取，且每个新功能 PR 都必须附带相应文档。

文档的特殊字段（如注释框等）可在[此处](https://squidfunk.github.io/mkdocs-material/reference/admonitions/)查看。

要本地测试文档，请使用以下命令。

``` bash
pip install -r docs/requirements-docs.txt
mkdocs serve
```

这将启动本地服务器（通常位于端口 8000），以便您检查所有内容是否符合预期。

## 开发者设置

要配置开发环境，您可以使用提供的 [DevContainer](#devContainer-setup)，或者在运行 `setup.sh` 脚本时，当询问 "Do you want to install dependencies for dev [y/N]? " 时回答 "y"。
或者（例如，如果您的系统不受 setup.sh 脚本支持），请遵循手动安装流程，运行 `pip3 install -r requirements-dev.txt`，然后运行 `pip3 install -e .[all]`。

这将安装开发所需的所有工具，包括 `pytest`、`ruff`、`mypy` 和 `coveralls`。

然后通过运行 `pre-commit install` 来安装 git 钩子脚本，这样您的更改在提交前会在本地进行验证。
这已经可以避免大量等待 CI 的时间，因为一些基本的格式检查会在您的本地机器上完成。

在开启拉取请求之前，请熟悉我们的 [贡献指南](https://github.com/freqtrade/freqtrade/blob/develop/CONTRIBUTING.md)。

### Devcontainer 设置

最快且最简单的入门方式是使用 [VSCode](https://code.visualstudio.com/) 及其远程容器扩展。
这使开发人员能够启动机器人，并具备所有必需的依赖项，而*无需*在本地机器上安装任何 freqtrade 特定的依赖项。

#### Devcontainer 依赖项

* [VSCode](https://code.visualstudio.com/)
* [docker](https://docs.docker.com/install/)
* [Remote container extension documentation](https://code.visualstudio.com/docs/remote)

有关 [Remote container extension](https://code.visualstudio.com/docs/remote) 的更多信息，建议查阅官方文档。

### 测试

新代码应包含基础单元测试覆盖。根据功能的复杂程度，评审者可能会要求更深入的单元测试。
如有必要，Freqtrade 团队可协助并提供编写良好测试的指导（但请不要期望有人会为您编写测试）。

#### 如何运行测试

在根目录使用 `pytest` 命令运行所有可用测试用例，并确认本地环境配置正确

!!! Note "功能分支"
    `develop` 和 `stable` 分支上的测试应能通过。其他分支可能处于开发中，测试可能尚未完全通过。

#### 在测试中检查日志内容

Freqtrade 使用两种主要方法在测试中检查日志内容：`log_has()` 和 `log_has_re()`（用于通过正则表达式检查动态日志消息）。
这些方法可从 `conftest.py` 导入，并可在任何测试模块中使用。

示例检查如下所示：

``` python
from tests.conftest import log_has, log_has_re

def test_method_to_test(caplog):
    method_to_test()

    assert log_has("This event happened", caplog)
    # Check regex with trailing number ...
    assert log_has_re(r"This dynamic event happened and produced \d+", caplog)

```

### 调试配置

要调试 freqtrade，我们推荐使用 VSCode（配合 Python 扩展）并采用以下启动配置（位于 `.vscode/launch.json` 中）。
具体配置显然会因环境而异——但这应该能帮你快速上手。

``` json
{
    "name": "freqtrade trade",
    "type": "debugpy",
    "request": "launch",
    "module": "freqtrade",
    "console": "integratedTerminal",
    "args": [
        "trade",
        // Optional:
        // "--userdir", "user_data",
        "--strategy", 
        "MyAwesomeStrategy",
    ]
},
```

命令行参数可在 `"args"` 数组中添加。
通过策略文件中设置断点，此方法也可用于调试策略。

PyCharm 也可采用类似配置——使用 `freqtrade` 作为模块名，并将命令行参数设为“参数”。

??? Tip "正确使用虚拟环境"
    使用虚拟环境（推荐）时，请确保编辑器使用了正确的虚拟环境，以避免问题或“未知导入”错误。

    #### Vscode

    You can select the correct environment in VSCode with the command "Python: Select Interpreter" - which will show you environments the extension detected.
    If your environment has not been detected, you can also pick a path manually.

    #### Pycharm

    In pycharm, you can select the appropriate Environment in the "Run/Debug Configurations" window.
    ![Pycharm debug configuration](assets/pycharm_debug.png)

!!! Note "启动目录"
    此配置假设您已检出代码仓库，且编辑器在仓库根目录启动（即 pyproject.toml 位于仓库顶层）。

## 错误处理

Freqtrade 的所有异常均继承自 `FreqtradeException`。
但不应直接使用此通用错误类，而是存在多个专门的子异常类。

以下是异常继承层次结构概览：

```
+ FreqtradeException
|
+---+ OperationalException
|   |
|   +---+ ConfigurationError
|
+---+ DependencyException
|   |
|   +---+ PricingError
|   |
|   +---+ ExchangeError
|       |
|       +---+ TemporaryError
|       |
|       +---+ DDosProtection
|       |
|       +---+ InvalidOrderException
|           |
|           +---+ RetryableOrderError
|           |
|           +---+ InsufficientFundsError
|
+---+ StrategyError
```

---

## 插件

### 交易对列表

您对新的交易对筛选算法有绝妙想法？太棒了。
希望您也愿意将其贡献给上游项目。

无论您的动机是什么——这应该能让您开始尝试开发一个新的交易对列表处理器。

首先，请查看 [VolumePairList](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/plugins/pairlist/VolumePairList.py) 处理器，最好将此文件复制并命名为您的新交易对列表处理器。

这是一个简单的处理器，但可以作为开始开发的良好示例。

接下来，修改处理器的类名（最好与模块文件名保持一致）。

基类提供了交易所实例（`self._exchange`）、交易对列表管理器（`self._pairlistmanager`），以及主配置（`self._config`）、交易对列表专用配置（`self._pairlistconfig`）和该处理器在交易对列表链中的绝对位置。

```python
        self._exchange = exchange
        self._pairlistmanager = pairlistmanager
        self._config = config
        self._pairlistconfig = pairlistconfig
        self._pairlist_pos = pairlist_pos
```

!!! Tip
    不要忘记在 `constants.py` 文件的 `AVAILABLE_PAIRLISTS` 变量下注册您的交易对列表——否则它将无法被选择使用。

现在，让我们逐步了解需要操作的方法：

#### 交易对列表配置

交易对列表处理器链的配置在机器人配置文件的 `"pairlists"` 元素中完成，这是一个为链中每个交易对列表处理器配置参数的数组。

按照惯例，`"number_assets"` 用于指定交易对列表中保留的最大交易对数量。请遵循此约定以确保一致的用户体验。

可以根据需要配置其他参数。例如，`VolumePairList` 使用 `"sort_key"` 来指定排序值——但请随意指定您的优秀算法成功运行并保持动态所需的任何参数。

#### short_desc

返回用于 Telegram 消息的描述。

该描述应包含配对列表处理程序的名称，以及包含资产数量的简短说明。请遵循 `"PairlistName - top/bottom X pairs"` 格式。

#### gen_pairlist

如果配对列表处理程序可用作链中的主导配对列表处理程序（用于定义初始配对列表，然后由链中的所有配对列表处理程序处理），请重写此方法。例如 `StaticPairList` 和 `VolumePairList`。

该方法在机器人每次迭代时都会被调用（仅当配对列表处理程序位于链首时）——因此请考虑为计算/网络密集型计算实施缓存。

它必须返回最终的配对列表（该列表随后可能被传递到配对列表处理程序链中）。

验证是可选的，父类提供了 `verify_blacklist(pairlist)` 和 `_whitelist_for_active_markets(pairlist)` 方法来进行默认过滤。如果您将结果限制在特定数量的交易对范围内，请使用此功能——以免最终结果比预期短。

#### filter_pairlist

配对列表管理器会为链中的每个配对列表处理程序调用此方法。

每次机器人迭代时都会调用此方法——因此对于计算/网络密集型计算，请考虑实现缓存。

它会接收一个交易对列表（可能是先前交易对列表处理器的结果）以及 `tickers`，即预获取的 `get_tickers()` 版本。

基类中的默认实现仅对交易对列表中的每个交易对调用 `_validate_pair()` 方法，但您可以重写它。因此，您应该在自己的交易对列表处理器中实现 `_validate_pair()` 方法，或者重写 `filter_pairlist()` 以执行其他操作。

如果被重写，它必须返回处理后的交易对列表（该列表随后可能传递给链中的下一个交易对列表处理器）。

验证是可选的，父类提供了 `verify_blacklist(pairlist)` 和 `_whitelist_for_active_markets(pairlist)` 方法来执行默认过滤。如果您将结果限制在特定数量的交易对内，请使用此功能——这样最终结果就不会比预期的短。

在 `VolumePairList` 中，此方法实现了不同的排序方式，并进行早期验证，以便只返回预期数量的交易对。

##### 示例

``` python
    def filter_pairlist(self, pairlist: list[str], tickers: dict) -> List[str]:
        # Generate dynamic whitelist
        pairs = self._calculate_pairlist(pairlist, tickers)
        return pairs
```

### 保护机制

建议先阅读[保护机制文档](plugins.md#protections)以了解保护机制。
本指南面向希望开发新保护机制的开发者。

不应直接使用 datetime，而应使用提供的 `date_now` 变量进行日期计算。这保留了回测保护功能的能力。

!!! Tip "编写新保护规则"
    最好复制现有的保护规则之一作为参考示例。

#### 新保护规则的实现

所有保护规则实现都必须以 `IProtection` 作为父类。
因此，它们必须实现以下方法：

* `short_desc()`
* `global_stop()`
* `stop_per_pair()`

`global_stop()` 和 `stop_per_pair()` 必须返回一个 ProtectionReturn 对象，该对象包含：

* 锁定交易对 - 布尔值
* 锁定截止时间 - datetime - 交易对应锁定至何时（将向上取整到下一个新K线）
* 原因 - 字符串，用于日志记录和数据库存储
* 锁定方向 - 做多、做空或 '*'（全部）

截止时间部分应使用提供的 `calculate_lock_end()` 方法计算。

所有保护规则都应使用 `"stop_duration"` / `"stop_duration_candles"` 来定义交易对（或所有交易对）应锁定的时长。
该内容以 `self._stop_duration` 的形式提供给每个保护规则。

如果您的保护规则需要回溯周期，请使用 `"lookback_period"` / `"lockback_period_candles"` 以保持所有保护规则的一致性。

#### 全局停止与本地停止

保护规则有两种不同的方式在有限时间内停止交易：

* 每交易对（本地）
* 所有交易对（全局）

##### 保护机制 - 每交易对

采用每交易对方法的保护机制必须设置 `has_local_stop=True`。
当交易关闭（退出订单完成）时，将调用 `stop_per_pair()` 方法。

##### 保护机制 - 全局保护

这些保护机制应对所有交易对进行评估，因此也将锁定所有交易对的交易（称为全局交易对锁定）。
全局保护必须设置 `has_global_stop=True` 才能进行全局停止评估。
当交易关闭（退出订单完成）时，将调用 `global_stop()` 方法。

##### 保护机制 - 计算锁定结束时间

保护机制应根据其考虑的最后一笔交易计算锁定结束时间。
这样可以避免在回看周期长于实际锁定周期时重新锁定。

`IProtection` 父类在 `calculate_lock_end()` 中为此提供了一个辅助方法。

---

## 实现新的交易所（进行中）

!!! Note
    本节为进行中的工作，并非关于如何使用 Freqtrade 测试新交易所的完整指南。

!!! Note
    在运行以下任何测试之前，请确保使用最新版本的 CCXT。
    您可以通过在激活的虚拟环境中运行 `pip install -U ccxt` 来获取最新版本的 ccxt。
    这些测试不支持原生 docker，但可用的开发容器将支持所有必需的操作以及最终必要的更改。

CCXT 支持的大多数交易所应该可以开箱即用。

如果您需要实现特定的交易所类，可以在 `freqtrade/exchange` 源代码文件夹中找到它们。您还需要将导入添加到 `freqtrade/exchange/__init__.py`，以使加载逻辑感知到新的交易所。
我们建议查看现有的交易所实现，以了解可能需要的内容。

!!! Warning
    实现和测试一个交易所可能需要大量的试错，请牢记这一点。
    您还应该具备一些开发经验，因为这不是一项适合初学者的任务。

要快速测试交易所的公共端点，请将您的交易所配置添加到 `tests/exchange_online/conftest.py`，并使用 `pytest --longrun tests/exchange_online/test_ccxt_compat.py` 运行这些测试。
成功完成这些测试是一个良好的基础点（实际上是一个要求），但这并不能保证交易所功能完全正确，因为这仅测试公共端点，而不测试私有端点（如下单或类似操作）。

同时尝试使用 `freqtrade download-data` 下载扩展时间段（数月）的数据，并验证数据是否正确下载（无数据缺口，实际下载了指定时间段）。

这些是将交易所列为"支持"或"社区测试"（在主页列出）的先决条件。
以下是"附加项"，它们将使交易所更完善（功能完整）——但对于这两个类别并非绝对必要。

需要完成的额外测试/步骤：

* 验证 `fetch_ohlcv()` 提供的数据——并最终调整该交易所的 `ohlcv_candle_limit`
* 检查L2订单簿限制范围（API文档）——并最终根据需要设置
* 检查余额显示是否正确（*）
* 创建市价订单（*）
* 创建限价订单（*）
* 取消订单（*）
* 完成交易（入场+出场）（*）
  * 比较交易所和机器人之间的结果计算
  * 确保费用正确应用（根据交易所核对数据库）

（*）需要API密钥和交易所余额。

### 交易所止损功能

检查新交易所是否通过其API支持交易所止损订单。

由于 CCXT 尚未提供交易所止损功能的统一实现，我们需要自行实现特定交易所的参数。最好参考 `binance.py` 中的示例实现。你需要仔细查阅交易所 API 文档，了解具体如何实现。[CCXT Issues](https://github.com/ccxt/ccxt/issues) 也可能提供很大帮助，因为其他人可能已经为他们的项目实现了类似功能。

### 不完整的K线数据

在获取K线（OHLCV）数据时，我们可能会得到不完整的K线（取决于交易所）。
为了演示这一点，我们将使用日线（`"1d"`）来简化说明。
我们通过API（`ct.fetch_ohlcv()`）查询对应时间周期的数据，并查看最后一条记录的日期。如果这条记录发生变化或显示"不完整"K线的日期，那么我们应该丢弃它，因为不完整的K线会带来问题——指标计算假设只接收完整的K线数据，否则会产生大量错误买入信号。因此，默认情况下我们会移除最后一根K线，假设它是不完整的。

要检查新交易所的行为方式，你可以使用以下代码片段：

``` python
import ccxt
from datetime import datetime, timezone
from freqtrade.data.converter import ohlcv_to_dataframe
ct = ccxt.binance()  # Use the exchange you're testing
timeframe = "1d"
pair = "BTC/USDT"  # Make sure to use a pair that exists on that exchange!
raw = ct.fetch_ohlcv(pair, timeframe=timeframe)

# convert to dataframe
df1 = ohlcv_to_dataframe(raw, timeframe, pair=pair, drop_incomplete=False)

print(df1.tail(1))
print(datetime.now(timezone.utc))
```

``` output
                         date      open      high       low     close  volume  
499 2019-06-08 00:00:00+00:00  0.000007  0.000007  0.000007  0.000007   26264344.0  
2019-06-09 12:30:27.873327
```

输出将显示交易所的最后一条记录以及当前的UTC日期。
如果日期显示为同一天，则可以假定最后一根K线不完整并应丢弃（保持交易所类中的 `"ohlcv_partial_candle"` 设置不变 / True）。否则，将 `"ohlcv_partial_candle"` 设置为 `False` 以保留K线（如上例所示）。
另一种方法是连续多次运行此命令，并观察交易量是否发生变化（而日期保持不变）。

### 更新币安缓存的杠杆层级

更新杠杆层级应定期进行 - 并且需要一个已启用期货功能的认证账户。

``` python
import ccxt
import json
from pathlib import Path

exchange = ccxt.binance({
    'apiKey': '<apikey>',
    'secret': '<secret>',
    'options': {'defaultType': 'swap'}
    })
_ = exchange.load_markets()

lev_tiers = exchange.fetch_leverage_tiers()

# Assumes this is running in the root of the repository.
file = Path('freqtrade/exchange/binance_leverage_tiers.json')
json.dump(dict(sorted(lev_tiers.items())), file.open('w'), indent=2)

```

该文件应向上游贡献，以便其他人也能从中受益。

## 更新示例笔记本

为使Jupyter笔记本与文档保持一致，在更新示例笔记本后应运行以下命令。

``` bash
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --inplace freqtrade/templates/strategy_analysis_example.ipynb
jupyter nbconvert --ClearOutputPreprocessor.enabled=True --to markdown freqtrade/templates/strategy_analysis_example.ipynb --stdout > docs/strategy_analysis_example.md
```

## 回测文档结果

要生成回测输出，请使用以下命令：

``` bash
# Assume a dedicated user directory for this output
freqtrade create-userdir --userdir user_data_bttest/
# set can_short = True
sed -i "s/can_short: bool = False/can_short: bool = True/" user_data_bttest/strategies/sample_strategy.py

freqtrade download-data --timerange 20250625-20250801 --config tests/testdata/config.tests.usdt.json --userdir user_data_bttest/ -t 5m

freqtrade backtesting --config tests/testdata/config.tests.usdt.json -s SampleStrategy --userdir user_data_bttest/ --cache none --timerange 20250701-20250801
```

## 持续集成

本文档记录了CI流水线的一些决策。

* CI 在所有操作系统变体上运行，包括 Linux (ubuntu)、macOS 和 Windows。
* 为 `stable` 和 `develop` 分支构建 Docker 镜像，并作为多架构构建，通过同一标签支持多个平台。
* 包含 Plot 依赖项的 Docker 镜像也可用，标签为 `stable_plot` 和 `develop_plot`。
* Docker 镜像包含一个文件 `/freqtrade/freqtrade_commit`，其中记录了该镜像所基于的提交。
* 完整的 Docker 镜像重建每周通过定时任务运行一次。
* 部署在 ubuntu 上运行。
* 所有测试必须通过，PR 才能合并到 `stable` 或 `develop` 分支。

## 创建版本

本文档的这部分面向维护者，展示了如何创建版本。

### 创建发布分支

!!! Note
    请确保 `stable` 分支是最新的！

首先，选择一个大约一周前的提交（以避免将最新添加的内容包含到版本中）。

``` bash
# create new branch
git checkout -b new_release <commitid>
```

确定在此提交与当前状态之间是否进行了关键的错误修复，并最终挑选这些修复。

* 将发布分支（stable）合并到此分支。
* 编辑 `freqtrade/__init__.py` 文件，添加与当前日期匹配的版本号（例如 2019年7月发布则使用 `2019.7`）。若同月需要二次发布，次要版本可为 `2019.7.1`。版本号必须遵循 PEP0440 允许的格式，以避免推送至 pypi 时失败。
* 提交这部分更改。
* 将该分支推送到远程仓库，并针对 **stable 分支** 创建 PR。
* 将开发版本更新为遵循 `2019.8-dev` 模式的下一个版本。

### 基于 git 提交记录生成更新日志

``` bash
# Needs to be done before merging / pulling that branch.
git log --oneline --no-decorate --no-merges stable..new_release
```

为保持发布日志简洁，建议将完整的 git 更新日志包裹在可折叠的 details 区块中。

```markdown
<details>
<summary>Expand full changelog</summary>

... Full git changelog

</details>
```

### FreqUI 发布

若 FreqUI 有重大更新，请确保在合并发布分支前先创建发布。
确保 FreqUI 在发布分支上的 CI 已完成并通过，再执行合并。

### 创建 GitHub 发布 / 标签

当针对 stable 的 PR 合并后（建议在合并后立即操作）：

* 在 GitHub UI 的发布版块点击 "Draft a new release" 按钮。
* 使用指定的版本号作为标签。
* 选择 "stable" 作为参考分支（此步骤在上述 PR 合并后进行）。
* 将前述更新日志作为发布说明（以代码块形式呈现）。
* 使用以下代码片段作为新发布模板

??? Tip "发布模板"
    ````
    --8<-- "includes/release_template.md"
    ````

## 发布流程

### pypi

!!! Warning "Manual Releases"
    此过程已作为 Github Actions 的一部分实现自动化。  
    通常无需手动推送至 pypi。

??? example "Manual release"
    如需手动创建 pypi 发布版本，请运行以下命令：

    Additional requirement: `wheel`, `twine` (for uploading), account on pypi with proper permissions.

    ``` bash
    pip install -U build
    python -m build --sdist --wheel

    # For pypi test (to check if some change to the installation did work)
    twine upload --repository-url https://test.pypi.org/legacy/ dist/*

    # For production:
    twine upload dist/*
    ```

    Please don't push non-releases to the productive / real pypi instance.