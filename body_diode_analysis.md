# SCZ4004DTA_L3 体二极管分析

## 体二极管的实现位置

体二极管在 `chip_SCZ4004DTA_L3` 子电路中以“行为电流源 + 串联电阻”的方式实现，并不是使用内置的 SPICE `D` 二极管元件。

关键位置：

- 参数：`VF`, `Kdi`, `Gdi`, `Cdi`, `NN`, `Rd_D`, `Dibase` 及其温度修正参数。
- 函数：`IDI(VGS,VDS)` 定义了二极管 I-V 表达式（电路里用的是等价的内联表达式）。
- 元件：`GD SI DDI VALUE={...}` 是体二极管电流源（从 `SI` 到 `DDI`）。
- 串联电阻：`RD D DI {RD_T/s_ratio}` 与 `RDD DI DDI {TRd_D_T-RD_T}` 共同形成二极管通路的串联电阻。

## 实现细节

### 行为电流源（体二极管）

体二极管电流的等效形式为：

```text
I_diode = s_ratio * clamp( gate(VDS) * KDi_T *
          (Dibase_T ** ((-VDS + VF_T + GDi_T*tanh(CDi_T*VGS)) / NN_T) - 1),
          0, 1000 )
```

其中：

- `VDS` 对应 `V(DDI,SI)`（二极管通路的漏源电压）。
- `VGS` 对应 `V(GI,SI)`，通过 `GDi_T*tanh(CDi_T*VGS)` 项调制二极管。
- `gate(VDS) = 0.5 + 0.5*tanh(-100*VDS)` 用于只在 `VDS` 为负时导通。
- `clamp(x,0,1000)` 来自 `MIN(MAX(...,0.0),1000.0)`，用于电流限幅以提高收敛性。

同样的表达式在函数中定义为：

```text
IDI(VGS,VDS) = (0.5+0.5*TANH(-100*VDS)) * KDi_T *
              (Dibase_T ** ((-VDS + VF_T + GDi_T*TANH(CDi_T*VGS))/NN_T) - 1)
```

### 串联电阻路径

体二极管电流并非直接接到漏极引脚，路径为：

`SI -> (GD 电流源) -> DDI -> RDD -> DI -> RD -> D`

电阻分配为：

- 沟道通路使用 `RD_T`（`D` 到 `DI`）。
- 二极管通路总串联电阻为 `RD_T + (TRd_D_T - RD_T) = TRd_D_T`。

### 温度依赖

体二极管相关参数通过以下温度修正得到：

```
VF_T, KDi_T, GDi_T, CDi_T, NN_T, TRd_D_T, Dibase_T
```

每个参数都带有 `TANH(0.01*(Tj-25))` 的温度项，影响二极管 I-V 特性、串联电阻以及栅压调制效应。

## 对体二极管行为的含义

- 体二极管是平滑的方向性行为模型，不是内置 `D` 元件。
- I-V 曲线通过 `Dibase_T ** (...)` 形成指数型特性。
- 栅极偏置可通过 `GDi_T*TANH(CDi_T*VGS)` 改变等效压降。
- 电流被 `MIN/MAX` 硬限幅，有助于仿真收敛。
- 反向恢复等动态效应未显式建模，瞬态主要由其他电容模型（`CDS`, `CGD2`, `CGS`）影响。
