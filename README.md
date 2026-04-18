# python-tutorial

本仓库用于学习与实践 Python（以官方教程为主），并包含一个来自 Nicolas P. Rougier《From Python to Numpy》风格的向量化示例：用 NumPy 将 “Boids（鸟群）” 的邻域规则计算从 Python 循环提升为矩阵/广播运算，并用 Matplotlib 动画可视化与导出 GIF/MP4。

---

## 目录

- [项目亮点](#项目亮点)
- [环境要求](#环境要求)
- [快速开始（推荐：Poetry）](#快速开始推荐poetry)
- [不使用 Poetry 的运行方式](#不使用-poetry-的运行方式)
- [示例说明：spatial_vectorization（Boids）](#示例说明spatial_vectorizationboids)
- [输出文件](#输出文件)
- [常见问题](#常见问题)
- [项目结构](#项目结构)
- [参考](#参考)

---

## 项目亮点

- **向量化 Boids**：使用 `np.subtract.outer / hypot / einsum / 广播(where=...)` 等方式，避免双重 for 循环。
- **高效绘制**：用 `Path + PathCollection` 将 n 个小三角形 marker 拼接成一个大 Path，每帧只更新顶点数组即可渲染整个鸟群。
- **可导出动画**：
  - 有 FFmpeg 时导出 `boid.mp4`
  - 没 FFmpeg 时自动回退导出 `boid.gif`

---

## 环境要求

- Windows / macOS / Linux 均可
- Python：`>= 3.13`（见 `pyproject.toml`）
- 运行示例需要的第三方库：
  - `numpy`
  - `matplotlib`
  - （可选）`pillow`：用于导出 GIF（当没有 FFmpeg 时）
  - （可选）`ffmpeg`：用于导出 MP4（H.264）

说明：
- 当前 `pyproject.toml` 的 dependencies 里只声明了 `ipykernel`。因此如果你使用 Poetry 环境运行示例，需要额外把 `numpy/matplotlib` 安装进 Poetry 虚拟环境（见下文命令）。

---

## 快速开始（推荐：Poetry）

在项目根目录（`c:\Users\Administrator\python-tutorial`）打开 PowerShell：

### 1) 安装 Poetry（如你尚未安装）

如果你已经有 Poetry，可跳过此步。Poetry 官方安装方式会随时间调整，你可以参考官方文档。

### 2) 创建/安装依赖

```powershell
poetry install
```

由于本仓库默认依赖声明较少，运行 Boids 示例还需要把 NumPy 和 Matplotlib 装进 Poetry 环境：

```powershell
poetry add numpy matplotlib
```

如果你的机器没有安装 FFmpeg，想要自动保存 GIF，建议确保 Pillow 可用：

```powershell
poetry add pillow
```

### 3) 运行示例（项目脚本入口）

`pyproject.toml` 里配置了脚本入口：

- `spatial_vectorization = 'python_tutorial.spatial_vectorization:main'`

因此可直接运行：

```powershell
poetry run spatial_vectorization
```

运行后会弹出窗口显示动画，并在当前目录生成 `boid.mp4` 或 `boid.gif`（取决于 FFmpeg 是否可用）。

---

## 不使用 Poetry 的运行方式

如果你更习惯用当前 Python（例如 Anaconda 的 base 环境）：

### 1) 安装依赖

```powershell
pip install numpy matplotlib pillow
```

### 2) 运行模块

在项目根目录执行：

```powershell
python -m python_tutorial.spatial_vectorization
```

如果你在 `src\python_tutorial` 目录里直接执行，有时会遇到包导入路径问题；更推荐在项目根目录用 `python -m ...` 的方式运行。

---

## 示例说明：spatial_vectorization（Boids）

入口文件：

- `src/python_tutorial/spatial_vectorization.py`

核心内容分两块：

### 1) Flock：鸟群规则（数值计算）

`Flock.run()` 每一帧更新所有个体的 `position/velocity`，典型包含三种规则：

- Separation（分离）：太近就推开
- Alignment（对齐）：与邻居速度方向对齐
- Cohesion（凝聚）：向邻居中心靠拢

这份实现的特点是：
- 用 `dx = np.subtract.outer(...)` / `dy = np.subtract.outer(...)` 直接得到两两差分矩阵
- 用 `distance = np.hypot(dx, dy)` 得到两两距离
- 用布尔 mask（如 `<25`, `<50`）选择不同邻域范围
- 用 `dot / sum / where` 计算群体 steering，并限制最大加速度/速度

### 2) MarkerCollection：绘制（批量更新顶点）

为了避免 “每只 boid 一个 artist” 的绘制开销，这里：
- 先定义一个局部坐标系下的小三角形 Path（表示 boid）
- 把它复制 n 份拼成一个大 Path（顶点数组形状类似 `(n, 4, 2)`）
- 每帧对所有 boid 的顶点进行缩放、旋转、平移（向量化），再交给 Matplotlib 渲染

---

## 输出文件

运行脚本后通常会生成：

- `boid.mp4`：当系统 FFmpeg 可用时
- `boid.gif`：当 FFmpeg 不可用时（回退方案）

仓库根目录也可能已有示例输出（例如 `boid.gif`），可直接在 README 中预览：

![boids](boid.gif)

---

## 常见问题

### 1) 为什么必须 `poetry install` 才能 `poetry run spatial_vectorization`？

`poetry run` 会在 Poetry 管理的虚拟环境中执行：
- 没有 `poetry install` 时，这个环境可能不存在或依赖未安装
- 脚本入口（`[project.scripts]`）也不会处于可用状态

结论：第一次使用 Poetry 跑项目脚本时，通常需要先 `poetry install`。

### 2) 报错：`ModuleNotFoundError: No module named 'numpy'` / `matplotlib`

说明当前环境没装依赖：
- Poetry 环境：运行 `poetry add numpy matplotlib`
- 非 Poetry 环境：运行 `pip install numpy matplotlib`

### 3) 报错：`MovieWriter ffmpeg unavailable`

这是提示 **系统未安装 FFmpeg**（或 Matplotlib 找不到 ffmpeg）。本项目会自动回退用 Pillow 导出 GIF：

- 有 FFmpeg：生成 `boid.mp4`
- 无 FFmpeg：生成 `boid.gif`

如果你想导出 MP4：
- 安装 FFmpeg，并确保 `ffmpeg` 在 PATH 中可用（`ffmpeg -version` 能运行）。

### 4) 我只想看动画，不想导出文件

你可以把保存那段代码注释/跳过，只保留 `plt.show()`。
（如果你希望我帮你加一个命令行开关来控制是否保存，也可以。）

---

## 项目结构

```text
python-tutorial/
  src/
    python_tutorial/
      spatial_vectorization.py   # Boids 向量化 + 动画可视化（脚本入口）
  tests/
    __init__.py
  pyproject.toml
  poetry.lock
  README.md
  boid.gif                       # 示例输出（可能存在）
```

---

## 参考

- Nicolas P. Rougier: From Python to Numpy（向量化思路与风格来源）
  - https://www.labri.fr/perso/nrougier/from-python-to-numpy/
- Matplotlib Animation 文档（FFmpegWriter / PillowWriter）
  - https://matplotlib.org/stable/api/animation_api.html