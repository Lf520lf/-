# 第 1 周｜从第一性原理理解功率 SiC MOSFET 与数据手册

目标（本周末可自测）：
- 能用材料常数与基本方程解释“为何 SiC 适合高压高速”
- 画出并解释 SiC MOSFET 等效电路与三大非线性电容的来源
- 读懂数据手册的关键字段及其物理含义：Ciss/Coss/Crss、Qg/Qgd、Eon/Eoff、Zth(t)
- 能用基本方程推导出 Miller 平台、dv/dt、di/dt 与寄生电感的影响关系

—

## 1. 从材料到器件的“尺度律”：泊松方程 + 漂移-扩散

核心方程
- 电场/电势：一维泊松方程 $ \frac{d^2\psi}{dx^2} = -\frac{\rho(x)}{\varepsilon_s} $
- 载流子输运（稳态）：$ \mathbf{J}_n = q n \mu_n \mathbf{E} + q D_n \nabla n $，低场近似 $v \approx \mu E$

高压单极性功率器件（如 MOSFET）的关键限制在漂移区（n 型均匀掺杂）的电场与电阻权衡：
- 击穿条件（近似）：当最大电场 $E_{\max}$ 达到材料临界电场 $E_{\text{crit}}$ 时雪崩发生
- 为给定耐压 BV，最优设计令漂移区厚度 $t_d \approx \frac{\text{BV}}{E_{\text{crit}}}$，掺杂 $N_d$ 与电场分布满足泊松方程与边界条件
- 漂移区电阻（比电阻乘厚度）：$ R_{\text{on,sp}} \approx \rho \, t_d = \frac{1}{q \mu_n N_d} \cdot \frac{\text{BV}}{E_{\text{crit}}} $

结合一维优化结果（使在 BV 下恰好达到 $E_{\text{crit}}$）可得到著名的 Baliga 尺度律：
- Baliga 优值因子（BFOM）：$ \text{BFOM} \propto \varepsilon_s \mu_n E_{\text{crit}}^3 $
- 最优比导通电阻随耐压的下界（量纲正确的常数略去）：
  $$ R_{\text{on,sp}}^{\min} \propto \frac{\text{BV}^2}{\varepsilon_s \mu_n E_{\text{crit}}^3} $$

物理结论（为何 SiC 更强）：
- SiC 的 $E_{\text{crit}}$ 约为 Si 的 8–10 倍量级，使 $R_{\text{on,sp}}$ 随 BV 的增长远缓于 Si（可小一个到数个数量级）
- SiC 热导率高（数百 W/m·K），利于更高功率密度与结温管理
- 电子速度饱和更高、介电强度更大，支撑更高 dv/dt 与更薄漂移区

—

## 2. MOS 电容与阈值：从电荷守恒与边界条件出发

基本结构：金属/氧化层/半导体（MOS）。从高斯定律与电荷守恒出发：
- 栅氧电容密度 $C_{\text{ox}} = \frac{\varepsilon_{\text{ox}}}{t_{\text{ox}}}$
- 栅电压分配：$ V_G = \phi_{ms} + V_{\text{ox}} + \psi_s $
  - $\phi_{ms}$：金属-半导体功函差
  - $V_{\text{ox}} = \frac{Q_{\text{ox}} + Q_{\text{it}} + Q_s}{C_{\text{ox}}}$（固定氧化层电荷、界面态电荷、半导体表面电荷）
  - $\psi_s$：表面势

阈值电压近似（p 型衬底 NMOS）：
$$ V_T \approx \phi_{ms} - \frac{Q_{\text{ox}} + Q_{\text{it}}}{C_{\text{ox}}} + 2\phi_F + \frac{\sqrt{4 \varepsilon_s q N_A \phi_F}}{C_{\text{ox}}} $$
- $\phi_F = \frac{kT}{q}\ln\!\frac{N_A}{n_i}$，受温度与掺杂影响
- SiC 的氧化界面态密度 Dit 往往较高，$Q_{\text{it}}$ 会抬升或漂移 $V_T$，并降低沟道迁移率（对直流导通和动态都有影响）

沟道导通（强反型）的一阶导通模型（近似）：
- 低场线性区：$ I_D \approx \mu_{\text{eff}} C_{\text{ox}} \frac{W}{L}\left[(V_{GS}-V_T)V_{DS} - \frac{V_{DS}^2}{2}\right] $
- 饱和区（速度饱和/短沟等非理想忽略时）：$ I_{D,\text{sat}} \approx \frac{1}{2}\mu_{\text{eff}} C_{\text{ox}} \frac{W}{L}(V_{GS}-V_T)^2 $

—

## 3. 非线性电容的来源与电荷分配

功率 MOSFET 的三大端口电容来自不同物理区域：
- $C_{gs}$：栅-源之间，主要由氧化层 + 沟道区电荷调制
- $C_{gd}$（Miller）：栅-漏重叠 + 漂移区耦合，是开关过程中 dv/dt 的关键控制项
- $C_{ds}$：漏-源漂移区耗尽层电容，随 $V_{DS}$ 反偏显著非线性

数据手册聚合电容与端口电容的关系：
- $C_{\text{iss}} = C_{gs} + C_{gd}$
- $C_{\text{oss}} = C_{ds} + C_{gd}$
- $C_{\text{rss}} = C_{gd}$

电荷守恒建模（数值更稳健）：
- 用电荷函数 $Q(V)$ 建模，电容为 $C(V) = \frac{dQ}{dV}$
- 开关时的栅电流：$ i_g = \frac{d}{dt}(Q_{gs} + Q_{gd}) $，Miller 平台阶段主要消耗 $Q_{gd}$

Ward–Dutton 电荷分配思想（要点）：将沟道电荷在源/漏两端以连续可导的权重分配，以保持电荷守恒与数值稳定（对紧凑模型与电路仿真重要）。

—

## 4. 开关瞬态与 Miller 平台：回路方程直达关键量

栅回路基本式（忽略寄生耦合的一阶近似）：
- $ i_g \approx \frac{V_{\text{drv}} - V_G}{R_g} $
- 平台阶段，$V_{GS}$ 近似“卡在”$V_{\text{plateau}}$，栅电流主要用于充放 $C_{gd}$，而 $V_{DS}$ 快速变化
- 近似关系：$ \frac{dV_{DS}}{dt} \approx \frac{i_g}{C_{gd}(V)} $（注意 $C_{gd}$ 强非线性，且与 $V_{DS}$、偏置相关）

电流上升率（上管开通）：电感负载近似
- $ \frac{di}{dt} \approx \frac{V_{DC} - V_{DS}}{L_{\text{loop}}} $
- 与 $R_g$、$C_{gd}$、$L_{\text{loop}}$、器件迟滞共同决定过冲与振铃

能量损耗定义（数据手册 Eon/Eoff）：
- $ E_{\text{on/off}} = \int v_{DS}(t) \, i_D(t) \, dt $（在规定电流、直流母线电压、栅电阻与温度等条件下测得）

—

## 5. 寄生参数与电磁一致性：公共源电感的首要性

寄生电感电压：$ v_L = L \, \frac{di}{dt} $
- 公共源电感 $L_{s,\text{cs}}$ 让源极“被抬升”，等效有效栅压下降：$ V_{GS,\text{eff}} = V_G - (L_{s,\text{cs}} \, \frac{di}{dt}) $
  - 结果：开通变慢（负反馈）、波形平台延长，甚至影响下管的误导通
- Kelvin 源（独立的源感测端）可显著减轻该负反馈，提高开关一致性

回路设计与 EMI：
- 最小化电流环路面积（漏极-功率回路），并联缓冲/RC–RCD 网络可调控过冲与振铃
- 分立 $R_{g,\text{on}}/R_{g,\text{off}}$、Miller 夹钳可分别控制上升/下降沿与误导通

—

## 6. 数据手册字段的物理意义与测量方式

- 绝对额定值（例如 $V_{DS,\max}$、$V_{GS}$ 范围、$T_j$ 上限）：由材料 $E_{\text{crit}}$、栅氧可靠性（TDDB）与热限制决定
- 静态参数：$R_{DS(on)}(T)$、$V_T(T)$、$I_D$–$V_{DS}$、$I_D$–$V_{GS}$，反映沟道迁移率、氧界面态、漂移区电阻
- 动态参数
  - 电容曲线：$C_{\text{iss}}$/$C_{\text{oss}}$/$C_{\text{rss}}$ vs $V_{DS}$，对应三端子电容的聚合
  - 栅电荷：$Q_g$/$Q_{gd}$ 曲线，由积分栅电流得到；Miller 平台区 $Q_{gd}$ 占主导
  - 能量损耗：$E_{\text{on}}/E_{\text{off}}$ 随 $R_g$、$I_D$、$V_{DC}$、$T$ 的表格或曲线（需留意测试二极管、治具寄生）
- 热学参数
  - 瞬态热阻 $Z_{\text{th}}(t)$：描述 $P(t)$ 到结温上升 $\Delta T_j(t)$ 的卷积核
  - $R_{\theta jc}$、$R_{\theta ja}$：稳态热阻；用 Foster/Cauer 网络近似 $Z_{\text{th}}(t)$

—

## 7. 将“第一性原理”映射为“读图与建模”的方法论

- 从尺度律推断趋势：高 BV 下，SiC 的 $R_{DS(on)}$ 随温度与 BV 的变化应显著优于 Si
- 用 $Q(V)$ 而非常数电容：保证充放电与能量的一致性，复现 Miller 平台更稳健
- 对齐条件是王道：任何与数据手册对比的仿真，必须严格匹配 $R_g$、$I_D$、$V_{DC}$、温度与治具寄生
- 优先识别“控制旋钮”：$R_g$、$L_{s,\text{cs}}$、$C_{gd}$ 在瞬态中的一阶影响最大

—

## 8. 本周练习与产出清单

练习
1) 用 $E_{\text{crit}}$、$\varepsilon_s$、$\mu_n$ 推导 $R_{\text{on,sp}}^{\min} \propto \frac{\text{BV}^2}{\varepsilon_s \mu_n E_{\text{crit}}^3}$ 的步骤（写出假设与边界条件）
2) 从 MOS 电荷守恒推导阈值电压近似式，并讨论 $Q_{\text{ox}}$、$Q_{\text{it}}$ 对 $V_T$ 的影响方向与数量级
3) 画出功率 MOSFET 等效电路，分别标注 $C_{gs}$/$C_{gd}$/$C_{ds}$ 的物理来源
4) 推导 Miller 平台近似关系：$dV_{DS}/dt \approx i_g / C_{gd}$；并说明为何 $C_{gd}(V)$ 的非线性会让平台时长随 $V_{DS}$ 变化
5) 用回路法写出公共源电感导致的有效栅压下降式，并定性讨论对 di/dt 与误导通的影响

产出
- 一页“材料→器件→数据手册字段”的映射思维导图
- 一页公式卡片：泊松方程、BFOM、$V_T$ 近似、$Q$–$C$ 关系、$E_{\text{on/off}}$ 定义、$Z_{\text{th}}$ 卷积
- 选定 1–2 颗 SiC MOSFET 数据手册，圈出上述字段的来源页并记录测试条件

—

## 9. 术语速查（本周用得到）
- BFOM：$\varepsilon_s \mu_n E_{\text{crit}}^3$；单极性高压器件优劣的材料指标
- $C_{\text{iss}}/C_{\text{oss}}/C_{\text{rss}}$：聚合电容，对应端口电容 $C_{gs}/C_{gd}/C_{ds}$
- $Q_g/Q_{gd}$：总栅电荷/米勒电荷；平台阶段主要消耗 $Q_{gd}$
- $Z_{\text{th}}(t)$：瞬态热阻；$\Delta T_j(t) = (P * Z_{\text{th}})(t)$
- Kelvin 源：独立源感测端，削弱公共源电感负反馈

—

参考阅读（按优先级）
- Baliga — Fundamentals of Power Semiconductor Devices（BFOM 与高压漂移区设计）
- Kimoto & Cooper — Fundamentals of Silicon Carbide Technology（SiC 材料与界面特性）
- 任一主流厂商 SiC MOSFET 应用手册（Wolfspeed/Infineon/ROHM/onsemi/ST），聚焦电容、电荷、双脉冲、热模型章节