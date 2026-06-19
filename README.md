# sve-pinns

> **Physics-Informed Neural Networks for 1D Saint-Venant Equations**
>
> 河道水流建模 · 糙率参数反演 · 水位流速预测

`sve-pinns` 是一个用物理信息神经网络（PINN）求解一维完整圣维南方程的商业 SDK。你提供河道几何、观测数据、边界条件，它帮你反演底坡 S₀ 和 Manning 糙率 n，并预测全时空域的 h(x,t) 和 u(x,t)。

---

## 目录

1. [安装](#1-安装)
2. [5 分钟上手](#2-5-分钟上手)
3. [核心概念](#3-核心概念)
4. [Python API 参考](#4-python-api-参考)
5. [YAML 配置文件](#5-yaml-配置文件)
6. [观测数据格式](#6-观测数据格式)
7. [模型保存与加载](#7-模型保存与加载)
8. [CLI 命令行](#8-cli-命令行)
9. [性能基准](#9-性能基准)
10. [许可](#10-许可)

---

## 1. 安装

### 系统要求

- Python 3.10+
- 4 GB RAM（训练依赖 CPU，无需 GPU）
- Windows / Linux / macOS

### 安装步骤

```bash
# 下载对应平台的 wheel（以 Windows 为例）
pip install https://github.com/dancepandas/PINNs-SVE-dist/releases/latest/download/sve_pinns-0.1.0-cp310-cp310-win_amd64.whl
```

| 平台 | wheel 文件名关键字 |
|------|-------------------|
| Windows x64 | `cp310-cp310-win_amd64` |
| Linux x64 | `cp310-cp310-linux_x86_64` |
| macOS (Intel + M 系列) | `cp310-cp310-macosx_10_9_universal2` |

### 安装后验证

```bash
python -c "from sve_pinns import RiverModel, Channel; print('OK')"
```

### 安装后必须做的事

**将 license.key 放到以下任一位置**（否则无法保存/加载模型）：
- 工作目录下的 `license.key`
- `~/.sve-pinns/license.key`（Linux/macOS）
- `%APPDATA%\sve-pinns\license.key`（Windows）

联系 `chs9710@163.com` 获取授权文件。

---

## 2. 5 分钟上手

### 场景：已知梯形河道，有几组测站的水位数据，想反推糙率 n

```python
from sve_pinns import RiverModel, Channel, TrainingConfig, load_observations
import numpy as np

# ============== 第一步：定义河道 ==============
# 梯形断面：底宽 5m、边坡 1:1.5、河长 1000m
channel = Channel.trapezoidal(
    bottom_width=5.0,
    side_slope=1.5,
    length=1000.0
)

# ============== 第二步：加载观测数据 ==============
# CSV 必须有 x, t, h 三列；u 和 mask_u 可选
obs = load_observations("station_data.csv")

# ============== 第三步：配置训练 ==============
# 反演模式（inverse）：从观测数据中学习 S0 和 n
# 开 MAP 先验：数据少、噪声大时更稳
training = TrainingConfig(
    mode="inverse",           # "inverse"=反演参数 / "forward"=仅拟合流场
    adam_iters=8000,          # Adam 迭代步数
    lbfgs_iters=300,          # L-BFGS 精细收敛
    prior_enabled=True,       # 开 MAP 先验（噪声鲁棒）
    prior_S0_mean=0.001,
    prior_S0_std=0.0005,
    prior_n_mean=0.03,
    prior_n_std=0.015,
    verbose=True,             # 打印训练进度
)

# ============== 第四步：训练 ==============
model = RiverModel(
    channel=channel,
    training=training,
    duration=200.0,            # 模拟时长 [s]
    g=9.81,                    # 重力加速度
    base_depth=2.0,            # 初始基态水深 [m]
    base_velocity=1.0,         # 初始基态流速 [m/s]
)
model.fit(observations=obs)    # None=用合成数据

# ============== 第五步：查看结果 ==============
print(model.params)
# → {'S0': 0.00103, 'n': 0.0327}   # 真值 S0=0.001, n=0.03

# ============== 第六步：推理 ==============
# 逐点配对模式：x 和 t 长度相同
h, u = model.predict(x=[100, 500, 800], t=[0, 100, 150])
print(h.shape)  # (3,)

# 网格模式：x 和 t 长度不同 → meshgrid
h, u = model.predict(x=[100, 500, 800], t=[0, 50, 100, 150])
print(h.shape)  # (3, 4)

# ============== 第七步：保存 ==============
model.save("my_river.pinn", notes="第一个模型")

# ============== 第八步：下次加载 ==============
model = RiverModel.load("my_river.pinn")
print(model.params)  # 与保存前完全一致
```

---

## 3. 核心概念

### 控制方程

模型求解一维完整圣维南方程（Saint-Venant Equations），含底坡和 Manning 摩阻：

```
连续性:  ∂h/∂t + ∂(hu)/∂x = q_L       （h=水深, u=流速, q_L=侧向入流）
动量:    ∂u/∂t + u·∂u/∂x + g·∂h/∂x = g·(S₀ − Sf)
摩阻:    Sf = n²·u|u| / R^(4/3)       （R=水力半径 = A/P）
```

### 两种工作模式

| 模式 | 做什么 | 适用场景 |
|------|--------|---------|
| **inverse（反演）** | 从观测数据学习 S₀ 和 n，同时拟合流场 | 校准模型参数 |
| **forward（正向）** | S₀ 和 n 已知，仅拟合流场 | 预测验证 |

### 断面类型

| 类型 | 构造方式 | 参数 |
|------|---------|------|
| 矩形 | `Channel.rectangular()` | `bottom_width, length` |
| 梯形 | `Channel.trapezoidal()` | `bottom_width, side_slope, length` |
| 幂律（自然河道近似）| `Channel.powerlaw()` | `k, p, length`（`A = k·h^p`） |

### 边界条件类型

| 类型 | YAML 写法 | 说明 |
|------|----------|------|
| 透射（零梯度）| `{type: transmissive}` | 默认，边界取相邻内单元值 |
| 定流量 | `{type: inflow, Q: 5.0}` | 上游恒定 Q |
| 变流量 | `{type: inflow, Q_series: [[t0,Q0],[t1,Q1],...]}` | 上游时变 Q，分段线性 |
| 定水位 | `{type: stage, h: 2.5}` | 下游恒定 h |
| 水位-流量关系 | `{type: rating, curve: [[h0,Q0],[h1,Q1],...]}` | 下游 Q=f(h) 查表 |

---

## 4. Python API 参考

从 `sve_pinns` 导入的所有公开符号：

### RiverModel

```python
model = RiverModel(
    channel,            # Channel 对象，必填
    training=None,      # TrainingConfig，默认反演模式
    duration=200.0,     # 模拟时长 [s]
    g=9.81,             # 重力加速度
    base_depth=2.0,     # 基态水深 [m]
    base_velocity=1.0,  # 基态流速 [m/s]
    perturbation=None,  # 初始扰动 dict：{amplitude, center, sigma}
    boundary=None,      # 边界 dict：{upstream: spec, downstream: spec}
)
```

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `model.fit(observations=None, mode=None)` | self | 训练。observations 为 dict / CSV路径 / None |
| `model.predict(x, t)` | (h, u) | 推理。x 和 t 均为标量/列表/数组 |
| `model.save(path, notes="")` | — | 保存为 .pinn 文件 |
| `model.params` | dict | `{"S0": float, "n": float}` |
| `RiverModel.load(path)` | RiverModel | 从 .pinn 文件加载 |

### Channel

```python
Channel.rectangular(bottom_width=1.0, length=1000.0, lateral_inflow=0.0)
Channel.trapezoidal(bottom_width=5.0, side_slope=1.5, length=1000.0, lateral_inflow=0.0)
Channel.powerlaw(k=1.0, p=1.0, length=1000.0, lateral_inflow=0.0)
Channel.from_yaml(path)  # 从 YAML 的 channel 段读取
```

### TrainingConfig

```python
TrainingConfig(
    mode="inverse",              # "inverse" | "forward"
    adam_iters=8000,             # Adam 优化步数
    lbfgs_iters=300,             # L-BFGS 精细收敛步数
    learning_rate=1e-3,          # Adam 学习率
    collocation_points=5000,     # PDE 残差配点数
    obs_points=200,              # 观测点采样数（仅合成数据）
    ic_points=200,               # 初始条件点数
    bc_points=200,               # 边界条件点数
    noise_assumed=0.0,           # 合成数据噪声水平
    seed=0,                      # 随机种子（可复现）
    true_S0=0.001, true_n=0.03,  # 真值（正向固定，反演对照）
    init_S0=0.002, init_n=0.05,  # 反演初始猜测
    prior_enabled=False,         # MAP 先验开关
    prior_weight=1.0,            # 先验权重
    prior_S0_mean=0.001, prior_S0_std=0.0005,
    prior_n_mean=0.03,  prior_n_std=0.015,
    verbose=True,                # 打印训练日志
)
TrainingConfig.default()         # 全默认值
TrainingConfig.from_yaml(path)   # 从 YAML 读取
```

### 便捷工厂

```python
model_from_yaml("config.yaml")   # YAML 一键构建未训练的 RiverModel
```

### 数据 I/O

```python
load_observations("data.csv")    # 读取 CSV/NPZ 观测表
save_model(model, "m.pinn")      # 函数式保存
load_model("m.pinn")             # 函数式加载
```

### 异常类型（均继承 SVEError）

| 异常 | 触发场景 |
|------|---------|
| `ConfigError` | YAML 解析/校验失败 |
| `DataError` | 观测文件缺失或格式错误 |
| `ModelIOError` | .pinn 读写错误（损坏/签名无效/到期） |
| `TrainingError` | 训练发散、NaN |

---

## 5. YAML 配置文件

除了 Python API，你也可以用 YAML 配置一键训练。

### 完整示例

```yaml
schema_version: "1.1"

# ---- 河道 ----
channel:
  geometry: trapezoidal          # rectangular | trapezoidal | powerlaw
  bottom_width: 5.0              # 底宽 [m]
  side_slope: 1.5                # 边坡（仅梯形）
  length: 1000.0                 # 河长 [m]
  lateral_inflow: 0.0            # 侧向入流 [m²/s]，默认 0

# ---- 物理 ----
physics:
  duration: 200.0                # 模拟时长 [s]
  g: 9.81
  initial:
    base_depth: 2.0              # 初始水深 [m]
    base_velocity: 1.0           # 初始流速 [m/s]
    perturbation:                # 可选初始扰动
      amplitude: 0.3
      center: 500.0
      sigma: 60.0

# ---- 边界条件 ----
boundary:
  upstream:
    type: inflow                 # transmissive | inflow | stage | rating
    Q: 20.0                      # 恒定流量 [m³/s]
    h: 2.0                       # 可选：同时固定水深
  downstream:
    type: stage                  # 定水位
    h: 2.3

# ---- 观测数据（可选：不给则用合成数据）----
data:
  observations: station_data.csv  # 相对此 yaml 的路径
  noise_assumed: 0.0              # 外部数据应填 0

# ---- 训练配置 ----
training:
  mode: inverse                  # forward | inverse
  true_params: { S0: 0.001, n: 0.03 }      # 真值
  initial_guess: { S0: 0.002, n: 0.05 }    # 反演初始猜测
  prior:                         # MAP 先验（噪声鲁棒）
    enabled: true
    weight: 1.0
    S0: { mean: 0.001, std: 0.0005 }
    n:  { mean: 0.03,  std: 0.015  }
  optimizer:
    adam_iters: 8000
    lbfgs_iters: 300
    learning_rate: 1.0e-3
  collocation_points: 5000
  obs_points: 200                # 仅合成采样时有效
  ic_points: 200
  bc_points: 200
  seed: 0
```

### 用 YAML 训练（CLI）

```bash
sve-pinns validate config.yaml      # 校验配置（不训练）
sve-pinns train config.yaml -o model.pinn   # 训练并保存
sve-pinns predict model.pinn --at "500,100 200,50"  # 推理
sve-pinns info model.pinn                      # 查看模型元信息
```

---

## 6. 观测数据格式

### CSV 格式（推荐）

```csv
x,t,h,u,mask_u
50.0,1.47,2.05,0.0,0.0
50.0,8.76,1.99,0.0,0.0
200.0,5.00,2.12,1.05,1.0
```

| 列 | 必填 | 说明 |
|----|------|------|
| `x` | 是 | 沿河道位置 [m] |
| `t` | 是 | 时间 [s] |
| `h` | 是 | 水位 [m] |
| `u` | 否 | 流速 [m/s]；缺测时填 0，`mask_u` 填 0 |
| `mask_u` | 否 | 标记该观测点是否有 u 值（1=有，0=无）；缺省全 1 |

### NPZ 格式

```python
import numpy as np
np.savez("data.npz",
    x=np.array([50.0, 200.0, ...]),
    t=np.array([1.47, 5.00, ...]),
    h=np.array([2.05, 2.12, ...]),
    u=np.array([0.0, 1.05, ...]),        # 可选
    mask_u=np.array([0.0, 1.0, ...])     # 可选
)
```

### 没有观测数据？

不传 `observations` → `model.fit()` 会从内部数值解（MacCormack 格式）自动生成合成观测数据。适合快速测试。

```python
model.fit()  # 自动合成数据
```

---

## 7. 模型保存与加载

### .pinn 文件格式

`.pinn` 本质是一个 zip 包，内含：
- `manifest.json` — 模型元信息（版本、创建时间、参数）
- `config.yaml` — 训练配置（可复现）
- `weights.pt` — PyTorch 模型权重
- `signature.bin` — HMAC-SHA256 防篡改签名

### 保存与加载

```python
model.save("my_river.pinn", notes="校准模型 v1")
model2 = RiverModel.load("my_river.pinn")
assert model2.params == model.params
```

### 注意事项

- `.pinn` 文件包含签名，手动修改其中任何文件会导致加载失败（防篡改）
- license 到期后 `.pinn` 保存/加载会被禁用，续期后恢复

---

## 8. CLI 命令行

```bash
# 校验 YAML 配置
sve-pinns validate config.yaml

# 训练
sve-pinns train config.yaml -o model.pinn

# 推理（逐点：x,t 等长配对）
sve-pinns predict model.pinn --at "500,100 200,50"

# 查看模型信息
sve-pinns info model.pinn

# 帮助
sve-pinns --help
sve-pinns train --help
```

---

## 9. 性能基准

模型通过 SWASHES 独立解析解库的 24 个 1D 算例验证（子临界/超临界/跨临界含水跃/降雨入流）。详见 [BENCHMARK_REPORT.md](BENCHMARK_REPORT.md)。

核心指标（Manning 摩擦适用范围内 17 个算例）：

| 工况 | 通过率 | n 相对误差 | h 相对 L2 |
|------|--------|-----------|----------|
| 子临界流 | 100% | 0.8–2.6% | 0.2–0.3% |
| 超临界流 | 100% | 0.4–2.0% | 0.1–0.3% |
| 跨临界（含水跃）| 100% | 0.1–11.3% | 0.2% |
| 含降雨 | 100% | 0.7–2.6% | 0.2–0.5% |

---

## 10. 许可

本软件为商业闭源软件，版权所有。安装即表示你同意以下条款：

1. 允许安装并使用 SDK 的公开 API 进行模型训练、推理、保存
2. 允许使用训练产出的 `.pinn` 模型文件用于任何合法目的
3. **禁止**反编译、反汇编、逆向工程 SDK 的任何部分
4. **禁止**再分发、出售、出租 SDK 本身

完整许可条款见 [LICENSE.txt](LICENSE.txt)。

### 授权与续期

授权以 `license.key` 文件形式分发，含到期日期。到期后需续期。

联系：**chs9710@163.com**
