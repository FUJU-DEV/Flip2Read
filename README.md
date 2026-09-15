# Flip2Read — SO-ARM101 翻转识单

> 翻转快递盒找面单，OCR 读单号 —— 一个基于低成本开源机械臂的 sim2real 项目。

[![ROS2](https://img.shields.io/badge/ROS2-Humble-blue)](https://docs.ros.org/en/humble/)
[![Gazebo](https://img.shields.io/badge/Gazebo-Sim%20Fortress-orange)](https://gazebosim.org/)
[![Python](https://img.shields.io/badge/Python-3.10-green)](https://www.python.org/)

**作者**：[福橘 (FUJU-DEV)](https://github.com/FUJU-DEV) · 2026

---

## 📌 项目简介

桌上放一个快递盒，面单贴在**未知的某一个面**上。机械臂要**翻转盒子找到面单，读取上面的单号**。检测和抓取只是前置步骤，真正的难点是「翻转找面单 + OCR 识别」这个闭环。

**核心理念**：视觉部分**不用颜色分割**（脆弱、无法泛化），而是用 **YOLO 目标检测 + RGBD 深度做 3D 定位**，让系统识别的是「快递盒」这个类别本身，而不是「蓝色的盒子」。

## 🎯 任务流程

| 环节 | 目标 | 技术方案 |
|------|------|---------|
| ① 检测 | 找到快递盒 | YOLOv8（非颜色分割） |
| ② 3D 定位 | 盒子在世界坐标的位置 | RGBD 深度 + 反投影 |
| ③ 抓取 | 夹起盒子 | MoveIt2 + ros2_control |
| ④ **翻转找面单** | 露出贴面单的那一面（未知是 1-2 个面中的哪个） | 翻转动作，限定候选面 |
| ⑤ **OCR** | 读取面单上的单号 | RapidOCR / PaddleOCR |

> **④ + ⑤ 是核心。** 「六个面不知道面单在哪」是难点：面单可能贴在任意一个面，所以机器人要翻转并重新确认，限定在 1-2 个候选面。

## 🧰 技术栈

- **机械臂**：SO-ARM101（LeRobot 开源 6 轴舵机臂 + 夹爪）
- **仿真**：Gazebo Sim（Ignition Fortress）+ ROS2 Humble + MoveIt2
- **视觉**：YOLOv8 + RGBD 深度相机
- **OCR**：RapidOCR（onnxruntime）
- **控制**：ros2_control（joint_trajectory_controller + gripper_controller）

## 🏗️ 系统架构

```
RGBD 相机 ──► YOLOv8 检测 ──► 深度反投影 ──► 3D 坐标
                                              │
                                              ▼
                                    MoveIt2 逆运动学(IK)
                                              │
                                              ▼
                              ros2_control 关节轨迹执行
                                              │
                                              ▼
                              抓取 → 翻转 → 面单 OCR（读单号）
                                  ▲       │
                                  └── 重试 ┘（没找到面单就再翻）
```

## 🔧 环境要求

| 依赖 | 版本 |
|------|------|
| Ubuntu | 22.04（WSL2 或原生） |
| ROS2 | Humble |
| Gazebo Sim | Fortress（`ros-humble-ros-gz`） |
| MoveIt2 | ros-humble-moveit |
| Python | 3.10 + torch(CUDA) + ultralytics |

> ⚠️ **WSL2 专用坑**：gz-sim 的 ogre2 渲染引擎在 WSL2 GPU 直通下会崩（`GL3PlusTextureGpu::copyTo` UnimplementedException，[gz-rendering #662](https://github.com/gazebosim/gz-rendering/issues/662)）。需把渲染引擎换成 ogre1（本项目 launch 文件已内置 `--render-engine ogre`）。

## 🚀 快速开始

```bash
# 1. 安装依赖
sudo apt update && sudo apt install -y ros-humble-ros-gz ros-humble-nav2-common

# 2. 拉取 SO-ARM101 官方包
mkdir -p ~/so_arm_ws/src && cd ~/so_arm_ws/src
git clone https://ghproxy.net/https://github.com/JafarAbdi/ros2_so_arm100.git
git clone https://ghproxy.net/https://github.com/JafarAbdi/feetech_ros2_driver.git

# 3. 安装依赖 + 构建
cd ~/so_arm_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install

# 4. 启动单臂仿真
source /opt/ros/humble/setup.bash && source ~/so_arm_ws/install/setup.bash
ros2 launch so_arm_gz so_arm_gz_bringup.launch.py arm_id:=so_arm101 gazebo_gui:=true
```

## 📁 项目组成

| 模块 | 内容 |
|------|------|
| 任务编排 | `parcel_task.py` —— 检测 → 抓取 → 翻转 → OCR → 放置 |
| 视觉定位 | `detect_yolo.py` —— YOLOv8 检测 + RGBD 深度反投影得到 3D 坐标 |
| 数据采集 | `collect_dataset.py` —— 在仿真场景里自动采集训练数据 |
| 仿真资产 | `so_arm101/` —— 场景、快递盒模型、对上游 ROS2 包的必要改动 |
| 面单识别 | `ocr_infer.py`、`gen_waybill.py` |
| 文档 | `docs/` —— 阶段汇报、演讲讲稿、真机迁移计划 |

## 📊 当前进度

- [x] SO-ARM101 单臂在 Gazebo Sim 仿真跑通（模型 + 控制器 + RGBD 相机）
- [x] 解决 WSL2 下 ogre2 渲染崩溃（换 ogre1）
- [ ] YOLOv8 快递盒检测模型训练（标注方案重做中）
- [ ] 抓取 + 翻转动作序列（MoveIt2 规划）
- [ ] 面单 OCR 读单号（待 ≥3 个真实面单素材）
- [ ] 真机迁移（SO-ARM101 + RealSense 深度相机）

## 🗺️ 路线图

1. **单盒单面**：检测 → 抓取 → 翻转一次 → OCR
2. **面单未知面**：面单在 1-2 个未知面 → 翻转 + 重新确认循环
3. **泛化**：多个盒子、堆叠、遮挡
4. **sim2real**：真实 SO-ARM101 + RealSense D435i + 真实面单

## 🙏 致谢

- [ros2_so_arm100](https://github.com/JafarAbdi/ros2_so_arm100)（BSD-3）：SO-ARM101 的 ROS2 描述 + MoveIt 配置 + Gazebo 仿真
- [feetech_ros2_driver](https://github.com/JafarAbdi/feetech_ros2_driver)：舵机硬件驱动
- [ultralytics](https://github.com/ultralytics/ultralytics)：YOLOv8

## 📄 License

本项目代码采用 [MIT License](LICENSE)。依赖的第三方包遵循各自的许可证。
