---
name: ros2-dev
description: 当用户提到 ROS2、ROS 2、Humble、Jazzy、colcon、ament、rqt、rviz2、launch、topic/service/action、tf2、rosbag2、Nav2、MoveIt2、Gazebo、机器人节点开发、功能包编写、机器人通信调试、多机 ROS2、机械臂/移动机器人 ROS2 工程时使用。覆盖环境搭建、包结构、节点编写、launch、调试、回放与常见坑，面向机械/机器人方向的工程实践。
---

# ROS2 开发助手（ros2-dev）

## 概述

本 skill 帮助用户在 ROS2（Humble / Jazzy）环境下完成功能包开发、节点编写、通信调试、仿真与数据回放。默认假设用户是机械/机器人方向的工程师或研究生，熟悉 Linux 基本命令，但可能对 ROS2 的概念体系和命令行不熟。

**版本对应（硬约束）**：
- Ubuntu 22.04 LTS → ROS2 **Humble Hawksbill**（LTS，2027 年 5 月 EOL，资料最多，推荐入门）
- Ubuntu 24.04 LTS → ROS2 **Jazzy Jalisco**（LTS，2029 年 5 月 EOL，新装机推荐）
- 不要在 Ubuntu 20.04 上装 Humble，也不要在 22.04 上装 Jazzy——版本错配是最常见的"装完跑不起来"原因。

## 使用场景

- 从零搭建 ROS2 工作空间、创建功能包
- 写 C++（rclcpp）或 Python（rclpy）节点
- 写 launch 文件、配置参数
- 排查 topic 无数据、节点看不到、时钟不同步、DDS 通信失败
- 录制 rosbag2 回放、在 rviz2 / rqt 里可视化
- 接入 Nav2 / MoveIt2 / Gazebo 仿真

不处理：ROS1 迁移细节、源码编译 ROS2 本体、驱动级内核调优——这些情况先建议用户用 apt 二进制安装再继续。

## 核心流程

1. **先问清版本与平台**：Ubuntu 版本？Humble 还是 Jazzy？目标是真机还是仿真？Python 还是 C++？
2. **给命令前先给一行"为什么"**：例如"source 一下是让当前 shell 能找到 ros2 命令"。用户是机械背景，不是 Linux 老玩家。
3. **命令必须可复制粘贴**：所有 shell 命令用 ```bash 代码块，一次给全，不要拆成"你先敲这个再敲那个"。
4. **报错先定位再解决**：让用户贴 `ros2 doctor --report` 和终端原始报错，不要凭猜测给修复方案。

## 一、环境与工作空间

### 1.1 首次安装（apt 方式，推荐）

```bash
# 一定要先设 locale
sudo apt update && sudo apt install -y locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

# 添加 ROS2 源
sudo apt install -y software-properties-common curl gnupg lsb-release
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# Humble (Ubuntu 22.04)
sudo apt update && sudo apt install -y ros-humble-desktop python3-colcon-common-extensions python3-rosdep python3-argcomplete
# Jazzy (Ubuntu 24.04) 把上面的 humble 换成 jazzy
```

### 1.2 每次开新终端必须 source

```bash
# Humble
source /opt/ros/humble/setup.bash
# Jazzy
source /opt/ros/jazzy/setup.bash

# 永久写入 ~/.bashrc 省去每次敲
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc   # 工作空间
```

**坑**：如果你在一个终端 source 了，新开的终端不 source，两个终端里敲 `ros2` 命令一个有一个没有——这是最常见的"我刚才还能跑现在怎么没了"。

### 1.3 工作空间结构

```
~/ros2_ws/
├── src/                # 你自己写的功能包都放这里
├── build/              # colcon 构建中间产物（自动生成）
├── install/            # 安装产物（自动生成）
└── log/                # 构建日志（自动生成）
```

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
colcon build --symlink-install --packages-select YOUR_PKG
# --symlink-install：改 Python 文件不用重新 colcon build，直接生效
# 改了 C++ 才需要重新 build
source install/setup.bash
```

## 二、创建功能包

### 2.1 Python 包（rclpy，适合快速原型、控制逻辑）

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python --node-name my_node my_pkg
```

目录结构：
```
my_pkg/
├── package.xml
├── resource/my_pkg
├── setup.py
├── setup.cfg
└── my_pkg/
    ├── __init__.py
    └── my_node.py
```

### 2.2 C++ 包（rclcpp，适合性能敏感、驱动、算法）

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake --node-name my_node my_pkg_cpp
```

### 2.3 package.xml 里要补的字段

`<description>`、`<maintainer>`、`<license>`（不填 colcon 会警告），`<depend>` 加你用到的消息包：

```xml
<depend>rclcpp</depend>
<depend>std_msgs</depend>
<depend>geometry_msgs</depend>
```

## 三、节点模板

### 3.1 Python 发布者/订阅者（my_pkg/my_pkg/my_node.py）

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Talker(Node):
    def __init__(self):
        super().__init__('talker')
        self.pub = self.create_publisher(String, 'chatter', 10)
        self.timer = self.create_timer(0.5, self.timer_cb)  # 2 Hz

    def timer_cb(self):
        msg = String()
        msg.data = f'hello ros2: {self.get_clock().now().nanoseconds}'
        self.pub.publish(msg)
        self.get_logger().info(f'published: {msg.data}')


def main():
    rclpy.init()
    node = Talker()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    finally:
        node.destroy_node()
        rclpy.shutdown()
```

在 `setup.py` 的 `entry_points` 里注册：
```python
entry_points={
    'console_scripts': [
        'talker = my_pkg.my_node:main',
    ],
},
```

### 3.2 C++ 节点模板

```cpp
#include <chrono>
#include <memory>
#include "rclcpp/rclcpp.hpp"
#include "std_msgs/msg/string.hpp"

using namespace std::chrono_literals;

class Talker : public rclcpp::Node {
public:
  Talker() : Node("talker_cpp") {
    pub_ = this->create_publisher<std_msgs::msg::String>("chatter", 10);
    timer_ = this->create_wall_timer(500ms, [this]() {
      auto msg = std_msgs::msg::String();
      msg.data = "hello from cpp";
      pub_->publish(msg);
    });
  }
private:
  rclcpp::Publisher<std_msgs::msg::String>::SharedPtr pub_;
  rclcpp::TimerBase::SharedPtr timer_;
};

int main(int argc, char ** argv) {
  rclcpp::init(argc, argv);
  rclcpp::spin(std::make_shared<Talker>());
  rclcpp::shutdown();
}
```

CMakeLists.txt 末尾确保：
```cmake
add_executable(talker_cpp src/talker_cpp.cpp)
ament_target_dependencies(talker_cpp rclcpp std_msgs)
install(TARGETS talker_cpp DESTINATION lib/${PROJECT_NAME})
```

## 四、常用命令速查

### 4.1 节点与话题

```bash
ros2 node list                       # 所有节点
ros2 node info /talker               # 某节点发布/订阅了什么
ros2 topic list                      # 所有 topic
ros2 topic echo /chatter            # 实时打印 topic 内容
ros2 topic hz /chatter               # 看发布频率
ros2 topic bw /chatter               # 看带宽
ros2 topic info /chatter -v           # 详细类型和发布者数

ros2 service list                     # 服务列表
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 1, b: 2}"

ros2 action list
ros2 interface show std_msgs/msg/String   # 看消息字段定义
```

### 4.2 包与构建

```bash
ros2 pkg list
ros2 pkg prefix <pkg_name>
colcon build --symlink-install
colcon build --packages-select my_pkg
colcon test --packages-select my_pkg
rm -rf build/ install/ log/         # 全清后重建（遇到玄学错误时）
```

### 4.3 参数与生命周期

```bash
ros2 param list
ros2 param get /node_name param_name
ros2 param set /node_name param_name value
ros2 param dump /node_name --output-file params.yaml   # 导出参数

ros2 lifecycle list /your_lifecycle_node
ros2 lifecycle set /your_lifecycle_node configure
ros2 lifecycle set /your_lifecycle_node activate
```

### 4.4 录制与回放（rosbag2）

```bash
ros2 bag record /chatter /odom -o my_bag        # 录多个 topic
ros2 bag record -a -o all_topics                # 录全部
ros2 bag info my_bag/
ros2 bag play my_bag/
ros2 bag play my_bag/ --rate 0.5                # 半速回放
```

## 五、调试三板斧

1. **`ros2 doctor --report`**：贴出来环境、DDS、网络、ROS_DOMAIN_ID 等所有信息。用户报错先让他跑这个。
2. **`rqt_graph`**：可视化节点和 topic 的连接图。如果两个节点在图上没连上，说明 topic 名、QoS 或消息类型不匹配。
3. **`ros2 topic echo` + `ros2 topic hz`**：发布者在跑但订阅者收不到？先 echo 一下看有没有数据，再 hz 看频率对不对。

### 常见坑

- **`could not find package`**：没 `source install/setup.bash`，或者没 `colcon build`。
- **节点跑起来但 `ros2 node list` 看不到**：两个终端 ROS_DOMAIN_ID 不一致，或者 RMW 实现不一样。统一 `export ROS_DOMAIN_ID=0`。
- **话题有但 echo 不到数据**：QoS 不匹配（传感器数据常用 sensor data QoS，订阅者要匹配）。临时用 `ros2 topic echo --qos-reliability best_effort /topic_name` 测试。
- **多机通信不通**：两台机器要 `export ROS_DOMAIN_ID=<同号>`，且在同一子网；防火墙放开 UDP 7400-7500 段。
- **Python 节点改了代码不生效**：Python 包用了 `--symlink-install` 才生效；C++ 必须重新 `colcon build`。

## 六、仿真栈速查

- **Gazebo Harmonic / Classic**：Humble 默认带 Gazebo Classic（`gazebo` 命令），Jazzy 用新的 `gz sim`。
- **Nav2**：导航栈，配合 `nav2_bringup` 一键起仿真：
  ```bash
  export TURTLEBOT3_MODEL=burger
  ros2 launch nav2_bringup tb3_simulation_launch.py
  ```
- **MoveIt2**：机械臂运动规划，配合 `moveit_setup_assistant` 生成配置包。

## 交付规范

- 给命令时**一次给全一段**，让用户整段粘贴；不要拆成"先敲这个看看"。
- 涉及改 `~/.bashrc` 或系统目录时，先说明改了什么、怎么回滚。
- 用户贴报错时，先让他给完整报错堆栈 + `ros2 doctor --report` 输出，不要凭一行红字猜。
- 推荐版本时优先 Humble（资料多、稳定），除非用户明确在 Ubuntu 24.04 上。
