# `ros2_humble/src` 整体架构图（目录反向分析）

本文基于工作区实际目录 `ros2_humble/src` 进行分层抽象，用一张图展示“从构建基础到机器人应用工具链”的整体架构。

---

## 1. 架构总图

```mermaid
flowchart TB
  subgraph L0["L0 基础构建与依赖层"]
    A0["ament/*\n(ament_cmake, ament_index, ament_lint, ament_package)"]
    V0["*_vendor + CMake modules\n(eigen3_cmake_module, python_cmake_module,\nspdlog/tinyxml/yaml/pybind11 ... )"]
  end

  subgraph L1["L1 通信中间件与抽象层"]
    RMWAPI["ros2/rmw\nRMW C 抽象接口"]
    RMWSEL["ros2/rmw_implementation\n运行时选择/加载 RMW"]
    RMWIMPL["ros2/rmw_fastrtps\nros2/rmw_cyclonedds\nros2/rmw_connextdds\n+ rmw_dds_common"]
    DDS["eProsima/Fast-DDS\nCycloneDDS\n(及 iceoryx 等 IPC 相关能力)"]
  end

  subgraph L2["L2 客户端运行时层"]
    RCUTILS["ros2/rcutils + rcpputils + rpyutils"]
    RCL["ros2/rcl\n(rcl, rcl_action, rcl_lifecycle,\nrcl_yaml_param_parser)"]
    RCLCPP["ros2/rclcpp\n(rclcpp, components, lifecycle, action)"]
    RCLPY["ros2/rclpy"]
    RCLLOG["ros2/rcl_logging"]
  end

  subgraph L3["L3 接口与代码生成层"]
    IFACE["common_interfaces\nrcl_interfaces\nexample_interfaces\nunique_identifier_msgs\n..."]
    ROSIDL["rosidl / rosidl_defaults\nrosidl_python\nrosidl_typesupport*\nrosidl_dds"]
  end

  subgraph L4["L4 系统能力与开发者工具层"]
    LAUNCH["launch + launch_ros"]
    CLI["ros2cli + ros2cli_common_extensions"]
    BAG["rosbag2"]
    RVIZ["rviz + ros-visualization/* (rqt/qt_gui)"]
    TF["geometry2(tf2), urdf, message_filters"]
    SEC["sros2"]
    TRACE["ros2_tracing"]
    TEST["ros_testing + system_tests + demos/examples"]
  end

  A0 --> V0
  V0 --> RMWAPI
  V0 --> RCUTILS
  V0 --> ROSIDL

  RMWAPI --> RMWSEL --> RMWIMPL --> DDS

  RCUTILS --> RCL
  RCL --> RCLCPP
  RCL --> RCLPY
  RCL --> RCLLOG
  RMWAPI --> RCL

  IFACE --> ROSIDL --> RMWAPI
  ROSIDL --> RCLCPP
  ROSIDL --> RCLPY

  RCLCPP --> LAUNCH
  RCLPY --> LAUNCH
  RCLCPP --> CLI
  RCLPY --> CLI
  RCLCPP --> BAG
  RCLCPP --> RVIZ
  RCLCPP --> TF
  RCLPY --> TF
  RCL --> SEC
  RCL --> TRACE
  RCLCPP --> TEST
  RCLPY --> TEST
```

---

## 2. 读图说明（简版）

- **L0** 提供构建系统与第三方依赖管理，是整棵源码树的地基。  
- **L1** 把通信“抽象”和“具体 DDS 实现”解耦：上层只依赖 `rmw`，底层可切换实现。  
- **L2** 是开发者常直接接触的客户端运行时：C 层 `rcl` + C++/Python 封装。  
- **L3** 是接口工程化核心：消息/服务/动作定义经 rosidl 生成后接入运行时与中间件。  
- **L4** 是工程落地层：启动、命令行、录包、可视化、安全、追踪、系统测试。  

---

## 3. 结论

从 `src` 目录可以看出，ROS 2 并非单一通信库，而是一个由 **构建系统 + 接口生成 + 中间件抽象 + 多语言运行时 + 完整工具链** 组成的分层机器人软件平台。

