![freqai-logo](assets/freqai_doc_logo.svg)

# FreqAI

## 项目说明

FreqAI 是一款旨在自动化与训练预测性机器学习模型相关的各种任务的软件，该模型可根据一组输入信号生成市场预测。总体而言，FreqAI 致力于成为一个沙盒环境，便于在实时数据上部署稳健的机器学习库（[详情](#freqai-position-in-open-source-machine-learning-landscape)）。

!!! Note
    FreqAI 是且将始终是一个非营利性的开源项目。FreqAI *不*发行加密代币，FreqAI *不*出售信号，并且除了当前的 [freqtrade 文档](https://www.freqtrade.io/en/latest/freqai/) 外，FreqAI 没有其他域名。

功能包括：

* **自适应重训练** - 在[实盘部署](freqai-running.md#live-deployments)期间重训练模型，以监督方式自适应市场变化
* **快速特征工程** - 基于用户创建的简单策略生成大规模丰富[特征集](freqai-feature-engineering.md#feature-engineering)（1万+特征）
* **高性能** - 多线程支持在独立线程（或可用GPU上）进行自适应模型重训练，与模型推理（预测）和机器人交易操作并行。最新模型和数据常驻内存以实现快速推理
* **真实回测** - 通过[回测模块](freqai-running.md#backtesting)在历史数据上模拟自适应训练，自动执行重训练过程
* **可扩展性** - 通用且稳健的架构支持集成Python中任何可用的[机器学习库/方法](freqai-configuration.md#using-different-prediction-models)。目前提供八个示例，包括分类器、回归器和卷积神经网络
* **智能异常值剔除** - 使用多种[异常检测技术](freqai-feature-engineering.md#outlier-detection)从训练和预测数据集中移除异常值
* **崩溃恢复** - 将训练好的模型存储到磁盘，实现崩溃后快速轻松重载，并[清理过期文件](freqai-running.md#purging-old-model-data)以维持长期模拟/实盘运行
* **自动数据标准化** - 以智能且统计安全的方式[标准化数据](freqai-feature-engineering.md#building-the-data-pipeline)
* **自动数据下载** - 计算数据下载的时间范围并更新历史数据（实盘部署中）
* **传入数据清洗** - 在训练和模型推理前安全处理缺失值
* **降维处理** - 通过[主成分分析](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)减小训练数据规模
* **部署机器人集群** - 设置一个机器人训练模型，同时由多个[消费者](producer-consumer.md)组成的集群使用信号进行交易

## 快速入门

快速测试 FreqAI 的最简单方法是使用以下命令在模拟模式下运行：

```bash
freqtrade trade --config config_examples/config_freqai.example.json --strategy FreqaiExampleStrategy --freqaimodel LightGBMRegressor --strategy-path freqtrade/templates
```

您将看到自动数据下载的启动过程，随后是同步进行的训练和交易。

!!! danger "非生产环境使用"
    随 Freqtrade 源代码提供的示例策略旨在展示/测试广泛的 FreqAI 功能。该策略也设计为可在小型计算机上运行，以便作为开发者和用户之间的基准测试工具。它*并非*为生产环境运行而设计。

可作为入门参考的示例策略、预测模型和配置文件分别位于：
`freqtrade/templates/FreqaiExampleStrategy.py`、`freqtrade/freqai/prediction_models/LightGBMRegressor.py` 和
`config_examples/config_freqai.example.json`。

## 通用方法

您为 FreqAI 提供一组自定义的*基础指标*（与[典型的 Freqtrade 策略](strategy-customization.md)中的方式相同）以及目标值（*标签*）。对于白名单中的每个交易对，FreqAI 会训练一个模型，根据自定义指标的输入来预测目标值。然后，模型会按照预定频率持续重新训练，以适应市场条件。FreqAI 提供了回测策略（通过历史数据定期重新训练以模拟现实情况）和部署模拟/实盘运行的能力。在模拟/实盘条件下，FreqAI 可以设置为在后台线程中持续重新训练，以保持模型尽可能最新。

下方展示了算法概述，解释了数据处理流程和模型使用方法。

![freqai-algo](assets/freqai_algo.jpg)

### 重要机器学习词汇

**特征** - 基于历史数据的参数，模型在此基础上进行训练。单个蜡烛图的所有特征存储为一个向量。在 FreqAI 中，您可以从策略中构建的任何内容创建特征数据集。

**标签** - 模型训练所朝向的目标值。每个特征向量都与您在策略中定义的单个标签相关联。这些标签有意展望未来，是您训练模型以期能够预测的内容。

**训练** - 指"教导"模型将特征集与对应标签相匹配的过程。不同类型的模型以不同方式"学习"，这意味着针对特定应用场景，某种模型可能比其他模型表现更优。有关 FreqAI 已实现的不同模型的更多信息，请参阅[此处](freqai-configuration.md#using-different-prediction-models)。

**训练数据** - 特征数据集中用于训练期间输入模型以"教导"其预测目标值的子集。这些数据直接影响模型中的权重连接。

**测试数据** - 特征数据集中用于评估训练后模型性能的子集。这些数据不会影响模型内部的节点权重。

**推理** - 将新的未见数据输入已训练模型以进行预测的过程。

## 安装前提条件

常规的 Freqtrade 安装过程会询问是否安装 FreqAI 依赖项。如需使用 FreqAI，请对此问题回答"是"。若未选择安装，可在安装后通过以下命令手动安装依赖：

``` bash
pip install -r requirements-freqai.txt
```

!!! Note
    Catboost 不会在低功耗 ARM 设备（如树莓派）上安装，因为该平台未提供相应的预编译包。

### Docker 环境使用

如果您使用 Docker，我们提供了一个包含 FreqAI 依赖项的专用标签 `:freqai`。因此，您可以将 docker compose 文件中的镜像行替换为 `image: freqtradeorg/freqtrade:stable_freqai`。该镜像包含常规的 FreqAI 依赖项。与原生安装类似，基于 ARM 架构的设备将无法使用 Catboost。如果您希望使用 PyTorch 或强化学习，则应使用对应的 torch 或 RL 标签：`image: freqtradeorg/freqtrade:stable_freqaitorch`、`image: freqtradeorg/freqtrade:stable_freqairl`。

!!! note "docker-compose-freqai.yml"
    我们在 `docker/docker-compose-freqai.yml` 中为此提供了明确的 docker-compose 文件 - 可通过 `docker compose -f docker/docker-compose-freqai.yml run ...` 使用，或可复制该文件以替换原始 docker 文件。此 docker-compose 文件还包含一个（已禁用的）部分，用于在 Docker 容器内启用 GPU 资源。这显然要求系统具备可用的 GPU 资源。

### FreqAI 在开源机器学习领域的定位

预测混沌时间序列系统，例如股票/加密货币市场，需要一套广泛的工具来测试各种假设。幸运的是，近期稳健的机器学习库（如 `scikit-learn`）的成熟为研究开辟了广阔的可能性。来自不同领域的科学家现在可以轻松地在大量成熟的机器学习算法上对其研究进行原型设计。同样，这些用户友好的库使"公民科学家"能够利用其基本的 Python 技能进行数据探索。然而，在历史和实时的混沌数据源上利用这些机器学习库在操作上可能既困难又昂贵。此外，稳健的数据收集、存储和处理带来了不同的挑战。[`FreqAI`](#freqai) 旨在提供一个通用且可扩展的开源框架，专注于市场预测的自适应建模的实时部署。`FreqAI` 框架实际上是丰富的开源机器学习库世界的一个沙盒。在 `FreqAI` 沙盒内，用户发现他们可以结合各种第三方库，在一个免费的、全天候运行的实时混沌数据源——加密货币交易所数据上测试创造性的假设。

### 引用 FreqAI

FreqAI [已在《开源软件杂志》上发表](https://joss.theoj.org/papers/10.21105/joss.04864)。如果您在研究中发现 FreqAI 有用，请使用以下引用方式：

```bibtex
@article{Caulk2022, 
    doi = {10.21105/joss.04864},
    url = {https://doi.org/10.21105/joss.04864},
    year = {2022}, publisher = {The Open Journal},
    volume = {7}, number = {80}, pages = {4864},
    author = {Robert A. Caulk and Elin Törnquist and Matthias Voppichler and Andrew R. Lawless and Ryan McMullan and Wagner Costa Santos and Timothy C. Pogue and Johan van der Vlugt and Stefan P. Gehring and Pascal Schmidt},
    title = {FreqAI: generalizing adaptive modeling for chaotic time-series market forecasts},
    journal = {Journal of Open Source Software} } 
```

## 常见陷阱

FreqAI 不能与动态 `VolumePairlists`（或任何动态添加和移除交易对的配对列表过滤器）结合使用。
这是出于性能考虑——FreqAI 依赖于快速预测/重新训练。为了有效实现这一点，
它需要在回测/实盘实例开始时下载所有训练数据。FreqAI 会自动存储并为未来的重新训练追加
新蜡烛数据。这意味着如果新的交易对在回测过程中由于交易量配对列表而稍后出现，FreqAI 将没有准备好的数据。但是，FreqAI 确实可以与 `ShufflePairlist` 或保持总配对列表不变（但根据交易量重新排序交易对）的 `VolumePairlist` 协同工作。

## 额外学习资料

这里我们整理了一些外部资料，可帮助更深入地了解 FreqAI 的各个组件：

- [实时对决：使用 XGBoost 和 CatBoost 对金融市场数据进行自适应建模](https://emergentmethods.medium.com/real-time-head-to-head-adaptive-modeling-of-financial-market-data-using-xgboost-and-catboost-995a115a7495)
- [FreqAI - 从价格到预测](https://emergentmethods.medium.com/freqai-from-price-to-prediction-6fadac18b665)

## 支持

您可以在多个地方找到 FreqAI 的支持，包括 [Freqtrade discord](https://discord.gg/Jd8JYeWHc4)、专门的 [FreqAI discord](https://discord.gg/7AMWACmbjT) 以及 [github issues](https://github.com/freqtrade/freqtrade/issues)。

## 致谢

FreqAI 由一群为项目贡献各自专业技能的个人开发。

概念构思与软件开发：
Robert Caulk @robcaulk

理论构思与数据分析：
Elin Törnquist @th0rntwig

代码审查与软件架构构思：
@xmatthias

软件开发：
Wagner Costa @wagnercosta
Emre Suzen @aemr3
Timothy Pogue @wizrds

Beta 测试与错误报告：
Stefan Gehring @bloodhunter4rc, @longyu, Andrew Lawless @paranoidandy, Pascal Schmidt @smidelis, Ryan McMullan @smarmau, Juha Nykänen @suikula, Johan van der Vlugt @jooopiert, Richárd Józsa @richardjosza