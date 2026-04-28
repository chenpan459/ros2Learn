# ROS 2 Humble 软件模块源码设计解析（`ros2_humble`）

本文在 [ros2_humble_代码架构说明.md](./ros2_humble_代码架构说明.md) 的工作区拓扑与分层图之上，按**模块**说明**设计意图、关键数据结构、调用链与扩展点**。路径默认相对于工作区根目录 `ros2_humble/`。

---

## 1. 设计总览：为何拆成这些层

ROS 2 把「语言绑定」「ROS 语义」「DDS 细节」拆开，便于多语言、多中间件实现并存：

| 层次 | 职责 | 典型依赖 |
|------|------|-----------|
| **rcutils** | C 侧分配器、日志宏、字符串、错误串、原子等，与 ROS 无关的底层工具 | 无 ROS 概念 |
| **rmw** | 纯头文件 API：node、pub/sub、wait、graph 等与实现无关的中间件抽象 | `rcutils`、`rosidl_runtime_c` |
| **rmw\_implementation** | 在进程内加载**某一个** `rmw_*` 共享库，并把 `rmw_*` 调用转发过去 | `rcpputils::SharedLibrary`、`ament_index` |
| **rcl** | 在 rmw 之上实现 **context、命令行参数、重映射、命名解析、QoS 默认值、loan、图 guard** 等 ROS 语义 | `rmw`、`rcutils` |
| **rclcpp / rclpy** | 面向用户的对象模型、executor、类型安全封装 | `rcl` |
| **rosidl** | 从 `.msg/.srv/.action` 生成类型与 **typesupport**，供 rmw 序列化 | CMake 插件链 |

这样替换 DDS 时只需换 **RMW 实现包**，rcl 与上层 API 尽量保持稳定。

---

## 2. `rcl`：进程与节点生命周期

### 2.1 Context 与 `rmw_context` 的一对一关系

`rcl_context_t` 的私有实现里持有 **`rmw_context_t`**，表示「这一次 `rcl_init` 对应的中间件上下文」：

```29:41:ros2_humble/src/ros2/rcl/rcl/src/rcl/context_impl.h
struct rcl_context_impl_s
{
  /// Allocator used during init and shutdown.
  rcl_allocator_t allocator;
  /// Copy of init options given during init.
  rcl_init_options_t init_options;
  /// Length of argv (may be `0`).
  int64_t argc;
  /// Copy of argv used during init (may be `NULL`).
  char ** argv;
  /// rmw context.
  rmw_context_t rmw_context;
};
```

**设计要点**：`instance_id`、`domain_id`、`localhost_only`、`enclave`、安全选项等先在 rcl 侧从环境变量与参数解析好，再填入 `rmw_init_options`，最后调用 **`rmw_init`**，保证 DDS 只看到已规范化的初始化选项。

### 2.2 `rcl_init` 中的关键顺序

`rcl_init` 在完成参数校验与 `context->impl` 分配后，会零初始化 `rmw_context`，再解析 argv、设置 instance_id、domain、localhost、enclave、security，最后进入中间件初始化：

```213:216:ros2_humble/src/ros2/rcl/rcl/src/rcl/init.c
  // Initialize rmw_init.
  rmw_ret_t rmw_ret = rmw_init(
    &(context->impl->init_options.impl->rmw_init_options),
    &(context->impl->rmw_context));
```

**设计要点**：失败路径用 `goto fail` 统一清理，避免 rmw/rcl 状态不一致；`rcl_shutdown` 与 `rcl_context_fini` 的职责分割写在 `context.c` / `init.h` 注释中（先 shutdown 再 fini）。

### 2.3 Node：`rcl` 句柄 + `rmw_node_t`

节点的实现结构显式持有 **`rmw_node_t *`** 与图相关的 **guard condition**，其余命名空间校验、logger 名等围绕其展开：

```58:65:ros2_humble/src/ros2/rcl/rcl/src/rcl/node.c
struct rcl_node_impl_s
{
  rcl_node_options_t options;
  rmw_node_t * rmw_node_handle;
  rcl_guard_condition_t * graph_guard_condition;
  const char * logger_name;
  const char * fq_name;
};
```

**设计要点**：`rcl_node_t` 对用户是不透明句柄；`rmw_node_t` 才是 DDS participant 侧「节点」的抽象。上层只通过 `rcl_*` API 访问，便于测试 mock 与多 RMW。

### 2.4 Publisher：ROS 语义在前，`rmw_create_publisher` 在后

`rcl_publisher_init` 的典型顺序：**校验 → 解析/重映射 topic → 分配 `impl` → 调 rmw 创建 → 取 actual QoS**：

```81:117:ros2_humble/src/ros2/rcl/rcl/src/rcl/publisher.c
  // Expand and remap the given topic name.
  char * remapped_topic_name = NULL;
  rcl_ret_t ret = rcl_node_resolve_name(
    node,
    topic_name,
    *allocator,
    false,
    false,
    &remapped_topic_name);
  ...
  publisher->impl->rmw_handle = rmw_create_publisher(
    rcl_node_get_rmw_handle(node),
    type_support,
    remapped_topic_name,
    &(options->qos),
    &(options->rmw_publisher_options));
  RCL_CHECK_FOR_NULL_WITH_MSG(
    publisher->impl->rmw_handle, rmw_get_error_string().str, goto fail);
```

**设计要点**：topic 字符串在进入 rmw 之前已定型，rmw 不负责 remap；**`rosidl_message_type_support_t *`** 把具体消息类型与 typesupport 绑定传给 rmw，实现多消息类型多模板实例化。

### 2.5 Wait set：rcl 索引 + rmw 等待原语

`rcl_wait_set` 的 `impl` 维护各类实体计数，并持有 **`rmw_wait_set_t *`**，与 `rmw_subscriptions_t` 等聚合类型对齐，便于最终调用 `rmw_wait`：

```36:61:ros2_humble/src/ros2/rcl/rcl/src/rcl/wait.c
struct rcl_wait_set_impl_s
{
  // number of subscriptions that have been added to the wait set
  size_t subscription_index;
  rmw_subscriptions_t rmw_subscriptions;
  ...
  rmw_wait_set_t * rmw_wait_set;
  // number of timers that have been added to the wait set
  size_t timer_index;
  // context with which the wait set is associated
  rcl_context_t * context;
  // allocator used in the wait set
  rcl_allocator_t allocator;
};
```

**设计要点**：**Timer 在 rcl 层维护**，不直接进入所有 rmw 实现；Executor 通过 `rcl_wait` 把「有消息 / 有 guard / 定时器到期」统一成一次阻塞唤醒，这是 rclcpp 单线程与多线程 executor 的公共底座。

---

## 3. `rclcpp`：组合优于巨类

### 3.1 Node 的「接口拆分」

`rclcpp::Node` 通过 **`node_interfaces`** 把能力拆成多个接口类（topics、services、parameters、graph、clock、logging、waitables 等），每个接口有 `*Interface` 与具体实现类（如 `NodeBase`）。**`NodeBase`** 直接持有 `rcl_node_t` 相关资源与 `Context`：

```36:48:ros2_humble/src/ros2/rclcpp/rclcpp/include/rclcpp/node_interfaces/node_base.hpp
/// Implementation of the NodeBase part of the Node API.
class NodeBase : public NodeBaseInterface, public std::enable_shared_from_this<NodeBase>
{
public:
  RCLCPP_SMART_PTR_ALIASES_ONLY(NodeBase)

  RCLCPP_PUBLIC
  NodeBase(
    const std::string & node_name,
    const std::string & namespace_,
    rclcpp::Context::SharedPtr context,
    const rcl_node_options_t & rcl_node_options,
    bool use_intra_process_default,
    bool enable_topic_statistics_default);
```

**设计要点**：便于单测替换某一接口；`enable_shared_from_this` 支持在回调里安全获取 Node 的 `shared_ptr`；**默认进程内通信、topic 统计**等策略在构造期注入。

### 3.2 Executor：与通信图解耦的执行模型

基类 **`rclcpp::Executor`** 文档说明其职责是调度「可用工作」（订阅回调、定时器等），并明确与 **`rcl/wait`** 的关系：

```55:72:ros2_humble/src/ros2/rclcpp/rclcpp/include/rclcpp/executor.hpp
/// Coordinate the order and timing of available communication tasks.
/**
 * Executor provides spin functions (including spin_node_once and spin_some).
 * It coordinates the nodes and callback groups by looking for available work and completing it,
 * based on the threading or concurrency scheme provided by the subclass implementation.
 * An example of available work is executing a subscription callback, or a timer callback.
 * The executor structure allows for a decoupling of the communication graph and the execution
 * model.
 * See SingleThreadedExecutor and MultiThreadedExecutor for examples of execution paradigms.
 */
class Executor
```

**设计要点**：**CallbackGroup** 控制可重入性与互斥；`MultiThreadedExecutor` 用线程池 `spin`，与 `SingleThreadedExecutor` 共享「建 wait set → wait → 分发」思路，差异在并发与锁。

---

## 4. `rosidl`：接口描述到运行时的流水线

### 4.1 CMake 聚合入口

`rosidl_generate_interfaces` 宏负责收集 `.msg/.srv/.action`，并驱动各 generator 的扩展点（注释即设计说明）：

```15:31:ros2_humble/src/ros2/rosidl/rosidl_cmake/cmake/rosidl_generate_interfaces.cmake
#
# Generate code for ROS IDL files using all available generators.
#
# Execute the extension point ``rosidl_generate_interfaces``.
...
#   If the parent directory is 'action', it is assumed to be
#   an action definition.
#   If an action interface is passed then you must add a depend tag for
#   'action_msgs' to your package.xml, otherwise this macro will error.
```

**设计要点**：**action** 通过目录约定与 `action_msgs` 依赖显式区分；非 `.idl` 会先走 `rosidl_adapter` 转成 `.idl`，兼容历史 `.msg` 工作流。

### 4.2 运行时 C 类型与支持结构

`rosidl_runtime_c` 提供 **`message_type_support_struct`** 等，与 rmw 的「发布某类型」签名衔接（见上文 `rcl_publisher_init` 的 `type_support` 参数）。序列化细节由 **`rosidl_typesupport_*`**（introspection、fastrtps、cpp 等）在编译期选链。

---

## 5. `rcl_action`：用已有原语组合 Action

Action 在实现上**不增加新的 rmw 原语**，而是在 **`rcl_client` + `rcl_subscription`** 上组合：例如 client 侧 impl 内含多个 client 与 subscription：

```49:67:ros2_humble/src/ros2/rcl/rcl_action/src/rcl_action/action_client.c
rcl_action_client_impl_t
_rcl_action_get_zero_initialized_client_impl(void)
{
  rcl_client_t null_client = rcl_get_zero_initialized_client();
  rcl_subscription_t null_subscription = rcl_get_zero_initialized_subscription();
  rcl_action_client_impl_t null_action_client = {
    null_client,
    null_client,
    null_client,
    null_subscription,
    null_subscription,
    ...
  };
  return null_action_client;
}
```

**设计要点**：goal / cancel / result 走 service 语义对应的 client；feedback、status 走 subscription；**状态机**（`goal_state_machine.c`）在 rcl 层统一，降低 rmw 负担。

---

## 6. `rmw_implementation` 与多实现共存

运行时通过 **`RMW_IMPLEMENTATION`** 或 ament index 选择共享库，把符号解析到具体实现（详见 `rmw_implementation/src/functions.cpp` 中 `load_library()` 注释）。**设计要点**：二进制包可只装一种 RMW，源码工作区可同时编译多种，用环境变量切换，无需重编应用。

---

## 7. 其他模块（简析）

| 模块 | 源码位置（示例） | 设计角色 |
|------|------------------|----------|
| **rcl_lifecycle** | `src/ros2/rcl/rcl_lifecycle/` | 在节点之上叠加有限状态机与 transition 服务，与 `rclcpp_lifecycle` 配合 |
| **rcl_yaml_param_parser** | `src/ros2/rcl/rcl_yaml_param_parser/` | 将 YAML 参数文件解析为供 rcl/rclcpp 使用的结构 |
| **launch / launch_ros** | `src/ros2/launch*` | 描述式启动：ComposableNode、参数、重映射以数据结构表达，可 Python/XML 执行 |
| **rosbag2** | `src/ros2/rosbag2/` | 存储抽象（sqlite/mcap）+ 传输层复用，与 `rclcpp` 序列化路径对接 |
| **geometry2 / tf2** | `src/ros2/geometry2/` | 坐标变换树，独立于 DDS 高频查询路径 |
| **sros2** | `src/ros2/sros2/` | 安全策略与 keystore，与 DDS governance 配置衔接（如 `policy/schemas`） |

---

## 8. 源码阅读路径建议

1. **`rmw/rmw/include/rmw/rmw.h`**：建立「中间件必须提供哪些原语」的心智模型。  
2. **`rcl/init.c` → `context_impl.h` → `node.c` → `publisher.c` → `wait.c`**：沿一条从进程启动到收发与等待的纵向切片。  
3. **`rclcpp/node_interfaces/` + `executor.hpp`**：看 C++ 如何把 rcl 拆成可测试、可扩展的接口。  
4. **`rosidl_generate_interfaces.cmake`** + 任意消息包的 `rosidl_generator_*` 生成物：理解编译期代码生成。  
5. **`rcl_action/action_client.c`**：理解「组合模式」在 ROS 2 API 设计中的典型用法。

---

## 9. 文档说明

本文描述 **Humble 分支在本工作区中的通用设计**；具体函数行为以头文件注释与单元测试为准。若与 [ros2_humble_代码架构说明.md](./ros2_humble_代码架构说明.md) 中的表格冲突，以 **`src/` 实际目录** 为准。

**`src/` 下各包路径与职责逐项列表**：[ros2_humble_src_源码详细解释.md](./ros2_humble_src_源码详细解释.md)。**仅 `src/ros2/` 目录结构、分层与阅读顺序**：[ros2_humble_src_ros2_源码详解.md](./ros2_humble_src_ros2_源码详解.md)。
