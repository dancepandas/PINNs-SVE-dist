# sve-pinns API 完整参考手册

> 版本 v0.1.0 | 所有公开 API 均从 `sve_pinns` 顶层导入

---

## 目录

1. [RiverModel](#1-rivermodel) — 河道水流模型（核心类）
2. [Channel](#2-channel) — 河道几何断面
3. [TrainingConfig](#3-trainingconfig) — 训练配置
4. [Boundary](#4-boundary) — 边界条件
5. [数据 I/O 函数](#5-数据-io-函数)
6. [CLI 命令行](#6-cli-命令行)
7. [异常类型](#7-异常类型)
8. [YAML 配置格式](#8-yaml-配置格式)

---

## 1. RiverModel

`RiverModel` 是 SDK 的核心类。生命周期：**构造 → fit() → predict() → save()/load()**。

### 构造方法

```python
RiverModel(
    channel: Channel,                    # 必填：河道几何
    training: TrainingConfig = None,     # 训练配置，默认 inverse 模式
    duration: float = 200.0,             # 模拟时长 [s]
    g: float = 9.81,                     # 重力加速度 [m/s²]
    base_depth: float = 2.0,             # 基态初始水深 [m]
    base_velocity: float = 1.0,          # 基态初始流速 [m/s]
    perturbation: dict = None,           # 初始高斯扰动
    boundary: dict = None,               # 边界条件
    device: str = "cpu",                 # 计算设备
)
```

#### 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `channel` | `Channel` | 必填 | 河道几何对象，定义断面形状和河长 |
| `training` | `TrainingConfig` | `None`（默认 inverse） | 训练超参数配置。`None` 时使用全默认值的反演模式 |
| `duration` | `float` | `200.0` | 模拟的时间长度 [s]。应覆盖观测数据的时间范围。数值解采用自适应时间步长，对 CPU 开销影响不大 |
| `g` | `float` | `9.81` | 重力加速度 [m/s²]。绝大多数场景使用默认值即可 |
| `base_depth` | `float` | `2.0` | 初始基态水深 [m]。数值解的初始条件——先设全部河道水深为此值，再叠加扰动 |
| `base_velocity` | `float` | `1.0` | 初始基态流速 [m/s]。同 `base_depth`，作为数值解初始条件 |
| `perturbation` | `dict` 或 `None` | `None` | 初始水面扰动。`None` 时使用默认高斯扰动（振幅 0.3 m，中心 L/2，宽度 60 m）。自定义：`{"amplitude": 0.5, "center": 300.0, "sigma": 40.0}`。三个键均为可选 |
| `boundary` | `dict` 或 `None` | `None` | 上下边界条件。格式：`{"upstream": {...}, "downstream": {...}}`。`None` 时两端均为透射边界。支持值详见 [Boundary](#4-boundary) |
| `device` | `str` | `"cpu"` | 计算设备。`"cpu"`（默认）、`"cuda"`（自动选 GPU）、`"cuda:0"`（指定 0 号 GPU）。需 PyTorch GPU 版本 + CUDA |

#### 初始扰动 perturbation 详解

```python
# 默认（不传此参数）
perturbation = {"amplitude": 0.3, "center": channel.length / 2, "sigma": 60.0}

# 无扰动（稳态算例）
perturbation = {"amplitude": 0.0}

# 自定义
perturbation = {"amplitude": 0.5, "center": 200.0, "sigma": 30.0}
```

| 键 | 类型 | 默认值 | 说明 |
|----|------|--------|------|
| `amplitude` | `float` | `0.3` | 高斯扰动幅度 [m]。设为 `0.0` 即为无扰动稳态 |
| `center` | `float` | `L/2` | 扰动峰值位置 [m]，沿河道方向 |
| `sigma` | `float` | `60.0` | 扰动宽度 [m]，高斯标准差。越小扰动越集中 |

### 实例方法

#### `fit(observations=None, mode=None) → self`

训练模型。该操作会：
1. 生成内部数值参考解（MacCormack 格式）
2. 从参考解采样训练点（观测 / 初始条件 / 边界条件 / PDE 配点）
3. 执行 Adam → L-BFGS 两阶段优化

**参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `observations` | `dict` / `str` / `None` | `None` | 观测数据。`None`：从内部合成数据自动采样（含 `noise_assumed` 噪声）。`str`：CSV 或 NPZ 文件路径，调用 `load_observations` 读取。`dict`：包含 `x`、`t`、`h` 键的观测字典，格式同 `load_observations` 返回值 |
| `mode` | `str` 或 `None` | `None` | 覆盖 `TrainingConfig.mode`。`"inverse"`：反演 S₀ 和 n。`"forward"`：S₀ 和 n 固定，仅拟合流场。`None` 时使用 `TrainingConfig.mode` |

**返回值**：`self`（支持链式调用 `model.fit().predict(...)`）

**观测字典格式**（当 `observations` 为 dict 时）：

```python
{
    "x": np.array([50.0, 200.0, ...]),      # 沿河道位置 [m]
    "t": np.array([1.47, 5.00, ...]),         # 时间 [s]
    "h": np.array([2.05, 2.12, ...]),         # 水位 [m]
    "u": np.array([0.0, 1.05, ...]),          # 流速 [m/s]，可选
    "mask_u": np.array([0.0, 1.0, ...]),      # u 掩码，可选。1=有流速，0=缺测
}
```

#### `predict(x, t) → (h, u)`

在给定时空坐标上推理水深 `h` 和流速 `u`。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `x` | `float` / `list` / `np.ndarray` | 沿河道位置 [m]。标量、列表或一维数组 |
| `t` | `float` / `list` / `np.ndarray` | 时间 [s]。标量、列表或一维数组 |

**返回值**：`(h, u)` 两个 numpy 数组。

**配对规则**：
- `x` 和 `t` **等长** → **逐点配对**：`(x[i], t[i])` 逐一计算，返回形状与输入相同
- `x` 和 `t` **不等长** → **网格模式**：`meshgrid(x, t)` 全组合，返回形状 `(len(x), len(t))`

**示例**：

```python
# 单点推理
h, u = model.predict(x=500.0, t=100.0)         # h.shape = (), u.shape = ()

# 逐点配对（3 个点）
h, u = model.predict(x=[100, 500, 800],
                     t=[0, 100, 200])            # h.shape = (3,), u.shape = (3,)

# 网格模式（3×4=12 个点）
h, u = model.predict(x=[100, 500, 800],
                     t=[0, 50, 100, 150])        # h.shape = (3, 4), u.shape = (3, 4)
```

#### `save(path, notes="") → None`

保存模型为 `.pinn` 文件。

**参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `path` | `str` | 必填 | 输出文件路径，建议以 `.pinn` 结尾 |
| `notes` | `str` | `""` | 备注信息，写入文件元数据 |

**前提条件**：已调用 `fit()` 训练完毕；有效的 `license.key` 存在。

#### 类方法 `RiverModel.load(path) → RiverModel`

从 `.pinn` 文件加载模型。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `path` | `str` | `.pinn` 文件路径 |

**返回值**：训练好的 `RiverModel` 实例，可直接 `predict()`。

**前提条件**：有效的 `license.key` 存在。文件签名校验通过（篡改过会拒绝）。

### 实例属性

#### `params → dict`

训练后返回反演结果。**仅 `fit()` 后可访问**。

```python
model.fit()
print(model.params)
# → {"S0": 0.00103, "n": 0.0327}
```

| 键 | 类型 | 说明 |
|----|------|------|
| `S0` | `float` | 底坡 |
| `n` | `float` | Manning 糙率 [s·m⁻¹/³] |

#### `advanced → _Advanced`

高级访问命名空间（非稳定 API，可能随版本变更）。

| 属性 | 类型 | 说明 |
|------|------|------|
| `model.advanced.net` | `nn.Module` | 原始 PyTorch 网络 |
| `model.advanced.geom` | `Geometry` | 河道几何对象 |
| `model.advanced.history` | `list[dict]` | 完整训练历史（每步 loss、S₀、n） |
| `model.advanced.reference` | `dict` | 内部数值参考解 |

---

## 2. Channel

`Channel` 定义河道几何断面。提供三种标准断面和 YAML 加载。

### 构造方法

```python
Channel(
    geometry_kind: str,         # "rectangular" | "trapezoidal" | "powerlaw"
    length: float,              # 河长 [m]
    geometry_kwargs: dict,      # 几何参数（可选）
    lateral_inflow: float = 0.0 # 侧向入流 [m²/s]
)
```

#### 参数详解

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `geometry_kind` | `str` | 必填 | 断面类型。`"rectangular"`：矩形。`"trapezoidal"`：梯形。`"powerlaw"`：幂律 |
| `length` | `float` | 必填 | 河道长度 [m] |
| `geometry_kwargs` | `dict` | `None` | 断面参数，随 `geometry_kind` 而异 |
| `lateral_inflow` | `float` | `0.0` | 侧向入流 [m²/s]。模拟降雨、地下水补给等。正值=入流，负值=出流 |

### 类方法（便捷构造器）

#### `Channel.rectangular(bottom_width=1.0, length=1000.0, lateral_inflow=0.0) → Channel`

矩形断面。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `bottom_width` | `float` | `1.0` | 底宽 [m] |
| `length` | `float` | `1000.0` | 河长 [m] |
| `lateral_inflow` | `float` | `0.0` | 侧向入流 [m²/s] |

**几何公式**：`A = b·h`，`P = b + 2h`，`R = A/P`

#### `Channel.trapezoidal(bottom_width=5.0, side_slope=1.5, length=1000.0, lateral_inflow=0.0) → Channel`

梯形断面。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `bottom_width` | `float` | `5.0` | 底宽 [m] |
| `side_slope` | `float` | `1.5` | 边坡系数（水平/垂直）。`m=1.5` 表示每上升 1 m，单侧拓宽 1.5 m |
| `length` | `float` | `1000.0` | 河长 [m] |
| `lateral_inflow` | `float` | `0.0` | 侧向入流 [m²/s] |

**几何公式**：`A = (b + m·h)·h`，`P = b + 2h·√(1+m²)`，`R = A/P`，`I₁ = b·h²/2 + m·h³/3`

#### `Channel.powerlaw(k=1.0, p=1.0, length=1000.0, lateral_inflow=0.0) → Channel`

幂律断面（自然河道近似）。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `k` | `float` | `1.0` | 系数。`A = k·h^p`。`k=1, p=1` 退化为 `b=1` 矩形 |
| `p` | `float` | `1.0` | 指数。`p=1` 退化为矩形，`p>1` 表示上宽下窄（天然河道特征） |
| `length` | `float` | `1000.0` | 河长 [m] |
| `lateral_inflow` | `float` | `0.0` | 侧向入流 [m²/s] |

**几何公式**：`A = k·h^p`，`P ≈ k·h^(p-1)`（宽浅近似），`R = A/P`

#### `Channel.from_yaml(path: str) → Channel`

从 YAML 配置文件读取河道参数。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `path` | `str` | YAML 文件路径。读取 `channel` 段的全部字段 |

### 实例属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `geometry_kind` | `str` | 断面类型字符串 |
| `length` | `float` | 河长 [m] |
| `geometry_kwargs` | `dict` | 几何参数字典 |
| `lateral_inflow` | `float` | 侧向入流 [m²/s] |
| `geom` | `Geometry` | 内部几何对象（非公开 API） |

---

## 3. TrainingConfig

`TrainingConfig` 是 dataclass，定义所有训练超参数。

### 构造方法

```python
TrainingConfig(
    mode: str = "inverse",
    adam_iters: int = 8000,
    lbfgs_iters: int = 300,
    learning_rate: float = 1e-3,
    collocation_points: int = 5000,
    obs_points: int = 200,
    ic_points: int = 200,
    bc_points: int = 200,
    noise_assumed: float = 0.0,
    seed: int = 0,
    true_S0: float = 0.001,
    true_n: float = 0.03,
    init_S0: float = 0.002,
    init_n: float = 0.05,
    prior_enabled: bool = False,
    prior_weight: float = 1.0,
    prior_S0_mean: float = 0.001,
    prior_S0_std: float = 0.0005,
    prior_n_mean: float = 0.03,
    prior_n_std: float = 0.015,
    verbose: bool = True,
)
```

### 参数详解

#### 工作模式

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `mode` | `str` | `"inverse"` | `"inverse"`：反演模式——从观测数据学习 S₀ 和 n。`"forward"`：正向模式——S₀ 和 n 固定，仅拟合流场 |

#### 优化器

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `adam_iters` | `int` | `8000` | Adam 优化器迭代步数。越大收敛越好，时间线性增加。CPU 上每 1000 步约 10-15 秒 |
| `lbfgs_iters` | `int` | `300` | L-BFGS 精细收敛步数。在 Adam 之后执行，对最终精度提升显著 |
| `learning_rate` | `float` | `1e-3` | Adam 学习率。一般不需修改。遇到发散可尝试 `5e-4` |

#### 训练采样

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `collocation_points` | `int` | `5000` | PDE 残差配点数。散布在时空域中，不依赖观测。越多物理约束越强，但训练时间增加。推荐 2000-10000 |
| `obs_points` | `int` | `200` | 从合成数据采样的观测点数。**仅在 `fit()` 不传 `observations` 时生效**。传入了真实观测则忽略此值 |
| `ic_points` | `int` | `200` | 初始条件采样点数（t=0 时刻） |
| `bc_points` | `int` | `200` | 边界条件采样点数（x=0 和 x=L 处） |
| `noise_assumed` | `float` | `0.0` | 合成观测的噪声水平。`0.01` 表示 1% 高斯噪声（相对水位均值）。真实观测场景应设为 `0.0` |

#### 物理参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `true_S0` | `float` | `0.001` | 底坡真值。正向模式固定此值；反演模式仅作对照，不参与训练 |
| `true_n` | `float` | `0.03` | Manning 糙率真值。同上 |
| `init_S0` | `float` | `0.002` | 反演模式 S₀ 的初始猜测。偏离真值以验证恢复能力 |
| `init_n` | `float` | `0.05` | 反演模式 n 的初始猜测。建议设到真值的 1.5-2 倍 |

#### MAP 先验（贝叶斯正则化）

在数据少、噪声大的场景下，MAP 先验将参数拉向先验均值，防止发散或漂移。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `prior_enabled` | `bool` | `False` | 先验开关。`True` 时启用 MAP 正则化 |
| `prior_weight` | `float` | `1.0` | 先验权重。越大拉向均值的力越强。推荐 `0.1~1.0` |
| `prior_S0_mean` | `float` | `0.001` | S₀ 的先验均值。基于文献或经验设定 |
| `prior_S0_std` | `float` | `0.0005` | S₀ 的先验标准差。越小对均值附近的约束越紧 |
| `prior_n_mean` | `float` | `0.03` | n 的先验均值 |
| `prior_n_std` | `float` | `0.015` | n 的先验标准差 |

#### 其他

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `seed` | `int` | `0` | 随机种子。固定后训练完全可复现 |
| `verbose` | `bool` | `True` | 是否打印训练日志（每 `log_every` 步输出 loss、S₀、n） |

### 类方法

#### `TrainingConfig.default() → TrainingConfig`

返回全默认值的 `TrainingConfig`（等价于 `TrainingConfig()`）。

#### `TrainingConfig.from_yaml(path: str) → TrainingConfig`

从 YAML 文件读取训练配置。

---

## 4. Boundary

`Boundary` 提供边界条件的便捷构造。所有方法返回 dict，可直接传给 `RiverModel(boundary={...})` 或 YAML 的 `boundary` 段。

### 类方法

#### `Boundary.transmissive() → dict`

透射（零梯度）边界。边界单元取相邻内单元的值。

```python
model = RiverModel(
    ...,
    boundary={"upstream": Boundary.transmissive(),
              "downstream": Boundary.transmissive()}
)
```

#### `Boundary.inflow(Q=None, Q_series=None, h=None, h_series=None) → dict`

定流量 / 变流量上游边界。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Q` | `float` | `None` | 恒定流量 [m³/s]。与 `Q_series` 二选一 |
| `Q_series` | `list` | `None` | 时变流量序列。格式：`[[t0,Q0],[t1,Q1],...]`，线性插值 |
| `h` | `float` | `None` | 固定上游水深 [m]。可选，不给则从内场外推 |
| `h_series` | `list` | `None` | 时变水深序列。格式同 `Q_series` |

```python
# 恒定流量
up = Boundary.inflow(Q=5.0, h=2.0)

# 时变流量
up = Boundary.inflow(
    Q_series=[[0, 5.0], [100, 8.0], [200, 5.0]],
    h=2.0
)
```

#### `Boundary.stage(h=None, h_series=None) → dict`

定水位下游边界。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `h` | `float` | `None` | 恒定水位 [m]。与 `h_series` 二选一 |
| `h_series` | `list` | `None` | 时变水位序列 |

```python
down = Boundary.stage(h=2.5)
```

#### `Boundary.rating(curve) → dict`

水位-流量关系下游边界。

| 参数 | 类型 | 说明 |
|------|------|------|
| `curve` | `list` | `[[h0,Q0],[h1,Q1],...]` 控制点，按 h 排序。线性插值得到 Q(h) |

```python
down = Boundary.rating(
    curve=[[1.0, 2.0], [1.5, 5.0], [2.0, 8.0], [2.5, 12.0]]
)
```

---

## 5. 数据 I/O 函数

### `load_observations(path: str) → dict`

读取观测数据文件。

| 参数 | 类型 | 说明 |
|------|------|------|
| `path` | `str` | CSV 或 NPZ 文件路径 |

**返回值**：dict `{x, t, h, u, mask_u}`，可直接传给 `model.fit(observations=...)`。

**CSV 格式要求**：

| 列 | 必填 | 说明 |
|----|------|------|
| `x` | 是 | 沿河道位置 [m] |
| `t` | 是 | 时间 [s] |
| `h` | 是 | 水位 [m] |
| `u` | 否 | 流速 [m/s]；缺填 `0`，同时 `mask_u` 填 `0` |
| `mask_u` | 否 | 标记流速是否可用（1=有，0=无）。默认全 1 |

**NPZ 格式要求**：键名同 CSV 列名，每个值为一维 numpy 数组。

### `save_model(model: RiverModel, path: str) → None`

函数式保存（等价于 `model.save(path)`）。

### `load_model(path: str) → RiverModel`

函数式加载（等价于 `RiverModel.load(path)`）。

### `model_from_yaml(path: str) → RiverModel`

从 YAML 配置文件构建未训练的 `RiverModel`。读取 `channel`、`physics`、`boundary`、`training` 全部段落。

```python
model = model_from_yaml("config.yaml")
model.fit()  # 还需手动调 fit
```

---

## 6. CLI 命令行

SDK 随附 `sve-pinns` 命令行工具。

### `sve-pinns validate <config.yaml>`

校验 YAML 配置文件。不进行训练，仅检查字段完整性。

### `sve-pinns train <config.yaml> [-o model.pinn]`

从 YAML 配置训练，输出 `.pinn` 模型文件。

| 参数 | 说明 |
|------|------|
| `config.yaml` | 配置文件路径（必填） |
| `-o` / `--output` | 输出 .pinn 路径。默认 `model.pinn` |

### `sve-pinns predict <model.pinn> --at "x,t [x,t ...]"`

从已训练的模型推理。

| 参数 | 说明 |
|------|------|
| `model.pinn` | .pinn 文件路径 |
| `--at` | 推理点，格式 `"x,t"`，多组用空格分隔。例：`"500,100 200,50"` |

### `sve-pinns info <model.pinn>`

显示模型元信息：版本号、创建时间、几何类型、训练模式、参数值。

### `sve-pinns --help`

列出所有子命令。

---

## 7. 异常类型

所有异常继承 `sve_pinns.SVEError`，可用 `except SVEError` 统一捕获。

| 异常 | 继承 | 触发场景 |
|------|------|---------|
| `SVEError` | `Exception` | 所有异常的根类 |
| `ConfigError` | `SVEError` | YAML 解析失败 / schema 校验失败 / 字段缺失或类型错误 |
| `DataError` | `SVEError` | 观测文件不存在 / CSV 缺列 / 数据格式错误 |
| `ModelIOError` | `SVEError` | .pinn 文件损坏 / 版本不兼容 / 签名校验失败 / license 过期或无效 |
| `TrainingError` | `SVEError` | 训练过程数值发散 (NaN) 或不可恢复错误 |

---

## 8. YAML 配置格式

所有 YAML 顶层字段一览：

```yaml
schema_version: "1.1"    # 必填，当前版本 "1.1"

channel:                  # 河道几何（必填）
  geometry: rectangular   # rectangular | trapezoidal | powerlaw
  bottom_width: 10.0      # 底宽 [m]，rect/trap
  side_slope: 1.5         # 边坡，trapezoidal
  k: 3.0                  # 幂律系数，powerlaw
  p: 1.3                  # 幂律指数，powerlaw
  length: 1000.0          # 河长 [m]
  lateral_inflow: 0.0     # 侧向入流 [m²/s]

physics:                  # 物理参数（可选，全有默认值）
  duration: 200.0         # 模拟时长 [s]
  g: 9.81                 # 重力加速度
  initial:                # 初始条件
    base_depth: 2.0       # 基态水深 [m]
    base_velocity: 1.0    # 基态流速 [m/s]
    perturbation:         # 初始扰动（可选）
      amplitude: 0.3      # 幅度 [m]
      center: 500.0       # 中心 [m]
      sigma: 60.0         # 宽度 [m]

boundary:                 # 边界条件（可选，默认透射）
  upstream:
    type: inflow          # transmissive | inflow | stage | rating
    Q: 5.0                # 流量 [m³/s]
    h: 2.0                # 水位 [m]（可选）
  downstream:
    type: stage
    h: 2.5

data:                     # 观测数据（可选）
  observations: data.csv  # 相对此 yaml 的路径；不填则自动合成
  noise_assumed: 0.0      # 合成数据噪声；外部数据填 0

device: cpu               # 计算设备（可选，默认 "cpu"）

training:                 # 训练配置（可选）
  mode: inverse           # forward | inverse
  true_params: {S0: 0.001, n: 0.03}
  initial_guess: {S0: 0.002, n: 0.05}
  prior:
    enabled: true
    weight: 1.0
    S0: {mean: 0.001, std: 0.0005}
    n:  {mean: 0.03,  std: 0.015}
  optimizer:
    adam_iters: 8000
    lbfgs_iters: 300
    learning_rate: 1.0e-3
  collocation_points: 5000
  obs_points: 200
  ic_points: 200
  bc_points: 200
  seed: 0
```
