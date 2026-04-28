# `ros2_humble/src` 源码详细解释

本文专门说明 **`ros2_humble/src/`** 下各路径**是什么、包含哪些 colcon 包、源码大致长什么样**。目录级总览仍可与 [ros2_humble_源码目录详细说明.md](./ros2_humble_源码目录详细说明.md) 对照阅读；运行时分层与模块设计见 [ros2_humble_代码架构说明.md](./ros2_humble_代码架构说明.md) 与 [ros2_humble_软件模块源码设计解析.md](./ros2_humble_软件模块源码设计解析.md)。

---

## 0. 阅读约定

| 概念 | 含义 |
|------|------|
| **组织目录** | `src/` 下第一级，如 `ros2`、`ament`，对应 `ros2.repos` 里的仓库分组。 |
| **仓库目录** | 如 `src/ros2/rcl`，常为一个 git 仓库根；其下可有**多个**并列的 ROS 包。 |
| **colcon 包** | 含 **`package.xml`** 的目录；`colcon build --packages-select <包名>` 只编该包。 |
| **典型布局** | C/C++ 包常见 `include/<包名>/`、`src/`、`test/`；Python 包常见 `setup.py` 与包名同名子目录。 |

下文路径均相对于 **`ros2_humble/src/`**。

---

## 1. `ament/` — 构建系统与索引

### 1.1 `ament/ament_cmake/`（多包仓库）

每个子目录是一个 **ament_cmake** 宏或辅助包，在其它包的 `CMakeLists.txt` 里通过 `find_package(ament_cmake_...)` 使用。

| 包名（目录） | 作用 |
|--------------|------|
| **ament_cmake** | 元包，聚合常用 ament_cmake 组件。 |
| **ament_cmake_core** | `ament_package()` 等核心逻辑。 |
| **ament_cmake_export_*** | 导出依赖、include、库、接口、link flags、targets 等。 |
| **ament_cmake_gtest / gmock / google_benchmark** | 拉接测试框架。 |
| **ament_cmake_pytest / nose** | Python 测试。 |
| **ament_cmake_python** | 安装 Python 模块、混合包。 |
| **ament_cmake_test** | CTest 与 ament 测试集成。 |
| **ament_cmake_vendor_package** | 构建 vendor 第三方源码的模板。 |
| **ament_cmake_gen_version_h / version** | 版本头与版本号。 |
| 其它 `ament_cmake_*` | include 路径、target 依赖、libraries 查询等细粒度 CMake 辅助。 |

### 1.2 `ament/ament_index/`

| 包名 | 作用 |
|------|------|
| **ament_index_cpp** / **ament_index_python** | 在 install 前缀下按**资源类型**查询包列表、前缀路径（RMW 实现、插件等依赖此机制）。 |

### 1.3 `ament/ament_lint/`

大量 **`ament_cmake_<linter>`** 与 **`ament_<linter>`** 包（如 flake8、cppcheck、uncrustify）：在构建或测试阶段对源码做风格/静态检查；**ament_lint_auto** 用于一键启用一组规则。

### 1.4 其它

| 包名 | 作用 |
|------|------|
| **ament_package** | 解析 `package.xml`，供构建与工具链使用。 |
| **googletest** | 内含 `googlemock`、`googletest` 子包，提供 GTest 源码构建。 |
| **google_benchmark_vendor** / **uncrustify_vendor** | 固定上游版本的 vendor 包。 |

---

## 2. 中间件与底层依赖

### 2.1 `eProsima/`

| 路径 | 说明 |
|------|------|
| **Fast-CDR** | CDR 序列化库（通常无 ROS `package.xml`，作为 CMake 子工程被依赖）。 |
| **Fast-DDS** | DDS 实现本体；含大量 C++ 源码与 CMake。 |
| **foonathan_memory_vendor** | Fast-DDS 使用的内存库 vendor。 |

### 2.2 `eclipse-cyclonedds/cyclonedds/`

CycloneDDS 完整实现；**colcon 包名一般为 `cyclonedds`**（以该目录下 `package.xml` 的 `<name>` 为准）。

### 2.3 `eclipse-iceoryx/iceoryx/`

iceoryx 2.x 仓库内常见 ROS 包（示例）：**iceoryx_hoofs**（基础库）、**iceoryx_posh**（POSH 中间件）、**iceoryx_binding_c**、**iceoryx_integrationtest**、**iceoryx_introspection** 等；用于零拷贝/共享内存 IPC，与具体 RMW 功能绑定方式依版本而定。

### 2.4 `gazebo-release/`、`osrf/`

与仿真、launch 测试基础设施相关，见 [源码目录详细说明 §2.11–2.12](./ros2_humble_源码目录详细说明.md#211-srcgazebo-release--gazebo-相关-cmakemath-vendor)。

---

## 3. `ros2/` — 核心栈（按仓库细拆）

**仅 `src/ros2/` 的纵向详解**（分层图、各仓库内 `include/`·`src/` 习惯、阅读顺序）：见 [ros2_humble_src_ros2_源码详解.md](./ros2_humble_src_ros2_源码详解.md)。下列为与本节互补的**速查表**。

### 3.1 `ros2/rcl/` — RCL C 库

| 包名 | 路径 | 内容要点 |
|------|------|----------|
| **rcl** | `rcl/rcl` | **主体**：`include/rcl/*.h` 对外 API；`src/rcl/*.c` 实现 init、node、publisher、subscription、service、client、timer、wait、graph、remap 等。 |
| **rcl_action** | `rcl/rcl_action` | Action 客户端/服务端 C API，内部组合 rcl 的 client/subscription。 |
| **rcl_lifecycle** | `rcl/rcl_lifecycle` | 生命周期状态机 C API。 |
| **rcl_yaml_param_parser** | `rcl/rcl_yaml_param_parser` | 解析参数 YAML 为 rcl 可用结构。 |

### 3.2 `ros2/rclcpp/` — C++ 客户端库

| 包名 | 内容要点 |
|------|----------|
| **rclcpp** | `Node`、`Executor`、`Publisher`、`Subscription`、QoS、参数、`node_interfaces` 等绝大部分用户 API。 |
| **rclcpp_action** | C++ Action 封装。 |
| **rclcpp_components** | **组件节点**（动态加载 `.so`、同进程多节点）。 |
| **rclcpp_lifecycle** | `LifecycleNode` 与 transition 服务封装。 |

### 3.3 `ros2/rclpy/`、`rcpputils/`、`rcutils/`、`rpyutils/`

| 包名 | 内容要点 |
|------|----------|
| **rclpy** | Python 绑定与运行时（常混合 C 扩展与纯 Python）。 |
| **rcutils** | 分配器、错误串、字符串、日志宏、时间与原子等 **C** 工具。 |
| **rcpputils** | **C++** 工具：共享库加载、scope_exit、线程与文件系统小工具等。 |
| **rpyutils** | Python 通用辅助。 |

### 3.4 `ros2/rmw*` — 中间件抽象与实现

| 包名 | 内容要点 |
|------|----------|
| **rmw** | 仅 **头文件** API（`include/rmw/`），无单独实现库。 |
| **rmw_implementation** | 加载 `librmw_*` 并转发符号；含 **test_rmw_implementation** 供一致性测试。 |
| **rmw_implementation_cmake** | CMake 辅助，供依赖 RMW 的包选择/检查实现。 |
| **rmw_fastrtps_cpp** / **rmw_fastrtps_dynamic_cpp** / **rmw_fastrtps_shared_cpp** | Fast DDS 相关 RMW；**shared** 为公共 C++ 实现片段。 |
| **rmw_cyclonedds_cpp** | Cyclone 的 RMW 实现。 |
| **rmw_connextdds**、**rmw_connextdds_common**、**rmw_connextddsmicro**、**rti_connext_dds_cmake_module** | Connext 系实现与 CMake 模块。 |
| **rmw_dds_common** | 多 DDS 实现共享的图、类型发现等逻辑。 |

### 3.5 `ros2/rosidl/` 与关联包 — 接口与代码生成

**`ros2/rosidl/` 仓库内多包：**

| 包名 | 作用 |
|------|------|
| **rosidl_parser** | 解析 `.idl` / 适配后的接口定义。 |
| **rosidl_adapter** | 将历史 `.msg` 等转为 `.idl`。 |
| **rosidl_cmake** | **`rosidl_generate_interfaces`** 等 CMake 宏（生成流水线入口）。 |
| **rosidl_cli** | 命令行调用生成器。 |
| **rosidl_generator_c** / **rosidl_generator_cpp** | 生成 C/C++ 消息结构体与函数。 |
| **rosidl_runtime_c** / **rosidl_runtime_cpp** | 运行时类型与 **type_support** 结构。 |
| **rosidl_typesupport_interface** | typesupport 插件接口定义。 |
| **rosidl_typesupport_introspection_c** / **_cpp** | 基于内省的序列化路径；**_tests** 为测试包。 |

**`ros2/rosidl_defaults/`**：**rosidl_default_generators**、**rosidl_default_runtime** — 默认生成器与运行时依赖元包。

**`ros2/rosidl_dds/rosidl_generator_dds_idl`**：由 rosidl 生成 DDS IDL 相关产物。

**`ros2/rosidl_python/rosidl_generator_py`**：生成 Python 消息类。

**`ros2/rosidl_runtime_py`**：Python 侧消息工具。

**`ros2/rosidl_typesupport_fastrtps/`**：**fastrtps_cmake_module**、**rosidl_typesupport_fastrtps_c**、**rosidl_typesupport_fastrtps_cpp** — 与 Fast DDS 绑定的 typesupport。

**`ros2/rosidl_typesupport/`**：**rosidl_typesupport_c**、**rosidl_typesupport_cpp** — 具体 typesupport 实现注册与封装。

### 3.6 `ros2/rcl_interfaces/` — 标准接口消息

| 包名 | 典型内容 |
|------|----------|
| **builtin_interfaces** | `Time`、`Duration` 等。 |
| **lifecycle_msgs**、**composition_interfaces**、**rosgraph_msgs**、**statistics_msgs**、**action_msgs**、**test_msgs** 等 | 生命周期、组件、图、统计、通用 Action 基础、测试消息。 |
| **rcl_interfaces** | 参数描述、SetParameters、Log 等消息/服务。 |

标准几何/传感器类消息多在 **`common_interfaces/`**（见 §3.7）。完整列表以 `src/ros2/rcl_interfaces/` 下各子目录为准。

### 3.7 `ros2/common_interfaces/`

每个子目录通常是一个**消息包**：如 **std_msgs**、**geometry_msgs**、**sensor_msgs**、**nav_msgs**、**visualization_msgs** 等；**common_interfaces** 为元依赖包；**sensor_msgs_py** 为 Python 辅助。

### 3.8 `ros2/launch/` 与 `ros2/launch_ros/`

| 包名 | 作用 |
|------|------|
| **launch** | 通用 `LaunchDescription`、事件、子进程等。 |
| **launch_xml** / **launch_yaml** | 从 XML/YAML 构建 launch 描述。 |
| **launch_testing**、**launch_pytest**、**launch_testing_ament_cmake** | launch 与测试集成。 |
| **launch_ros** | `Node`、`ComposableNode`、ROS 参数等实体。 |
| **ros2launch** | `ros2 launch` 命令入口。 |
| **launch_testing_ros**、**test_launch_ros** | ROS 侧 launch 测试。 |

### 3.9 `ros2/ros2cli/` — 命令行工具（多包）

每个 **`ros2<子命令>`** 常对应独立包，便于依赖隔离：

| 包名 | 对应命令/功能 |
|------|----------------|
| **ros2cli** | `ros2` 主入口与插件发现框架。 |
| **ros2action** | `ros2 action` |
| **ros2component** | `ros2 component` |
| **ros2doctor** | `ros2 doctor` |
| **ros2interface** | `ros2 interface` |
| **ros2lifecycle** | `ros2 lifecycle` |
| **ros2multicast** | `ros2 multicast` |
| **ros2node** | `ros2 node` |
| **ros2param** | `ros2 param` |
| **ros2pkg** | `ros2 pkg` |
| **ros2run** | `ros2 run` |
| **ros2service** | `ros2 service` |
| **ros2topic** | `ros2 topic` |
| **ros2cli_test_interfaces** | CLI 测试用接口。 |

### 3.10 `ros2/rosbag2/` — 录制与回放

| 包名 | 作用 |
|------|------|
| **rosbag2** | 元包或顶层聚合。 |
| **rosbag2_cpp** / **rosbag2_py** | C++ 与 Python API。 |
| **rosbag2_storage** | 存储抽象接口。 |
| **rosbag2_storage_default_plugins** | 默认存储实现（如 sqlite）。 |
| **rosbag2_storage_mcap**、**mcap_vendor** | MCAP 格式与依赖。 |
| **rosbag2_transport** | 实际订阅/发布与写盘调度。 |
| **rosbag2_compression**、**rosbag2_compression_zstd**、**zstd_vendor** | 压缩与依赖。 |
| **rosbag2_interfaces** | 服务/消息定义。 |
| **sqlite3_vendor**、**shared_queues_vendor** | 第三方 vendor。 |
| **ros2bag** | `ros2 bag` CLI。 |
| **rosbag2_tests**、**rosbag2_test_common**、**rosbag2_storage_mcap_testdata** | 测试与夹具。 |

### 3.11 `ros2/rviz/` — 可视化

| 包名 | 作用 |
|------|------|
| **rviz_common** | 核心数据模型与插件接口。 |
| **rviz_rendering** | 渲染抽象。 |
| **rviz_default_plugins** | 默认显示类型。 |
| **rviz2** | 应用程序入口。 |
| **rviz_ogre_vendor**、**rviz_assimp_vendor** | 图形与模型格式依赖。 |
| **rviz_rendering_tests**、**rviz_visual_testing_framework** | 测试支持。 |

### 3.12 `ros2/ros2_tracing/`

| 包名 | 作用 |
|------|------|
| **tracetools** | 在 rcl/rclcpp 等中插入的跟踪点（tracepoint）宏。 |
| **ros2trace**、**tracetools_trace**、**tracetools_read**、**tracetools_launch** | 命令行与 launch 集成、读取 trace。 |
| **test_tracetools**、**test_tracetools_launch**、**tracetools_test** | 测试。 |

### 3.13 `ros2/geometry2/` — TF2

| 包名 | 作用 |
|------|------|
| **tf2** | 核心变换库（C++）。 |
| **tf2_ros** / **tf2_ros_py** | 节点与话题封装。 |
| **tf2_msgs** | 消息定义。 |
| **tf2_py** | Python 绑定。 |
| **tf2_geometry_msgs**、**tf2_sensor_msgs**、**tf2_eigen**、**tf2_bullet**、**tf2_kdl** 等 | 与各类几何/库的类型转换。 |
| **tf2_tools**、**examples_tf2_py** | 工具与示例。 |
| **geometry2**、**test_tf2** | 元包与测试。 |

### 3.14 `ros2/demos/` — 演示节点

如 **composition**、**demo_nodes_cpp**、**demo_nodes_py**、**image_tools**、**intra_process_demo**、**lifecycle**、**logging_demo**、**pendulum_control**、**topic_statistics_demo** 等：每个子目录多为**独立可运行示例包**，用于展示单一特性。

### 3.15 `ros2/examples/` — 最小示例

按 **`rclcpp/`**、**`rclpy/`** 再分子目录：

- **topics**：minimal_publisher / minimal_subscriber  
- **services**：minimal_service / minimal_client / async_client  
- **actions**：minimal_action_server / client  
- **executors**：multithreaded_executor、cbg_executor  
- **composition**：minimal_composition  
- **timers**、**wait_set**、**guard_conditions** 等  

另有 **launch_testing/launch_testing_examples**：launch 测试示例。

### 3.16 `ros2/system_tests/` 与 `ros2/ros_testing/`

| 包名 | 作用 |
|------|------|
| **test_communication**、**test_rclcpp**、**test_security**、**test_quality_of_service**、**test_cli**、**test_cli_remapping** | 跨栈系统测试。 |
| **ros_testing**、**ros2test** | 测试框架与 ament 集成。 |

### 3.17 `ros2/sros2/`、`ros2/urdf/`、`ros2/message_filters/` 等

| 包名 | 作用 |
|------|------|
| **sros2**、**sros2_cmake** | 安全策略生成与 CMake 钩子。 |
| **urdf**、**urdf_parser_plugin** | URDF 解析与插件接口。 |
| **message_filters** | 时间同步过滤器。 |
| **unique_identifier_msgs**、**test_interface_files**、**example_interfaces** | 消息与测试接口定义。 |
| **各类 *_vendor** | 固定版本第三方库，供 rviz、yaml、logging 等使用。 |

---

## 4. `ros/` — 与机器人模型、插件、教程相关

| 包名 | 作用 |
|------|------|
| **class_loader**、**pluginlib**、**ros2plugin** | 插件加载基础设施。 |
| **urdfdom** | URDF XML DOM 解析。 |
| **kdl_parser**、**kdl_parser_py** | URDF → KDL 树。 |
| **robot_state_publisher** | 关节状态 → TF 广播。 |
| **resource_retriever**、**libcurl_vendor** | 远程/本地资源获取。 |
| **ros_environment** | 发行版环境变量。 |
| **ros_tutorials**（**turtlesim**、**roscpp_tutorials** 等） | 经典教程节点 ROS 2 移植。 |

---

## 5. `ros-visualization/` — Qt 与 rqt

| 区域 | 包示例 |
|------|--------|
| **python_qt_binding** | 在 ROS 中统一绑定 PyQt。 |
| **qt_gui_core** | **qt_gui**、**qt_gui_cpp**、**qt_dotgraph**、**qt_gui_app** 等：rqt 宿主与插件容器。 |
| **rqt/** | **rqt**、**rqt_gui**、**rqt_gui_cpp**、**rqt_gui_py**、**rqt_py_common** 元包与公共库。 |
| **rqt_*** | 各插件：**rqt_graph**、**rqt_console**、**rqt_plot**、**rqt_bag**（含 **rqt_bag_plugins**）等。 |
| **interactive_markers** | 交互式标记服务端库。 |
| **tango_icons_vendor** | 图标资源。 |

---

## 6. `ros-perception/`、`ros-planning/`、`ros-tooling/`、`ros2-rust/`

- **image_common**、**laser_geometry**：感知管线常用库（见 [源码目录详细说明](./ros2_humble_源码目录详细说明.md)）。  
- **navigation_msgs**：导航相关消息。  
- **keyboard_handler**、**libstatistics_collector**：输入与统计采集。  
- **ros2-rust/rosidl_rust**：**rosidl_generator_rs**，从接口生成 Rust。

---

## 7. 如何在本地核对「包名 ↔ 路径」

在已配置 ROS 2 环境的 shell 中（或仅浏览文件系统）：

```bash
# 列出某目录下所有包名（需安装 fd 或自行用 find）
find ros2_humble/src/ros2 -name package.xml -exec grep -H '<name>' {} \;
```

或直接打开某目录下的 **`package.xml`**，其中 **`<name>...</name>`** 即为 **colcon 包名**（可能与目录名不同，以 `<name>` 为准）。

---

## 8. 延伸阅读

| 文档 |
|------|
| [ros2_humble_src_ros2_源码详解.md](./ros2_humble_src_ros2_源码详解.md) |
| [ros2_humble_源码目录详细说明.md](./ros2_humble_源码目录详细说明.md) |
| [ros2_humble_软件模块源码设计解析.md](./ros2_humble_软件模块源码设计解析.md) |
| [ros2_humble_代码架构说明.md](./ros2_humble_代码架构说明.md) |

---

*本文基于当前工作区 `src/` 实际内容整理；上游增删包后请以 `find … package.xml` 结果为准更新。*
