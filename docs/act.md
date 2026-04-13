# ACT 复现

## ACT 训练与推理

> [!IMPORTANT]
> 请先把下面命令中的所有 `<...>` 占位符替换成你自己的配置，再运行。

### 1. 模型训练

ACT 在这个项目里适合作为第一个真机 baseline。相比更复杂的 VLA 模型，ACT 的训练链路更直接，更适合先验证数据集质量、任务定义和真机控制链路是否跑通。

训练命令如下：

```bash
wandb login
```

```bash
lerobot-train \
  --dataset.repo_id=<YOUR_DATASET_REPO_ID> \
  --policy.type=act \
  --output_dir=outputs/train/<YOUR_ACT_OUTPUT_DIR> \
  --job_name=<YOUR_ACT_JOB_NAME> \
  --policy.device=cuda \
  --wandb.enable=true \
  --steps=20000 \
  --batch_size=8 \
  --policy.push_to_hub=false
```

### 2. 训练参数说明

- `<YOUR_DATASET_REPO_ID>`
  - 训练所使用的数据集。
  - 例如：`your_hf_username/your_dataset_name`

- `<YOUR_ACT_OUTPUT_DIR>`
  - 本地训练输出目录名称。
  - 例如：`act_pick_place_dataset`

- `<YOUR_ACT_JOB_NAME>`
  - 当前训练任务名称。
  - 建议与输出目录保持一致。

- `--steps=20000`
  - 训练在20000步后loss下降就不明显了，使用我的数据集大概跌到0.1左右。这是一个适合起步验证的训练步数。
  - 如果效果不够稳定，可以继续增加。

- `--batch_size=8`
  - 可以根据自己的显存大小调整。
  - 如果显存不足，可以适当减小。

### 3. 训练结果保存位置

训练完成后，模型一般会保存在下面这个目录：

```bash
outputs/train/<YOUR_ACT_OUTPUT_DIR>/checkpoints/last/pretrained_model
```

如果训练中断，也可以从已有 checkpoint 继续恢复训练：

```bash
lerobot-train \
  --config_path=outputs/train/<YOUR_ACT_OUTPUT_DIR>/checkpoints/last/pretrained_model/train_config.json \
  --resume=true
```

### 4. 上传模型到 Hugging Face

如果本地训练确认没有问题，可以把训练好的模型上传到 Hugging Face，方便后续推理直接加载。

```bash
huggingface-cli upload <YOUR_MODEL_REPO_ID> \
  outputs/train/<YOUR_ACT_OUTPUT_DIR>/checkpoints/last/pretrained_model
```

上传完成后，就可以通过下面这种形式加载模型：

```text
<YOUR_MODEL_REPO_ID>
```

例如：

```text
your_hf_username/your_act_model_name
```

### 5. 真机推理与评估

ACT 训练完成后，可以使用 `lerobot-record` 挂载策略进行真机推理和评估。使用以下命令记录十轮推理结果，每轮10s，复位时间6s。（可根据自己实际情况修改）

```bash
lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=<YOUR_FOLLOWER_ID> \                      #你的从动臂ID
  --robot.cameras='{ front: {type: opencv, index_or_path: "<YOUR_FRONT_CAMERA_PATH>", width: 640, height: 480, fps: 30, fourcc: "MJPG"}, side: {type: opencv, index_or_path: "<YOUR_SIDE_CAMERA_PATH>", width: 640, height: 480, fps: 30, fourcc: "MJPG"} }' \           #相机路径
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=<YOUR_LEADER_ID> \                       #你的引导臂ID
  --display_data=true \
  --dataset.repo_id=<YOUR_EVAL_DATASET_REPO_ID> \      #你要保存的评估数据集的路径
  --dataset.num_episodes=10 \                          #重复十轮
  --dataset.single_task="<YOUR_TASK_DESCRIPTION>" \    #你的任务描述
  --dataset.episode_time_s=10 \                        #每轮时间为10s
  --dataset.reset_time_s=6 \                           #每轮回到原位时间6s
  --dataset.reset_max_relative_target=10.0 \
  --dataset.push_to_hub=false \
  --policy.path=<YOUR_MODEL_REPO_ID> \                 #你的模型名称
  --policy.temporal_ensemble_coeff=-0.01 \             #论文中有关时间集成的参数m，决定模型更偏向新预测还是旧预测
  --policy.n_action_steps=1                            #动作执行步数
```

### 6. 推理阶段的问题

#### 1. 要进行多轮重复推理机械臂不会自动复位。

解决方法：使用遥操作进行复位，同时注意限制最大速度防止机械臂过快复位。

```bash
--teleop.type=so101_leader \
--teleop.port=/dev/ttyACM1 \
--teleop.id=<YOUR_LEADER_ID> \
```

使用 `dataset.reset_max_relative_target` 限制速度：

```bash
--dataset.reset_max_relative_target=10.0
```

#### 2. 几乎完全是模仿学习，即使是两个数据点中间的位置也无法抓到。

解决方法：采集数据时可以变更物体的方向，物体同方向的数据尽量距离远一些。

因为 CVAE 编码器其实是拿到多个不同位置的信息来生成中间的信息，数据样本十分接近会导致过拟合，让机械臂只会一味模仿。

#### 3. 时间聚合问题

如果不显式指定 `policy.temporal_ensemble_coeff`，并将 `policy.n_action_steps` 设为 `1` 的话，推理默认是不使用时间集成的，同时也默认预测动作和执行动作均为 `100`。

不使用时间集成机械臂会比较抖，抓取效果也会相对较差，将动作执行改为 `50` 步会对抓取效果有所提升。

若开启时间集成，设定 `policy.temporal_ensemble_coeff` 值：

- 为负值会对当前观测更敏感，响应更快，但是更容易抖动。
- 为正值会更偏向于旧预测，动作更平滑，更不容易抖，对当前观测会反应慢一些。

个人实验下来 `policy.temporal_ensemble_coeff=-0.01` 抓取效果比较好，也并无明显抖动。有关该参数值的设置可详见：

https://github.com/huggingface/lerobot/pull/319

ACT 的优势在于训练和推理链路都相对清晰，适合作为 `LeRobot + SO101` 复现过程中的第一个基线模型。
