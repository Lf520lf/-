# 多模型对比-模型生成模板（可直接复制给其他 AI）

请根据以下要求生成一个新的二极管体二极管模型文件，用于本工程多模型竞技对比。

## 1. 工程信息
- 工程路径：`d:\VScode\功率模块_体二极管_20260118`
- 模型目录：`DiodeFitter/models/`
- 现有模型示例：`model_rohm_idi.m`、`model_data_poly.m`

## 2. 数据与约定
- 数据列：`Vds`, `Id`, `Vgs`, `Tj`
- 第三象限数据：`Vds < 0`、`Id < 0`
- 程序内部使用：`Vf = -Vds`、`If = -Id`
- 目标是拟合体二极管 `Id(Vds, Vgs)`（显式 IofV 形式）

## 3. 需要你输出的内容（必须）
请输出一个 **完整可用的 MATLAB 文件**，命名为：
```
DiodeFitter/models/model_{{模型名}}.m
```
文件必须返回以下结构体字段（字段名不可改）：
- `spec.name`：模型名称（字符串）
- `spec.description`：模型说明（字符串）
- `spec.mode`：固定为 `'IofV'`
- `spec.param_names`：参数名（cell 数组）
- `spec.p0`：参数初值（1xN）
- `spec.lb` / `spec.ub`：参数上下限（1xN）
- `spec.equation`：函数句柄 `@(p, vds, vgs, cfg)`，返回 **正值电流 If**
- `spec.model_fun`：建议等于 `spec.equation`
- `spec.x0_generator`：函数句柄，返回 1xN 初值（可随机扰动）
- （可选）`spec.skip_fit`：如无需优化可设为 `true`

函数必须支持向量化输入，并保证 `If >= 0`。

## 4. 公式要求
- 公式应尽量稳定，避免指数溢出（可用 `min(expo, cfg.model.expo_max)`）
- 若需要 `Vf`，请使用 `vf = -vds`
- 允许使用 `tanh`、`log`、`exp` 等基础函数
- 不要依赖额外工具箱

## 5. 物理约束建议（可选）
- `If` 对 `Vf` 单调不减
- `If >= 0`
- 对 `Vgs` 的影响保持合理增益（不要爆炸）

## 6. 你需要额外提供的信息（可选但推荐）
1. 参数的物理含义（简短说明）
2. 初值与边界的来源依据（经验/文献/估计）

## 7. 输出示例（格式参考）
请输出完整文件内容，不要输出其他说明。

```matlab
function spec = model_example(cfg)
% 这里是模型说明
if nargin < 1 || isempty(cfg)
    cfg = config_fit();
end

spec.name = 'EXAMPLE';
spec.description = '示例模型说明';
spec.mode = 'IofV';
spec.param_names = {'a','b','c'};
spec.p0 = [1, 2, 3];
spec.lb = [0, 0, 0];
spec.ub = [10, 10, 10];
spec.equation = @example_fun;
spec.model_fun = spec.equation;
spec.x0_generator = @() spec.p0;
end

function If = example_fun(p, vds, vgs, cfg)
vf = -vds;
If = p(1) * exp(p(2) * vf) + p(3) * max(vgs, 0);
If = max(If, 0);
end
```

## 8. 我方接入方式（无需你修改）
我会在配置里加入你的模型：
```matlab
cfg.model.benchmark.models = {'model_rohm_idi', 'model_data_poly', 'model_{{模型名}}'};
```

请严格遵循上述格式。
