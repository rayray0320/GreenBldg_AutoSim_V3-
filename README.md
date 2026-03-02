# 🏗️ GreenBldg_AutoSim_V3 (建筑绿色性能模拟自动化生成工具)

![Version](https://img.shields.io/badge/Version-3.0-blue.svg)
![Python](https://img.shields.io/badge/Python-3.8%2B-brightgreen.svg)
![License](https://img.shields.io/badge/License-MIT-green.svg)

**GreenBldg_AutoSim** 是一款基于 Python 开发的开源工程自动化工具，旨在打破传统建筑 2D 平面设计与 3D 多物理场性能模拟（BEM、CFD、Daylighting）之间的“数据孤岛”。

本工具可一键将普通的 2D CAD（`.dxf`）图纸，自动转化为可以直接运行的 **EnergyPlus** (能耗)、**OpenFOAM** (风环境) 和 **Radiance** (光环境) 仿真模型，并基于内置的中国国家建筑节能规范，智能推断并补全缺失的热工与负荷参数。

---

## ✨ 核心特性 (Key Features)

- 📐 **智能几何清洗与拓扑重构**
  - 解析 DXF 中的直线 (`LINE`) 与多段线 (`LWPOLYLINE`)。
  - 基于 **K-Means / DBSCAN** 降维聚类算法，将双线墙体智能合成为单根传热拓扑中线。
  - 引入容差吸附与图论闭合算法 (`Shapely`)，自动识别建筑热区 (Thermal Zones) 并精准打断门窗洞口。
- 🧠 **基于国家规范的参数智能补全**
  - 内置基于 SQLite 的 `MetaModel` 关系型数据库。
  - 涵盖《GB 50189 公共建筑节能设计标准》及各气候区居住建筑节能规范（严寒、寒冷、夏热冬冷、夏热冬暖、温和）。
  - 根据图纸提取的“房间文字”，自动映射并填补人员密度、照明功率 (LPD)、设备功率 (EPD) 及围护结构 U 值/SHGC。
- 🌐 **多物理场引擎一键导出**
  - **EnergyPlus**: 自动生成包含几何、材质、负荷与时间表的 `.idf` 宏文件。
  - **OpenFOAM**: 全自动生成 `blockMesh`, `snappyHexMesh`, `controlDict` 及初始化场边界条件 (k-ε RANS)。
  - **Radiance**: 自动提取玻璃透过率与不透明构件反射率，生成 `.rad` 材质与面片多边形。
- 💻 **开箱即用的 Web 交互界面**
  - 基于 `Streamlit` 构建现代化 UI，无需编写代码，浏览器内一键上传图纸、选择气候区并下载仿真包。

---

## 🚀 快速开始 (Quick Start)

### 1. 环境依赖安装
请确保您的电脑已安装 Python 3.8 或更高版本。
克隆本仓库后，在终端运行以下命令安装所需依赖库：

```bash
pip install numpy pandas ezdxf scikit-learn shapely streamlit

```

### 2. 运行 Web 图形化界面 (推荐)

通过 Streamlit 启动现代化交互界面：

```bash
streamlit run app.py

```

此时浏览器会自动打开 `http://localhost:8501`。您只需：

1. 上传您的 `.dxf` 建筑底图。
2. 在下拉菜单选择项目所在的气候区（如：寒冷、夏热冬冷等）。
3. 点击 **“🚀 开始一键生成模拟模型”**，稍等片刻即可下载包含三大物理引擎的 `Simulation_Outputs.zip` 压缩包！

### 3. 命令行静默运行 (CLI)

如果您希望在后台或脚本中调用，可以直接运行主程序：

```bash
python main.py

```

按提示输入文件路径与地区序号即可完成生成。

---

## 📏 图纸绘制规范 (DXF Preparation Guide)

为了让算法能准确识别您的图纸，请在 AutoCAD 导出 `.dxf` 前遵循以下简单的图层命名规则（不区分大小写）：

* **墙体线段**：图层名需包含 `WALL` 或 `墙` (例如：`A-WALL`, `新建墙体`)。
* **门窗线段**：图层名需包含 `WIN`、`WINDOW`、`DOOR` 或 `门`/`窗`。
* **房间功能**：请使用单行文字 (`TEXT`) 或多行文字 (`MTEXT`) 标注房间名称（如"办公室"），并放置在包含 `SPACE` 或 `房间` 的图层中。
* **单位**：推荐使用毫米 (mm) 或米 (m) 为单位绘图，算法会自动读取 DXF 头部变量进行自适应缩放。

---

## 📂 产出文件结构 (Output Structure)

执行成功后，系统会生成本地数据库 `metamodel.db` 并输出三大引擎的工作目录：

```text
📦 Simulation_Outputs.zip
 ┣ 📂 energyplus_run
 ┃ ┗ 📜 model.idf                 # E+ 能耗几何与负荷文件
 ┣ 📂 openfoam_run
 ┃ ┣ 📂 0                         # CFD 初始场边界条件 (U, p, k, nut, epsilon)
 ┃ ┣ 📂 constant
 ┃ ┃ ┗ 📂 triSurface 
 ┃ ┃   ┗ 📜 building.stl          # 建筑外壳 STL 网格
 ┃ ┣ 📂 system                    # 网格划分与求解器控制字典
 ┃ ┗ 📜 Allrun                    # 一键运行 Shell 脚本
 ┗ 📂 radiance_run
   ┣ 📜 geometry.rad              # 采光建筑多边形几何体
   ┗ 📜 material.rad              # 材质反射与透光率定义

```

---

## 🛠️ 架构与技术栈 (Architecture)

本工具采用了高度解耦的数据流水线（Pipeline）设计：
`Phase 1: 几何解析` -> `Phase 2: MetaModel 持久化` -> `Phase 3: 数据智能推断` -> `Phase 4: 异构文件渲染`。
所有的中间态数据均存储于 SQLite 的关系型数据库中，极大地增强了工具的可扩展性与数据的可追溯性。

---

## 📄 开源协议 (License)

本项目采用 [MIT License](https://www.google.com/search?q=LICENSE) 协议开源，欢迎自由使用、修改与二次开发。

```

```
