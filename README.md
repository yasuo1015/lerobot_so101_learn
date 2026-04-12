# LeRobot + SO101 快速入门与策略复现中遇到的问题。
## 目录

- [项目定位](#项目定位)
- [硬件与环境](#硬件与环境)
- [快速开始](#快速开始)
- [ACT 复现](#act-复现)
- [Diffusion 复现](#diffusion-复现)
- [SmolVLA 复现](#smolvla-复现)
- [Pi0 复现](#pi0-复现)
- [实验结果汇总](#实验结果汇总)
- [参考资料](#参考资料)
## 项目定位
- lerobot真机部署过程中的问题，思考与解决。
## 硬件与环境
- 机械臂：`SOARM101`
- 相机：双相机，包含场景相机和手眼相机（设置均为{type: opencv, width: 640, height: 480, fps: 30, fourcc: "MJPG"}）
- 系统：`Ubuntu 22.04`
- Python：`3.10.20`
- CUDA：`12.6`
- PyTorch：`2.7.1`
- LeRobot 版本：`v0.4.4`
- 本地电脑显卡：4060 8G
### 任务设置
- 任务名称：`Grab the rectangular box and place it into the paper box`
- 任务类型：pick and place
- 数据采集方式：遥操作采集
- 相机视角：`front + side`
## 快速开始
### 1.环境安装及基础配置
具体操作详见
- https://zihao-ai.feishu.cn/wiki/TS6swApHbinx01kHDi5cf5n5n8c 同济子豪兄lerobot教程
- https://wiki.seeedstudio.com/cn/lerobot_so100m_new/  矽递科技lerobot教程
### 2.常用命令
- 查找相机：
```bash
lerobot-find-cameras
- 串口权限使能：
```bash
sudo chmod 666 /dev/ttyACM*
- 固定相机路径（保持 USB 口位不变）：
防止相机连接不稳定编号乱跳，可考虑固定住。（注意相机连线尽量插在电脑端的两个不同USB接口上）
```bash
前视：{your_path}
侧视：{your_path}
- 使用双相机遥操作
```bash
lerobot-teleoperate \
    --robot.type=so101_follower \
    --robot.port=/dev/ttyACM0 \
    --robot.id={your foller name}\
    --robot.cameras='{ front: {type: opencv, index_or_path: "your_path", width: 640, height: 480, fps: 30, fourcc: "MJPG"}, side: {type: opencv, index_or_path: "your_path", width: 640, height: 480, fps: 30, fourcc: "MJPG"} }' \
    --teleop.type=so101_leader \
    --teleop.port=/dev/ttyACM1 \
    --teleop.id={your leader name} \
    --display_data=true
- 数据采集


