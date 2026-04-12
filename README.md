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
### 任务设置
- 任务名称：`Grab the rectangular box and place it into the paper box`
- 任务类型：pick and place
- 数据采集方式：遥操作采集
- 相机视角：`front + side`
