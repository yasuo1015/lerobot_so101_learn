# LeRobot + SO101 快速入门与策略复现中遇到的问题

## 目录

- [项目定位](#项目定位)
- [硬件与环境](#硬件与环境)
- [快速开始](#快速开始)
- [ACT 复现](docs/act.md)
- [Diffusion 复现](docs/diffusion.md)
- [SmolVLA 复现](docs/smolvla.md)
- [Pi0 复现](docs/pi0.md)
- [实验结果汇总](#实验结果汇总)
- [参考资料](#参考资料)

## 项目定位

### 仓库目标

- 记录 LeRobot 真机部署过程中的问题、思考与解决方法。
- 面向想快速上手 `LeRobot + SO101` 的同学，提供尽量简洁的实操流程。
- 重点总结 `ACT`、`Diffusion`、`SmolVLA`、`Pi0` 四类策略在真机复现中遇到的问题。

## 硬件与环境

### 硬件配置

- 机械臂：`SOARM101`
- 相机配置：双相机，包含场景相机和手眼相机。`{type: opencv, width: 640, height: 480, fps: 30, fourcc: "MJPG"}`
- 本地电脑显卡：`RTX 4060 8G`

### 软件环境

- 系统：`Ubuntu 22.04`
- Python：`3.10.20`
- CUDA：`12.6`
- PyTorch：`2.7.1`
- LeRobot 版本：`v0.4.4`

### 任务设置

- 任务名称：`Grab the rectangular box and place it into the paper box`
- 任务类型：`pick and place`
- 数据采集方式：遥操作采集
- 相机视角：`front + side`

## 快速开始

### 1. 环境安装与基础配置

#### 参考教程

- [同济子豪兄 LeRobot 教程](https://zihao-ai.feishu.cn/wiki/TS6swApHbinx01kHDi5cf5n5n8c)
- [矽递科技 LeRobot 教程](https://wiki.seeedstudio.com/cn/lerobot_so100m_new/)

### 2. 常用命令

#### 查找相机

```bash
lerobot-find-cameras
```

#### 串口权限使能

```bash
sudo chmod 666 /dev/ttyACM*
```

#### 固定相机路径

防止相机连接不稳定导致编号乱跳，建议尽量固定 USB 口位不变。  
两个相机也尽量插在电脑端不同的 USB 接口上。

```text
前视相机：{your_front_camera_path}
侧视相机：{your_side_camera_path}
```

#### 使用双相机进行遥操作

```bash
lerobot-teleoperate \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id={your_follower_name} \
  --robot.cameras='{ front: {type: opencv, index_or_path: "your_front_camera_path", width: 640, height: 480, fps: 30, fourcc: "MJPG"}, side: {type: opencv, index_or_path: "your_side_camera_path", width: 640, height: 480, fps: 30, fourcc: "MJPG"} }' \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id={your_leader_name} \
  --display_data=true
```

需要自行替换的参数：

- `{your_follower_name}`：你的 follower 机械臂名称
- `{your_leader_name}`：你的 leader 机械臂名称
- `your_front_camera_path`：前视相机设备路径
- `your_side_camera_path`：侧视相机设备路径

#### 数据采集

推荐先保存在本地，确认数据正常后再上传到 Hugging Face。

数据采集命令：

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port="${FOLLOWER_PORT}" \
  --robot.id="${FOLLOWER_ID}" \
  --robot.cameras="{ front: {type: opencv, index_or_path: \"${FRONT_CAM}\", width: 640, height: 480, fps: 30, fourcc: \"MJPG\"}, side: {type: opencv, index_or_path: \"${SIDE_CAM}\", width: 640, height: 480, fps: 30, fourcc: \"MJPG\"} }" \
  --teleop.type=so101_leader \
  --teleop.port="${LEADER_PORT}" \
  --teleop.id="${LEADER_ID}" \
  --display_data=true \
  --dataset.repo_id="${DATASET_REPO_ID}" \
  --dataset.num_episodes="${NUM_EPISODES}" \
  --dataset.single_task="${TASK_NAME}" \
  --dataset.push_to_hub=false \
  --dataset.episode_time_s="${EPISODE_TIME}" \
  --dataset.reset_time_s="${RESET_TIME}" \
  --resume=true
```

参数说明：

- `FOLLOWER_PORT`：follower 机械臂串口
- `LEADER_PORT`：leader 机械臂串口
- `FOLLOWER_ID`：follower 机械臂名称
- `LEADER_ID`：leader 机械臂名称
- `FRONT_CAM`：前视相机设备路径
- `SIDE_CAM`：侧视相机设备路径
- `DATASET_REPO_ID`：数据集仓库名，建议写成 `你的 Hugging Face 用户名/数据集名`
- `TASK_NAME`：任务描述，训练和评估时建议保持一致
- `NUM_EPISODES`：采集回合数
- `EPISODE_TIME`：每回合录制时间，单位秒
- `RESET_TIME`：每回合之间的重置时间，单位秒
- `--resume=true`：如果已有同名本地数据，则继续追加采集；如果想重新开始，改成 `false`

#### 上传到 Hugging Face

先登录：

```bash
hf auth login
```

再上传数据集：

```bash
python - <<'PY'
from lerobot.datasets.lerobot_dataset import LeRobotDataset

local_path = "your_local_dataset_path"
target_repo = "your_hf_username/your_dataset_name"

ds = LeRobotDataset(
    repo_id=target_repo,
    root=local_path,
    download_videos=False,
)

ds.finalize()
ds.push_to_hub(private=False)

print(f"Uploaded to: https://huggingface.co/datasets/{target_repo}")
PY
```

参数说明：

- `your_local_dataset_path`：本地保存的数据集路径
- `your_hf_username/your_dataset_name`：目标 Hugging Face 数据集仓库名



