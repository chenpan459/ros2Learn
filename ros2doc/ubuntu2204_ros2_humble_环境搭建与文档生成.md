# Ubuntu 22.04 下 ROS 2 Humble：环境搭建、编译与文档生成

> 说明：LTS 版本为 **22.04**（非 22.4）。以下命令默认使用 `jammy` 与 ROS 2 **Humble**，与当前工作区 `ros2_humble` 一致。

---

## 1. 系统与基础依赖

```bash
sudo apt update && sudo apt install -y \
  locales curl gnupg lsb-release software-properties-common \
  build-essential cmake git python3-pip python3-venv \
  python3-colcon-common-extensions python3-vcstool
```

设置 UTF-8 区域（ROS 2 官方建议）：

```bash
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```

---

## 2. 安装 ROS 2 Humble（二进制）

按 [官方文档](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html) 添加源并安装。典型最小开发环境可用 **Desktop**（含 rviz 等）或 **ROS-Base**（更轻）：

```bash
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" \
  | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

sudo apt update
sudo apt install -y ros-humble-desktop ros-dev-tools
```

每次新开终端加载环境：

```bash
source /opt/ros/humble/setup.bash
```

（可写入 `~/.bashrc`。）

---

## 3. 编译本工作区 `ros2_humble`

在已 `source /opt/ros/humble/setup.bash` 的前提下：

```bash
cd /path/to/ros2Learn/ros2_humble
# 若尚未有 src，需按你方流程用 vcstool 拉取；已有 src 则直接编译
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=RelWithDebInfo
```

编译完成后：

```bash
source install/setup.bash
```

常见注意点：

- 全量源码树体量大，可先只编关心的包：  
  `colcon build --packages-up-to rclcpp`
- 内存不足时可限制并行：  
  `colcon build --parallel-workers 2`

---

## 4. 「生成文档」的几种含义与做法

### 4.1 本仓库 `doc/` 下的 Markdown（含 Mermaid）

这些是静态说明，**无需编译**；若需要 **HTML/PDF** 或统一站点：

| 方式 | 用途 |
|------|------|
| VS Code / Cursor 预览 | 直接打开 `.md`，安装 Mermaid 插件可看图 |
| **MkDocs** + `mkdocs-material` + mermaid 插件 | 生成静态站点 |
| **mdBook** | 适合书式文档，可集成 mermaid |

示例（MkDocs，在项目根或 `doc/` 下自建 `mkdocs.yml` 后）：

```bash
pip install mkdocs mkdocs-material pymdown-extensions
mkdocs serve   # 本地预览
mkdocs build   # 输出到 site/
```

（当前仓库若未配置 `mkdocs.yml`，需自行添加；不属于 ROS 核心工作区的一部分。）

### 4.2 从 C/C++ 源码生成 API 文档（Doxygen）

适用于阅读 **rcl / rmw / rclcpp** 等 API：

```bash
sudo apt install -y doxygen graphviz
```

在**单个包**里若存在 `Doxyfile` 或 CMake 里启用了 `doxygen_add_docs`，则在该包 `build` 目录执行对应 target；很多上游包默认**不**打开 Doxygen，可自行在包根目录生成配置：

```bash
cd /path/to/some_package
doxygen -g Doxyfile
# 编辑 Doxyfile：INPUT、RECURSIVE、OUTPUT_DIRECTORY 等
doxygen Doxyfile
```

生成结果一般在 `html/index.html`，用浏览器打开即可。

### 4.3 官方 ROS 2 **手册**（Sphinx，独立仓库）

若目标是生成与 [docs.ros.org](https://docs.ros.org) 同风格的 **Humble 手册**（非你本地 `ros2_humble/src` 的 API 站）：

```bash
sudo apt install -y python3-pip
pip install --user sphinx sphinx-rtd-theme
git clone https://github.com/ros2/ros2_documentation.git
cd ros2_documentation
pip install --user -r requirements.txt
make html
# 输出在 build/html，打开 build/html/index.html
```

分支需选 **humble**（克隆后 `git checkout humble`）。

### 4.4 包内自带的文档目标（若存在）

部分包会定义 `doc` 或 `docs` 的 CMake target。可在编译后尝试：

```bash
cd ros2_humble
colcon build --packages-select SOME_PACKAGE
# 再进入 build/SOME_PACKAGE 查看是否有 doc 相关 target
cmake --build build/SOME_PACKAGE --target help 2>/dev/null | grep -i doc
```

有则例如：`cmake --build build/SOME_PACKAGE --target doc`。

---

## 5. 建议小结

| 目标 | 推荐做法 |
|------|-----------|
| 能编译、运行本工作区 | Ubuntu 22.04 + `ros-humble-desktop` + `colcon build` + `source install/setup.bash` |
| 维护架构说明 Markdown | 继续放在 `doc/`，编辑器预览；需要站点时用 MkDocs/mdBook |
| 啃 C API | `apt install doxygen graphviz`，对目标包生成或自建 Doxyfile |
| 啃用户级教程与概念 | 克隆 `ros2_documentation`，`humble` 分支，`make html` |

若你后续把「文档」限定为某一种（例如只要 MkDocs 站点或只要 rcl 的 Doxygen），可以在 `doc/` 里再拆一篇专用步骤并配上仓库内的配置文件。
