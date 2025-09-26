# 参数表格

下表将列出 FreqAI 所有可用的配置参数。部分参数示例可在 `config_examples/config_freqai.example.json` 中找到。

标有 **必需** 的参数为必填项，必须通过建议的方式之一进行设置。

### 通用配置参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`config.freqai` 树结构中的通用配置参数** |
| `freqai` | **必需。** <br> 包含所有控制 FreqAI 参数的主字典。 <br> **数据类型：** 字典。 |
| `train_period_days` | **必需。** <br> 用于训练数据的天数（滑动窗口的宽度）。 <br> **数据类型：** 正整数。 |
| `backtest_period_days` | **必需。** <br> 在回测期间，从训练模型进行推理的天数，然后滑动上述定义的 `train_period_days` 窗口并重新训练模型（更多信息请参见[此处](freqai-running.md#backtesting)）。可以是小数天，但请注意，提供的 `timerange` 将除以该数字以得出完成回测所需的训练次数。 <br> **数据类型：** 浮点数。 |
| `identifier` | **必需。** <br> 当前模型的唯一标识符。如果将模型保存到磁盘，`identifier` 允许重新加载特定的预训练模型/数据。 <br> **数据类型：** 字符串。 |
| `live_retrain_hours` | 在模拟/实盘运行期间重新训练的频率。 <br> **数据类型：** 浮点数 > 0。 <br> 默认值：`0`（模型尽可能频繁地重新训练）。 |
| `expiration_hours` | 如果模型超过 `expiration_hours` 旧，则避免进行预测。 <br> **数据类型：** 正整数。 <br> 默认值：`0`（模型永不过期）。 |
| `purge_old_models` | 保留在磁盘上的模型数量（与回测无关）。默认值为 2，这意味着模拟/实盘运行将保留最新的 2 个模型在磁盘上。设置为 0 则保留所有模型。此参数也接受布尔值以保持向后兼容性。 <br> **数据类型：** 整数。 <br> 默认值：`2`。 |
| `save_backtest_models` | 在运行回测时将模型保存到磁盘。回测通过保存预测数据并直接将其重用于后续运行（当您希望调整入场/出场参数时）来最高效地操作。将回测模型保存到磁盘还允许使用相同的模型文件启动具有相同模型 `identifier` 的模拟/实盘实例。 <br> **数据类型：** 布尔值。 <br> 默认值：`False`（不保存模型）。 |
| `fit_live_predictions_candles` | 用于根据预测数据计算目标（标签）统计量的历史蜡烛图数量，而不是使用训练数据集（更多信息请参见[此处](freqai-configuration.md#creating-a-dynamic-target-threshold)）。 <br> **数据类型：** 正整数。 |
| `continual_learning` | 使用最近训练模型的最终状态作为新模型的起点，允许增量学习（更多信息请参见[此处](freqai-running.md#continual-learning)）。请注意，这目前是一种朴素的增量学习方法，当市场偏离您的模型时，它有很大概率过拟合/陷入局部最小值。我们在此处提供连接主要是为了实验目的，并为在像加密货币市场这样的混沌系统中采用更成熟的持续学习方法做好准备。 <br> **数据类型：** 布尔值。 <br> 默认值：`False`。 |
| `write_metrics_to_disk` | 将训练计时、推理计时和 CPU 使用情况收集到 json 文件中。 <br> **数据类型：** 布尔值。 <br> 默认值：`False`。 |
| `data_kitchen_thread_count` | <br> 指定用于数据处理（异常值方法、归一化等）的线程数。这对用于训练的线程数没有影响。如果用户未设置（默认），FreqAI 将使用最大线程数 - 2（为 Freqtrade 机器人和 FreqUI 留出 1 个物理核心）。 <br> **数据类型：** 正整数。 |
| `activate_tensorboard` | <br> 指示是否为支持 tensorboard 的模块（当前为强化学习、XGBoost、Catboost 和 PyTorch）激活 tensorboard。Tensorboard 需要安装 Torch，这意味着您需要 torch/RL docker 镜像，或者需要在安装问题时回答“是”关于是否希望安装 Torch。 <br> **数据类型：** 布尔值。 <br> 默认值：`True`。 |
| `wait_for_training_iteration_on_reload` | <br> 当使用 /reload 或 ctrl-c 时，等待当前训练迭代完成后再完成优雅关闭。如果设置为 `False`，FreqAI 将中断当前训练迭代，允许您更快地优雅关闭，但您将丢失当前训练迭代。 <br> **数据类型：** 布尔值。 <br> 默认值：`True`。 |

### 功能参数

| 参数 | 描述 |
|------------|-------------|
|  |  **`freqai.feature_parameters` 子字典内的特征参数** |
| `feature_parameters` | 包含用于构建特征集的参数字典。详细说明和示例见[此处](freqai-feature-engineering.md)。<br> **数据类型：** 字典。 |
| `include_timeframes` | 时间帧列表，`feature_engineering_expand_*()` 中的所有指标将为此列表中的每个时间帧创建。这些指标会作为特征添加到基础指标数据集中。<br> **数据类型：** 时间帧字符串列表。 |
| `include_corr_pairlist` | 相关币种列表，FreqAI 会将这些币种作为额外特征添加到所有 `pair_whitelist` 币种中。在特征工程期间（详见[此处](freqai-feature-engineering.md)），`feature_engineering_expand_*()` 中设置的所有指标将为每个相关币种创建。相关币种的特征会被添加到基础指标数据集中。<br> **数据类型：** 资产字符串列表。 |
| `label_period_candles` | 创建标签时所展望的未来蜡烛数。可在 `set_freqai_targets()` 中使用（详细用法参见 `templates/FreqaiExampleStrategy.py`）。此参数并非必需，您可以创建自定义标签并选择是否使用此参数。请参阅 `templates/FreqaiExampleStrategy.py` 查看示例用法。<br> **数据类型：** 正整数。 |
| `include_shifted_candles` | 将之前蜡烛的特征添加到后续蜡烛中，旨在增加历史信息。如果启用，FreqAI 会复制并平移前 `include_shifted_candles` 根蜡烛的所有特征，使得这些信息对后续蜡烛可用。<br> **数据类型：** 正整数。 |
| `weight_factor` | 根据数据点的新近程度对训练数据点进行加权（详见[此处](freqai-feature-engineering.md#weighting-features-for-temporal-importance)）。<br> **数据类型：** 正浮点数（通常 < 1）。 |
| `indicator_max_period_candles` | **不再使用 (#7325)**。已被[策略](freqai-configuration.md#building-a-freqai-strategy)中设置的 `startup_candle_count` 替代。`startup_candle_count` 与时间帧无关，用于定义 `feature_engineering_*()` 中创建指标时使用的最大*周期*。FreqAI 使用此参数与 `include_time_frames` 中的最大时间帧一起计算需要下载的数据点数，以确保第一个数据点不包含 NaN。<br> **数据类型：** 正整数。 |
| `indicator_periods_candles` | 计算指标的时间周期。这些指标会被添加到基础指标数据集中。<br> **数据类型：** 正整数列表。 |
| `principal_component_analysis` | 使用主成分分析自动降低数据集的维度。工作原理详见[此处](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis)。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `plot_feature_importances` | 为每个模型创建特征重要性图，显示顶部/底部 `plot_feature_importances` 个特征。图表存储在 `user_data/models/<identifier>/sub-train-<COIN>_<timestamp>.html`。<br> **数据类型：** 整数。<br> 默认值：`0`。 |
| `DI_threshold` | 当设置大于 0 时，激活使用相异指数进行异常值检测。工作原理详见[此处](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)。<br> **数据类型：** 正浮点数（通常 < 1）。 |
| `use_SVM_to_remove_outliers` | 训练支持向量机来检测并移除训练数据集以及传入数据点中的异常值。工作原理详见[此处](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)。<br> **数据类型：** 布尔值。 |
| `svm_params` | Sklearn 的 `SGDOneClassSVM()` 中所有可用的参数。部分选定参数的详细说明见[此处](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm)。<br> **数据类型：** 字典。 |
| `use_DBSCAN_to_remove_outliers` | 使用 DBSCAN 算法对数据进行聚类，以识别并移除训练和预测数据中的异常值。工作原理详见[此处](freqai-feature-engineering.md#identifying-outliers-with-dbscan)。<br> **数据类型：** 布尔值。 |
| `noise_standard_deviation` | 如果设置，FreqAI 会向训练特征添加噪声，旨在防止过拟合。FreqAI 生成标准差为 `noise_standard_deviation` 的高斯分布随机偏差，并将其添加到所有数据点。`noise_standard_deviation` 应相对于归一化空间保持，即介于 -1 和 1 之间。换句话说，由于 FreqAI 中的数据总是归一化到 -1 和 1 之间，设置 `noise_standard_deviation: 0.05` 将导致 32% 的数据随机增加/减少超过 2.5%（即落在第一个标准差内的数据百分比）。<br> **数据类型：** 整数。<br> 默认值：`0`。 |
| `outlier_protection_percentage` | 启用后可防止异常值检测方法丢弃过多数据。如果 SVM 或 DBSCAN 检测到的异常值点超过 `outlier_protection_percentage`%，FreqAI 将记录警告信息并忽略异常值检测，即保持原始数据集完整。如果触发异常值保护，将不会基于该训练数据集进行任何预测。<br> **数据类型：** 浮点数。<br> 默认值：`30`。 |
| `reverse_train_test_order` | 分割特征数据集（见下文），并使用最新的数据分割进行训练，在历史数据分割上进行测试。这允许模型训练到最新的数据点，同时避免过拟合。但是，在使用此参数前，您应谨慎理解其非传统性质。<br> **数据类型：** 布尔值。<br> 默认值：`False`（不反转）。 |
| `shuffle_after_split` | 将数据分割为训练集和测试集，然后分别对两个集合进行洗牌。<br> **数据类型：** 布尔值。<br> 默认值：`False`。 |
| `buffer_train_data_candles` | 在指标填充*之后*，从训练数据的开头和结尾截去 `buffer_train_data_candles` 根蜡烛。主要用例是当预测极大值和极小值时，argrelextrema 函数无法知道时间范围边缘的极大值/极小值。为了提高模型准确性，最好在整个时间范围上计算 argrelextrema，然后使用此函数通过内核截去边缘（缓冲区）。另一种情况，如果目标设置为平移的价格变动，则此缓冲区是不必要的，因为时间范围末尾的平移蜡烛将是 NaN，FreqAI 会自动将这些从训练数据集中截去。<br> **数据类型：** 整数。<br> 默认值：`0`。

### 数据分割参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.data_split_parameters` 子字典中的数据分割参数**
| `data_split_parameters` | 包含 scikit-learn `test_train_split()` 中可用的任何附加参数，这些参数显示在[此处](https://scikit-learn.org/stable/modules/generated/sklearn.model_selection.train_test_split.html)（外部网站）。 <br> **数据类型：** 字典。
| `test_size` | 应用于测试而非训练的数据比例。 <br> **数据类型：** 小于1的正浮点数。
| `shuffle` | 在训练期间打乱训练数据点。通常在时间序列预测中为了不破坏数据的时序性，此参数设置为 `False`。 <br> **数据类型：** 布尔值。 <br> 默认值：`False`。

### 模型训练参数

| 参数 | 描述 |
|------------|-------------|
|  |  **`freqai.model_training_parameters` 子字典内的模型训练参数** |
| `model_training_parameters` | 一个灵活的字典，包含所选模型库提供的所有可用参数。例如，如果您使用 `LightGBMRegressor`，此字典可以包含 `LightGBMRegressor` [此处](https://lightgbm.readthedocs.io/en/latest/pythonapi/lightgbm.LGBMRegressor.html)（外部网站）提供的任何参数。如果您选择不同的模型，此字典可以包含该模型的任何参数。当前可用模型的列表可以在[此处](freqai-configuration.md#using-different-prediction-models)找到。<br> **数据类型：** 字典。 |
| `n_estimators` | 在模型训练中要拟合的提升树的数量。<br> **数据类型：** 整数。 |
| `learning_rate` | 模型训练期间的提升学习率。<br> **数据类型：** 浮点数。 |
| `n_jobs`, `thread_count`, `task_type` | 设置并行处理的线程数以及 `task_type`（`gpu` 或 `cpu`）。不同的模型库使用不同的参数名称。<br> **数据类型：** 浮点数。 |

### 强化学习参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.rl_config` 子字典中的强化学习参数** |
| `rl_config` | 包含强化学习模型控制参数的字典。<br> **数据类型:** 字典。 |
| `train_cycles` | 训练时间步长将根据 `训练周期数 * 训练数据点数` 来设置。<br> **数据类型:** 整数。 |
| `max_trade_duration_candles`| 指导智能体训练将交易保持在期望长度以下。示例用法见 `prediction_models/ReinforcementLearner.py` 中可自定义的 `calculate_reward()` 函数。<br> **数据类型:** int。 |
| `model_type` | 来自 stable_baselines3 或 SBcontrib 的模型字符串。可用字符串包括：`'TRPO', 'ARS', 'RecurrentPPO', 'MaskablePPO', 'PPO', 'A2C', 'DQN'`。用户应通过查阅其文档确保 `model_training_parameters` 与相应 stable_baselines3 模型可用的参数匹配。[PPO 文档](https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html)（外部网站）<br> **数据类型:** 字符串。 |
| `policy_type` | stable_baselines3 中可用的策略类型之一。<br> **数据类型:** 字符串。 |
| `max_training_drawdown_pct` | 智能体在训练期间允许经历的最大回撤。<br> **数据类型:** 浮点数。<br> 默认值: 0.8 |
| `cpu_count` | 分配给强化学习训练过程的线程/CPU 数量（取决于是否选择了 `ReinforcementLearning_multiproc`）。建议保持此值不变，默认情况下，此值设置为物理核心总数减 1。<br> **数据类型:** int。 |
| `model_reward_parameters` | 在 `ReinforcementLearner.py` 的可自定义 `calculate_reward()` 函数内部使用的参数。<br> **数据类型:** int。 |
| `add_state_info` | 告知 FreqAI 在训练和推理的特征集中包含状态信息。当前状态变量包括交易持续时间、当前利润、交易头寸。这仅在模拟/实盘运行中可用，在回测时会自动切换为 false。<br> **数据类型:** 布尔值。<br> 默认值: `False`。 |
| `net_arch` | 网络架构，在 [`stable_baselines3` 文档](https://stable-baselines3.readthedocs.io/en/master/guide/custom_policy.html#examples)中有详细描述。简而言之：`[<共享层>, dict(vf=[<非共享价值网络层>], pi=[<非共享策略网络层>])]`。默认设置为 `[128, 128]`，这定义了 2 个共享隐藏层，每层有 128 个单元。 |
| `randomize_starting_position` | 随机化每个训练周期的起始点以避免过拟合。<br> **数据类型:** 布尔值。<br> 默认值: `False`。 |
| `drop_ohlc_from_features` | 在训练期间传递给智能体的特征集中不包含归一化的 OHLC 数据（在所有情况下，OHLC 仍将用于驱动环境）。<br> **数据类型:** 布尔值。<br> **默认值:** `False` |
| `progress_bar` | 显示进度条，包含当前进度、已用时间和预计剩余时间。<br> **数据类型:** 布尔值。<br> 默认值: `False`。

### PyTorch 参数

#### 常规参数

|  参数 | 描述 |
|------------|-------------|
|  |  **`freqai.model_training_parameters` 子字典中的模型训练参数**
| `learning_rate` | 传递给优化器的学习率。 <br> **数据类型:** 浮点数。 <br> 默认值: `3e-4`。
| `model_kwargs` | 传递给模型类的参数。 <br> **数据类型:** 字典。 <br> 默认值: `{}`。
| `trainer_kwargs` | 传递给训练器类的参数。 <br> **数据类型:** 字典。 <br> 默认值: `{}`。

#### trainer_kwargs

| 参数        | 描述 |
|--------------|-------------|
|              |  **`freqai.model_training_parameters.model_kwargs` 子字典内的模型训练参数**
| `n_epochs`   | `n_epochs` 参数是 PyTorch 训练循环中的一个关键设置，它决定了整个训练数据集用于更新模型参数的次数。一个 epoch 表示对整个训练数据集的一次完整遍历。覆盖 `n_steps`。必须设置 `n_epochs` 或 `n_steps` 中的一个。<br><br> **数据类型:** int. 可选. <br> 默认值: `10`。
| `n_steps`    | 设置 `n_epochs` 的另一种方式 - 要运行的训练迭代次数。此处的迭代指的是我们调用 `optimizer.step()` 的次数。如果设置了 `n_epochs`，则忽略此参数。该函数的简化版本为：<br><br> n_epochs = n_steps / (n_obs / batch_size) <br><br> 这里的动机是 `n_steps` 更容易优化，并且在不同 n_obs（数据点数量）下保持稳定。<br> <br> **数据类型:** int. 可选. <br> 默认值: `None`。
| `batch_size` | 训练期间使用的批次大小。<br><br> **数据类型:** int. <br> 默认值: `64`。

### 附加参数

|  参数 | 描述 |
|------------|-------------|
|  |  **额外参数**
| `freqai.keras` | 如果所选模型使用了 Keras（通常基于 TensorFlow 的预测模型），则需要激活此标志，以便模型保存/加载遵循 Keras 标准。<br> **数据类型：** 布尔值。<br> 默认值：`False`。
| `freqai.conv_width` | 神经网络输入张量的宽度。该参数通过将历史数据点作为张量的第二维度输入，取代了移动蜡烛线（`include_shifted_candles`）的需求。从技术上讲，此参数也可用于回归器，但只会增加计算开销，不会改变模型训练/预测。<br> **数据类型：** 整数。<br> 默认值：`2`。
| `freqai.reduce_df_footprint` | 将所有数值列重新转换为 float32/int32 类型，旨在减少内存/磁盘使用量并缩短训练/推理时间。此参数在 Freqtrade 配置文件的主层级设置（不在 FreqAI 内部）。<br> **数据类型：** 布尔值。<br> 默认值：`False`。