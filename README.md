# sve-pinns

> Physics-Informed Neural Networks for 1D Saint-Venant Equations.

## 安装

```bash
pip install https://github.com/dancepandas/PINNs-SVE-dist/releases/latest/download/sve_pinns-0.1.0-cp310-cp310-win_amd64.whl
```

（Windows 用户请用 `win_amd64`，Linux 用 `linux_x86_64`，macOS 用 `macosx_10_9_universal2`）

## 快速上手

```python
from sve_pinns import RiverModel, Channel, TrainingConfig, load_observations

# 定义河道
channel = Channel.trapezoidal(bottom_width=5.0, side_slope=1.5, length=1000.0)

# 加载观测
obs = load_observations("station_data.csv")

# 训练（反演 S0, n）
model = RiverModel(channel=channel, training=TrainingConfig(mode="inverse"))
model.fit(observations=obs)
print(model.params)

# 推理
h, u = model.predict(x=[100, 500, 800], t=[0, 50, 100])
```

## 许可

安装后请将 `license.key` 放在工作目录。联系 chs9710@163.com 获取授权。

## 更多

- [完整 API 文档](https://github.com/dancepandas/PINNs-SVE)
- [配置字段说明](https://github.com/dancepandas/PINNs-SVE)
