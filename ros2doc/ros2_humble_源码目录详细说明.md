# `ros2_humble` 目录与源码结构详细说明

本文说明本工作区 **`ros2_humble/`** 根目录及各 **`src/`** 子树的**组织方式、各仓库职责、与编译产物目录的关系**。更偏「**目录与包级**」说明；分层调用关系见 [ros2_humble_代码架构说明.md](./ros2_humble_代码架构说明.md)，模块内类/函数设计见 [ros2_humble_软件模块源码设计解析.md](./ros2_humble_软件模块源码设计解析.md)。

**`src/` 下各仓库内 colcon 包逐项说明**（含 `ament/`、`ros2/rcl`～`rosbag2`、`ros2cli` 等）：见姊妹篇 [ros2_humble_src_源码详细解释.md](./ros2_humble_src_源码详细解释.md)。**仅 `src/ros2/` 的源码结构、分层与阅读路径**：见 [ros2_humble_src_ros2_源码详解.md](./ros2_humble_src_ros2_源码详解.md)。

---

## 1. 根目录一览

| 路径 | 说明 |
|------|------|
| **`src/`** | 由 `ros2.repos`（及可能的额外 clone）拉取的**全部上游源码**；colcon 只编译此处出现的包。 |
| **`build/`** | 各包的 CMake/Python 构建中间目录，**可删后重编**（`colcon build` 会再生）。 |
| **`install/`** | 安装前缀；`source install/setup.bash` 后使用已安装包、消息与插件索引。 |
| **`log/`** | colcon 每次构建的日志（stdout、事件等），便于排错。 |
| **`ros2.repos`** | **vcstool** 用的清单：列出各 git 仓库 URL 与分支/tag（本工作区为 **Humble** 相关版本）。 |

工作区**不是**单一 git 仓库，而是**多仓库并列**；版本以 `ros2.repos` 与各仓库内 `package.xml` 为准。

---

## 2. `src/` 顶层组织（按目录）

下列目录为 `src/` 下**第一级**组织名，对应社区或厂商维护的仓库集合。

### 2.1 `src/ament/` — 构建与元数据基础

| 子目录 | 作用简述 |
|--------|----------|
| **ament_cmake** | 扩展 CMake：`ament_package()`、测试、导出依赖等 ROS 2 包构建惯例。 |
| **ament_index** | 在 install 前缀内维护**资源索引**（包名、共享库前缀、RMW 实现注册等），供运行时查找。 |
| **ament_lint** | 代码风格与静态检查（uncrustify、pycodestyle、cppcheck 等）的 CMake 封装。 |
| **ament_package** | 解析 `package.xml`、Python 侧包元数据。 |
| **google_benchmark_vendor / googletest / uncrustify_vendor** | 将第三方依赖以 **vendor** 形式打入工作区，保证版本与可重复构建。 |

### 2.2 `src/eProsima/` — Fast DDS 栈

| 子目录 | 作用简述 |
|--------|----------|
| **Fast-CDR** | CDR 序列化库，DDS 常用底层编码。 |
| **Fast-DDS** | eProsima 的 DDS 实现；与 `rmw_fastrtps` 配合。 |
| **foonathan_memory_vendor** | Fast-DDS 依赖的内存分配器库的 vendor 包装。 |

### 2.3 `src/eclipse-cyclonedds/` — CycloneDDS

| 子目录 | 作用简述 |
|--------|----------|
| **cyclonedds** | Eclipse CycloneDDS 实现；与 `rmw_cyclonedds_cpp` 配合。 |

### 2.4 `src/eclipse-iceoryx/` — iceoryx

| 子目录 | 作用简述 |
|--------|----------|
| **iceoryx** | 高性能进程间通信框架；与部分零拷贝、共享内存路径相关（依 RMW/配置而定）。 |

### 2.5 `src/ros2/` — ROS 2 核心与官方生态（重点）

以下按**功能分组**列出当前工作区中 **`src/ros2/`** 下各**一级子目录**（每个通常对应一个或多个 colcon 包）。

#### 客户端库与 RCL 家族

| 目录 | 作用简述 |
|------|----------|
| **rcl** | C 客户端库：init、context、node、pub/sub、service、client、timer、wait、graph 等。 |
| **rclcpp** | C++ 客户端库：`Node`、`Executor`、组件、参数、QoS 封装等。 |
| **rclpy** | Python 客户端库。 |
| **rcl_interfaces** | 参数、日志等标准接口消息定义。 |
| **rcl_logging** | 日志后端抽象（如 spdlog）。 |
| **rcpputils** | C++ 工具（共享库加载、环境变量等）。 |
| **rcutils** | C 工具：分配器、错误串、日志宏、字符串等。 |
| **rpyutils** | Python 侧通用小工具。 |

#### 中间件抽象与实现

| 目录 | 作用简述 |
|------|----------|
| **rmw** | RMW **C 头文件 API**（无单独实现库）。 |
| **rmw_implementation** | 运行时**加载**具体 `rmw_*` 共享库并转发调用。 |
| **rmw_fastrtps** | Fast DDS 的 RMW 实现（C++）。 |
| **rmw_cyclonedds** | CycloneDDS 的 RMW 实现。 |
| **rmw_connextdds** | RTI Connext 的 RMW 实现。 |
| **rmw_dds_common** | 多 DDS RMW 共用的图与工具代码。 |

#### 接口描述与代码生成（rosidl）

| 目录 | 作用简述 |
|------|----------|
| **rosidl** | 核心：CMake 插件、`rosidl_generate_interfaces`、多种语言/ typesupport 生成器与运行时。 |
| **rosidl_dds** | 与 DDS IDL 相关的生成支持。 |
| **rosidl_defaults** | 默认启用的生成器组合。 |
| **rosidl_python** | Python 生成与运行时衔接。 |
| **rosidl_runtime_py** | Python 运行时辅助。 |
| **rosidl_typesupport** | typesupport 接口与实现（如 introspection）。 |
| **rosidl_typesupport_fastrtps** | 面向 Fast DDS 的 typesupport。 |

#### 消息与示例

| 目录 | 作用简述 |
|------|----------|
| **common_interfaces** | 常用标准消息（geometry、sensor 等）。 |
| **example_interfaces** | 教程用简单 `.msg/.srv/.action`。 |
| **unique_identifier_msgs** | UUID 等标识消息。 |
| **test_interface_files** | 测试用接口定义。 |
| **examples** / **demos** | 官方示例与演示包。 |

#### 启动、命令行、安全与测试

| 目录 | 作用简述 |
|------|----------|
| **launch** | 通用启动框架（Python）。 |
| **launch_ros** | ROS 2 专用 launch 实体（节点、参数等）。 |
| **ros2cli** | `ros2` 命令行工具实现。 |
| **ros2cli_common_extensions** | 常用 CLI 扩展集合元包。 |
| **sros2** | DDS 安全策略与 keystore 等工具。 |
| **ros2_tracing** | 与 LTTng 等跟踪集成。 |
| **ros_testing** | 测试框架辅助。 |
| **system_tests** | 跨包系统级测试。 |

#### 工具链与 CMake 辅助

| 目录 | 作用简述 |
|------|----------|
| **ament_cmake_ros** | 面向 ROS 包的 ament_cmake 变体/钩子。 |
| **eigen3_cmake_module** / **python_cmake_module** | 查找 Eigen、Python 的 CMake 模块。 |
| **console_bridge_vendor** / **libyaml_vendor** / **mimick_vendor** / **orocos_kdl_vendor** / **pybind11_vendor** / **spdlog_vendor** / **tinyxml_vendor** / **tinyxml2_vendor** / **yaml_cpp_vendor** | 第三方库 **vendor**，固定版本便于构建。 |
| **performance_test_fixture** | 性能测试夹具。 |
| **tlsf** | TLSF 内存分配器包（实时相关场景）。 |
| **realtime_support** | 实时性相关支持与示例。 |

#### 几何、URDF、可视化与录制

| 目录 | 作用简述 |
|------|----------|
| **geometry2** | **TF2**：坐标变换库与消息工具链。 |
| **urdf** | URDF 解析与模型描述相关。 |
| **message_filters** | 基于时间同步的过滤器（常与 TF/感知配合）。 |
| **rviz** | RViz2 可视化（含 Ogre 等 vendor 子包）。 |
| **rosbag2** | 录制与回放（多种存储后端）。 |

### 2.6 `src/ros/` — 与经典 ROS 生态衔接的通用库

| 子目录 | 作用简述 |
|--------|----------|
| **class_loader** | 多态类动态加载（插件体系基础之一）。 |
| **pluginlib** | 插件描述与加载框架。 |
| **urdfdom / urdfdom_headers** | URDF DOM 解析（C++）。 |
| **kdl_parser** | 从 URDF 构建 KDL 树。 |
| **robot_state_publisher** | 发布机器人关节状态对应的 TF。 |
| **resource_retriever** | 通过网络/文件解析 `package://` 等资源 URL。 |
| **ros_environment** | 环境变量钩子（如 distro 名）。 |
| **ros_tutorials** | 经典教程包（ROS 2 适配）。 |

### 2.7 `src/ros-visualization/` — Qt / rqt 工具链

含 **rqt_***、**qt_gui_core**、**python_qt_binding**、**interactive_markers** 等：基于 Qt 的桌面调试与可视化插件生态。

### 2.8 `src/ros-planning/` — 规划相关消息

| 子目录 | 作用简述 |
|--------|----------|
| **navigation_msgs** | 导航栈常用消息（如 costmap、map meta 等）。 |

### 2.9 `src/ros-perception/` — 感知通用包

| 子目录 | 作用简述 |
|--------|----------|
| **image_common** | 相机图像传输与 camera_info 等公共组件。 |
| **laser_geometry** | 激光扫描与 PointCloud 等几何转换。 |

### 2.10 `src/ros-tooling/` — 工具类依赖

| 子目录 | 作用简述 |
|--------|----------|
| **keyboard_handler** | 键盘输入抽象（供 rviz 等使用）。 |
| **libstatistics_collector** | 统计信息采集（与 topic 统计等配合）。 |

### 2.11 `src/gazebo-release/` — Gazebo 相关 CMake/Math vendor

| 子目录 | 作用简述 |
|--------|----------|
| **gz_cmake2_vendor** / **gz_math6_vendor** | 将 Gazebo 新版 CMake/Math 以 vendor 形式引入，供仿真相关栈构建。 |

### 2.12 `src/osrf/` — OSRF 通用基础库

| 子目录 | 作用简述 |
|--------|----------|
| **osrf_pycommon** | launch 等 Python 组件共用小库。 |
| **osrf_testing_tools_cpp** | C++ 测试内存工具等。 |

### 2.13 `src/ros2-rust/` — Rust 生成器（可选扩展）

| 子目录 | 作用简述 |
|--------|----------|
| **rosidl_rust** | 含 **rosidl_generator_rs**：从 rosidl 生成 Rust 绑定（实验/扩展用途）。 |

---

## 3. 编译与依赖的粗粒度顺序（阅读源码时）

1. **ament / rcutils / rosidl 运行时与生成器** — 无 ROS 节点亦可独立测。  
2. **rmw → rmw_implementation → 某一 rmw_*_cpp** — 决定进程内实际 DDS。  
3. **rcl → rclcpp / rclpy** — 应用直接面对的 API。  
4. **geometry2、urdf、rviz、rosbag2** — 在 RCL 之上叠功能。

具体包依赖以各包 **`package.xml`** 为准；`colcon graph` 可查看依赖图。

---

## 4. 与 `ros2.repos` 的关系

- **`ros2.repos`** 声明了本工作区**期望**存在的仓库路径与版本；新增/删减仓库后需重新 `vcs import` 或手动 clone，并注意 **URL 与 branch/tag** 与 Humble 发行说明一致。  
- **`src/ros2/`** 下列出的包名应与官方 **ros2.repos** 或发行版 manifest 对齐；若本地做过裁剪，以 **`ls src/ros2`** 实际结果为准。

---

## 5. 延伸阅读

| 文档 | 内容侧重 |
|------|----------|
| [ros2_humble_src_源码详细解释.md](./ros2_humble_src_源码详细解释.md) | **`src/` 下按包/按仓库** 的逐项说明（与本文互补）。 |
| [ros2_humble_src_ros2_源码详解.md](./ros2_humble_src_ros2_源码详解.md) | **仅 `src/ros2/`**：仓库布局、分层、阅读顺序。 |
| [ros2_humble_代码架构说明.md](./ros2_humble_代码架构说明.md) | 运行时分层、Mermaid 总览、推荐阅读顺序。 |
| [ros2_humble_软件模块源码设计解析.md](./ros2_humble_软件模块源码设计解析.md) | rcl / rclcpp / rosidl / rcl_action 等**内部设计**与源码引用。 |
| [ubuntu2204_ros2_humble_环境搭建与文档生成.md](./ubuntu2204_ros2_humble_环境搭建与文档生成.md) | Ubuntu 22.04 编译与文档生成步骤。 |
| [ROS2命名与机器人操作系统含义辨析.md](./ROS2命名与机器人操作系统含义辨析.md) | ROS 2 命名与「机器人操作系统」概念。 |

---

*本文档随 `ros2_humble/src` 实际目录生成；若你增删了 `ros2.repos` 中的仓库，请同步更新 §2 各表。*
