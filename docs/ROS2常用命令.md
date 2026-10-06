# ROS2 常用命令（速查）

- 版本：ROS2 Humble（和 26 赛季 UPPC、四足 `legged_ws` 保持一致）
- 工作空间下文用 `~/rc2027_ws` 代替，按你们实际路径改
- 只收"平时真的会敲"的命令；冷门的、一次性的不列

---

## 0. 每个终端的第一步

```bash
source /opt/ros/humble/setup.bash
source ~/rc2027_ws/install/setup.bash
```

> 忘了第二条，最常见的结果就是"找不到包"或"我自己的节点不在这"。

---

## 1. 编译（colcon）

```bash
colcon build --symlink-install              # 全编；python 包改代码不用重编，最常用
colcon build --packages-select rc2027_mission   # 只编一个包
colcon build --packages-up-to rc2027_bringup    # 连同它依赖的一起编
colcon test && colcon test-result --verbose     # 跑测试
rm -rf build install log                    # 编崩了、出现怪错误时，清掉重来
```

**`--symlink-install` 一定要加**：Python 节点改了代码直接生效，不用每次重编，能省大量时间。

---

## 2. 跑起来

```bash
ros2 launch rc2027_bringup br.launch.py sim:=true    # 仿真
ros2 launch rc2027_bringup br.launch.py sim:=false   # 真车
ros2 launch rc2027_bringup br.launch.py --show-args  # 看这个 launch 有哪些参数
ros2 run rc2027_state odom_fusion                    # 只跑单个节点
```

---

## 3. 看有什么在跑

```bash
ros2 node list                    # 有哪些节点
ros2 node info /br/mission_fsm    # ★这个节点收什么、发什么（查连线最快）
ros2 topic list -t                # 有哪些话题，带消息类型
ros2 topic info /br/odom -v       # ★谁在发、谁在收、QoS 对不对（收不到数据时先看这个）
ros2 service list
ros2 action list
ros2 param list /br/chassis_controller
```

---

## 4. 看数据（调试主力）

```bash
ros2 topic echo /br/odom --once            # 看一帧
ros2 topic echo /br/blocks                 # 一直刷
ros2 topic echo /br/odom --field pose.pose.position.x   # 只看一个字段
ros2 topic hz /br/odom                     # ★频率对不对（该 50Hz 却在 8Hz，一眼看出）
ros2 topic bw /br/imu                      # 带宽，评估串口 / USB 够不够
ros2 interface show nav_msgs/msg/Odometry  # 这个消息有哪些字段
```

---

## 5. 手动发命令（不写代码就能测）

```bash
# 给底盘发速度
ros2 topic pub /br/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.3, y: 0.0}, angular: {z: 0.0}}"

# 只发一次
ros2 topic pub --once /br/mech_command rc2027_msgs/msg/MechCommand "{...}"

# 按 10Hz 一直发（比如模拟"按住前进"）
ros2 topic pub -r 10 /br/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.3}}"

# 调参数（线上改，不用重启）
ros2 param get /br/chassis_controller max_speed
ros2 param set /br/chassis_controller max_speed 2.5

# 调服务
ros2 service call /br/set_detection rc2027_msgs/srv/SetDetection "{enable: true}"

# 发动作目标（定点导航）
ros2 action send_goal /br/goto_pose rc2027_msgs/action/GotoPose "{...}" --feedback
```

---

## 6. 录与回放（做回归测试的关键）

```bash
ros2 bag record -o logs/run01 /br/odom /br/blocks /br/slots /br/motor_state
ros2 bag info logs/run01         # 录了多久、多少条
ros2 bag play logs/run01         # 回放：不用真车、不用 Gazebo，就能测决策
ros2 bag play logs/run01 -r 2.0  # 2 倍速回放
```

> 录一次真车数据，就能在电脑上反复测决策层，这是"能单独喂数据测它"这条标准的实际用法。

---

## 7. 仿真相关（Gazebo + ros2_control）

```bash
ros2 control list_controllers                  # 控制器起没起、是不是 active
ros2 control list_hardware_interfaces          # 关节接口状态
ros2 run tf2_tools view_frames                 # 生成 TF 树 PDF
ros2 run tf2_ros tf2_echo base_link laser      # 查两个坐标系之间的变换
ros2 run rqt_graph rqt_graph                   # 节点连线图（比看话题列表直观）
```

---

## 8. 出问题时的三招

```bash
ros2 node info /br/mission_fsm     # 1. 我收的话题名对不对？
ros2 topic info /br/odom -v        # 2. QoS 对不对？（Python 节点和 Gazebo 插件对不上，八成是 QoS）
ros2 doctor --report               # 3. 环境本身有没有问题
ros2 daemon stop && ros2 daemon start   # list 命令卡住 / 返回不全时，重启守护进程
```

---

## 9. 本项目最常用的 8 条

```bash
colcon build --symlink-install
ros2 launch rc2027_bringup br.launch.py sim:=true
ros2 node info /br/mission_fsm
ros2 topic hz /br/odom
ros2 topic echo /br/blocks
ros2 topic pub -r 10 /br/cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.3}}"
ros2 bag record -o logs/run01 /br/odom /br/blocks /br/slots
ros2 bag play logs/run01
```

---

## 10. 两条经验

1. **`ros2 topic hz` 是最好的朋友**。仿真里"车不动"这类问题，一半是话题名不对，一半是频率为 0（根本没发）。
2. **收不到数据先看 `ros2 topic info -v`**，重点看 QoS：一边是 `RELIABLE`、一边是 `BEST_EFFORT` 时，两边就是接不上，且不报错。
