# DiodeFitter 操作手册

本手册与工程当前代码保持同步，后续新增功能我会继续更新此文档。

## 快速开始

1. 打开 MATLAB，切换到工程根目录：
   ```matlab
   cd d:\VScode\功率模块_体二极管_20260118
   addpath('DiodeFitter')
   ```
2. 只做光滑/内插（两种方法一起跑）：
   ```matlab
   preprocess_only
   ```
3. 选择要使用的平滑数据（在配置里切换）：
   ```matlab
   % 在 DiodeFitter/config_fit.m 中设置
   cfg.preprocess.select = 'vgs'; % 或 'nn3'
   ```
4. 开启多模型竞技并拟合：
   ```matlab
   % 在 DiodeFitter/config_fit.m 中设置
   cfg.model.benchmark.enable = true;
   cfg.model.benchmark.models = {'model_rohm_idi', 'model_data_poly'};
   ```
   然后运行：
   ```matlab
   fit_only
   % 或 main
   ```
5. 若当前只做平滑内插，到第 3 步即可，不需要运行拟合流程。

## 标准流程（多模型竞技）

1. `preprocess_only` 生成两套平滑数据（Vgs 分组与 3 输入 1 输出）。
2. 在 `DiodeFitter/config_fit.m` 中选择要用于拟合的平滑数据：
   ```matlab
   cfg.preprocess.select = 'vgs'; % 或 'nn3'
   ```
3. 在 `DiodeFitter/config_fit.m` 中配置参赛模型与评分指标：
   ```matlab
   cfg.model.benchmark.enable = true;
   cfg.model.benchmark.metric = 'AIC';
   cfg.model.benchmark.models = {'model_rohm_idi', 'model_data_poly'};
   ```
4. 运行 `fit_only`（或 `main`），自动输出对比图与评分表。
5. 查看 `results/model_benchmark.csv` 与 `results/model_compare_raw_*` 图，选出最合适的模型。

## 工程结构

```
DiodeFitter/
  main.m                   % 主流程：预处理 + 拟合 + 绘图
  preprocess_only.m        % 预处理（两种方法一起跑）
  fit_only.m               % 仅拟合
  nn_fit_test.m            % 3 输入 1 输出神经网络拟合测试
  config_fit.m             % 全局配置入口
  data_processor/          % 数据处理模块
    read_raw_data.m
    prepare_data.m
    preprocess_nn.m        % BP 神经网络平滑/内插
    preprocess_nn_3in1out.m
    get_proc_csv.m
    write_processed_csv.m
  models/                  % 模型定义模块
    load_model.m
    list_models.m
    normalize_model_spec.m
    model_data_poly.m
    model_rohm_idi.m
  solver/                  % 求解器模块
    auto_fit_models.m      % 多模型竞技入口
    run_fitting.m
    cost_functions.m
    compute_weight.m
    flatten_data.m
    predict_by_vgs.m
  utils/                   % 绘图与评估
    plot_preprocess.m
    plot_fit.m
    plot_error.m
    calc_metrics.m
    compare_models_with_raw.m
    data_poly_fit.m
    data_poly_eval.m
    fit_from_table.m
    write_pred_csv.m

data/
  Id_Vds.csv                % 原始数据
  Id_Vds_processed_vgs.csv  % 方法1：按 Vgs 平滑
  Id_Vds_processed_nn3.csv  % 方法2：3输入1输出平滑

results/
  preprocess_compare.png    % 预处理前后对比图
  fit_compare.png           % 原始/平滑/拟合对比图
  error_percent.png         % 误差曲线
  metrics.csv               % 每个 Vgs 的 MAPE/RMSE/最大误差
  fit_params.mat            % 拟合参数与配置快照
  fit_pred.csv              % 拟合预测曲线
```

## 数据要求

- 文件路径：`data/Id_Vds.csv`
- 列名：`Vds`, `Id`, `Vgs`, `Tj`（大小写不敏感）
- 第三象限数据：`Vds < 0`、`Id < 0`
- 程序内部会使用 `Vf = -Vds`、`If = -Id` 进行拟合

## 预处理（BP 神经网络）

配置在 `DiodeFitter/config_fit.m` 的 `cfg.preprocess`：

- `cfg.preprocess.hidden_layers`：隐层规模（支持多层，如 `[10 10]`）
- `cfg.preprocess.hidden_size`：兼容旧字段（优先使用 `hidden_layers`）
- `cfg.preprocess.trainFcn`：训练函数（如 `trainlm`、`trainscg`、`trainbr`）
- `cfg.preprocess.transfer_fcn_hidden`：隐藏层激活函数（如 `tansig`、`logsig`）
- `cfg.preprocess.transfer_fcn_output`：输出层激活函数（如 `purelin`）
- `cfg.preprocess.regularization`：正则化系数
- `cfg.preprocess.epochs`：最大迭代次数
- `cfg.preprocess.gridN`：输出网格点数
- `cfg.preprocess.outlier.enable`：是否剔除离群点
- `cfg.preprocess.show_net`：显示网络结构图
- `cfg.preprocess.show_net_summary`：在控制台输出网络结构
- `cfg.preprocess.show_perf`：显示训练性能曲线
- `cfg.preprocess.show_fit`：显示 BP 拟合曲线
- `cfg.preprocess.show_each_vgs`：每个 Vgs 都显示（否则只显示第一个）

如需指定激活函数，可在 `DiodeFitter/data_processor/preprocess_nn.m` 中设置：
```matlab
net.layers{1}.transferFcn = 'tansig';
net.layers{end}.transferFcn = 'purelin';
```

预处理可生成以下网络可视化图：

- `results/nn_structure.png`：网络结构图
- `results/nn_performance.png`：训练性能曲线
- `results/nn_fit.png`：BP 拟合效果图

## 3 输入 1 输出神经网络拟合（测试）

用途：用 `Vds, Vgs, Tj -> Id` 的神经网络直接拟合数据，便于对比“函数模型 vs 神经网络”的上限效果。

运行：
```matlab
nn_fit_test
```

配置在 `DiodeFitter/config_fit.m` 的 `cfg.nnfit`：

- `cfg.nnfit.use_processed`：是否使用预处理数据
- `cfg.nnfit.hidden_layers`：隐层结构
- `cfg.nnfit.trainFcn`：训练函数
- `cfg.nnfit.transfer_fcn_hidden` / `transfer_fcn_output`：激活函数
- `cfg.nnfit.epochs` / `regularization`：训练参数
- `cfg.nnfit.show_net` / `show_perf` / `show_fit`：显示网络图与拟合图

输出：
- `results/nn_fit_3in1out_structure.png`
- `results/nn_fit_3in1out_performance.png`
- `results/nn_fit_3in1out_compare.png`
- `results/nn_fit_3in1out_error.png`
- `results/nn_fit_model.mat`

## 基于数据的显式拟合函数（非查表）

使用二次多项式拟合 `log(If)`，得到一个显式函数用于对比原 SPICE。

运行拟合：
```matlab
fit = data_poly_fit(config_fit());
```

使用拟合函数：
```matlab
Id_fit = data_poly_eval(Vds, Vgs, fit.coeff, config_fit());
```

输出文件：
- `results/data_poly_fit.mat`：拟合参数与配置
- `results/data_poly_coeffs.csv`：系数表

## 拟合与求解

配置在 `DiodeFitter/config_fit.m`：

- 求解器：`cfg.fit.solver`（`fmincon` / `lsqnonlin` / `lsqcurvefit`）
- 多起点：`cfg.fit.use_multi_start`、`cfg.fit.multi_start.n_start`、`cfg.fit.multi_start.use_model_x0`、`cfg.fit.multi_start.jitter`
- 权重：`cfg.fit.weight.*`（当前为线性权重，强调大电流）
- 代价函数组合：`cfg.fit.cost.*`
  - 说明：`lsqcurvefit` 仅使用 `weighted_mse`，如需 `combo` 等模式请用 `lsqnonlin`/`fmincon`

模型定义在 `DiodeFitter/models/`：

- `model_rohm_idi.m`：模型公式 + 参数初值/上下限（单文件维护）
- `normalize_model_spec.m`：统一模型字段（支持旧模型结构体）
- `load_model.m`：根据 `cfg.model.name` 加载模型
- `list_models.m`：自动发现 `model_*.m`

要替换模型：
1. 在 `DiodeFitter/models/` 新增 `model_xxx.m`，在该文件中同时定义模型公式与初值/上下限  
2. 在 `DiodeFitter/config_fit.m` 中设置 `cfg.model.name = 'model_xxx';`

给其他 AI 生成新模型时，可直接复制模板：
- `MODEL_REQUEST_TEMPLATE.md`

多模型竞技（Model Selection / Benchmarking）：

- 在 `DiodeFitter/config_fit.m` 中设置：
  ```matlab
  cfg.model.benchmark.enable = true;
  cfg.model.benchmark.metric = 'AIC'; % 也可用 'MAPE' / 'RMSE' / 'Max' / 'Resnorm' / 'SSE' / 'BIC'
  cfg.model.benchmark.models = {'model_rohm_idi', 'model_data_poly'}; % 为空则自动扫描 model_*.m
  ```
- 程序会逐个拟合并选择评分最小的模型（评分由 `metric` 决定）
- 会输出 `results/model_benchmark.csv` 作为模型对比表（含 AIC/BIC/参数数目）

两模型与原始数据对比（同图）：

- 在 `DiodeFitter/config_fit.m` 中设置：
```matlab
cfg.model.compare.enable = true;
cfg.model.compare.models = {'model_rohm_idi', 'model_data_poly'};
```
- 说明：即使未开启 `cfg.model.benchmark.enable`，只要 `cfg.model.compare.enable = true` 且 `models` 至少包含两个模型，`main`/`fit_only` 会自动补跑对比并生成图；不需要时关闭 `cfg.model.compare.enable`。
- 输出：  
  - `results/model_compare_raw_Vgs_*.png`
  - `results/model_compare_raw_err_Vgs_*.png`
  - `results/model_compare_raw_metrics.csv`
  - `results/model_compare_raw_summary.csv`

## 输出说明

拟合完成后：

- `results/fit_compare.png`：原始/平滑/拟合对比
- `results/error_percent.png`：每个 Vgs 的百分比误差曲线
- `results/metrics.csv`：每个 Vgs 的 MAPE/RMSE/最大误差
- `results/fit_params.mat`：参数与配置快照
- `results/model_benchmark.csv`：多模型竞技评分表（含 AIC/BIC/参数数目）

预处理输出：

- `data/Id_Vds_processed_vgs.csv`：方法1（按 Vgs 平滑）
- `data/Id_Vds_processed_nn3.csv`：方法2（3输入1输出平滑）
- 每个 Vgs 的数据点数 = `cfg.preprocess.gridN`，总行数 = `gridN * Vgs 数量`

两种方法对比输出（每个 Vgs 单独图）：

- `results/preprocess_compare_Vgs_*.png`：原始 + 两种平滑曲线对比
- `results/preprocess_error_Vgs_*.png`：百分比误差对比曲线
- `results/preprocess_compare_metrics.csv`：每个 Vgs 的 MAPE/RMSE/最大误差
- `results/preprocess_compare_metrics_summary.csv`：全局汇总指标

查看行数与 Vgs 数量的示例：
```matlab
cfg = config_fit();
proc_csv = get_proc_csv(cfg);
T = readtable(proc_csv);
numVgs = numel(unique(T.Vgs));
nRows = size(T, 1);
```

选择使用哪一种预处理数据：

- 在 `DiodeFitter/config_fit.m` 中设置：
  ```matlab
  cfg.preprocess.select = 'vgs'; % 或 'nn3'
  ```

## 基于拟合后数据的查表函数

当你只想用“拟合后数据”生成一个可调用函数，与原 SPICE 模型做对比时，可使用：

```matlab
cfg = config_fit();
Id_fit = fit_from_table(Vds, Vgs, cfg);   % 默认读取当前选择的预处理数据
```

如需指定其他拟合数据文件（例如 `results/fit_pred.csv`）：

```matlab
Id_fit = fit_from_table(Vds, Vgs, cfg, 'results/fit_pred.csv');
```

对比原 SPICE（以 ROHM IDI 为例）：

```matlab
spec = model_rohm_idi(cfg);
Id_spice = spec.model_fun(spec.p0, Vds, Vgs, cfg);
```

## 依赖

- 预处理：需要 Deep Learning Toolbox（Neural Network Toolbox）
- 拟合：需要 Optimization Toolbox（fmincon / lsqnonlin / lsqcurvefit）

## 常见问题

- 预处理报错 “Input data sizes do not match”  
  已在 `preprocess_nn` 中统一为行向量输入，如仍报错，请确认 `Vds/Id` 列无非数值或空行。

- 不想重复预处理  
  只运行拟合流程即可（会直接读取 `data/Id_Vds_processed_vgs.csv` 或 `data/Id_Vds_processed_nn3.csv`，取决于 `cfg.preprocess.select`）。
