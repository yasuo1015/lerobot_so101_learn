# Diffusion 复现

## Diffusion 训练与推理

> [!IMPORTANT]
> 请先把下面命令中的所有 `<...>` 占位符替换成你自己的配置，再运行。

### 1. 模型训练

Diffusion 在这个项目里适合作为 ACT 之后的第二个 imitation learning baseline。相比 ACT，Diffusion 更擅长建模多模态动作分布，但训练和推理开销也更大。

训练命令如下：

```bash
wandb login
```

```bash
lerobot-train \
  --dataset.repo_id=<YOUR_DATASET_REPO_ID> \
  --policy.type=diffusion \
  --output_dir=outputs/train/<YOUR_DIFFUSION_OUTPUT_DIR> \
  --job_name=<YOUR_DIFFUSION_JOB_NAME> \
  --policy.device=cuda \
  --wandb.enable=true \
  --steps=50000 \
  --policy.horizon=32 \
  --policy.push_to_hub=false
```

### 2. 训练参数说明

- `<YOUR_DATASET_REPO_ID>`
  - 训练所使用的数据集。
  - 例如：`your_hf_username/your_dataset_name`

- `<YOUR_DIFFUSION_OUTPUT_DIR>`
  - 本地训练输出目录名称。
  - 例如：`diffusion_pick_place_dataset`

- `<YOUR_DIFFUSION_JOB_NAME>`
  - 当前训练任务名称。
  - 建议与输出目录保持一致。

- `--steps=50000`
  - 作为起步验证用的训练步数。
  - Diffusion 通常比 ACT 更慢收敛，如果效果不够稳定，可以继续增加。

- `--policy.horizon=32`
  - 这里给出的是一个适合真机起步验证的经验值。
  - `horizon` 太小会让动作规划窗口偏短，训练时 loss 下降可能更慢，推理时也更容易抖动。

### 3. 训练结果保存位置

训练完成后，模型一般会保存在下面这个目录：

```bash
outputs/train/<YOUR_DIFFUSION_OUTPUT_DIR>/checkpoints/last/pretrained_model
```

如果训练中断，也可以从已有 checkpoint 继续恢复训练：

```bash
lerobot-train \
  --config_path=outputs/train/<YOUR_DIFFUSION_OUTPUT_DIR>/checkpoints/last/pretrained_model/train_config.json \
  --resume=true
```

### 4. 上传模型到 Hugging Face

如果本地训练确认没有问题，可以把训练好的模型上传到 Hugging Face，方便后续推理直接加载。

```bash
huggingface-cli upload <YOUR_MODEL_REPO_ID> \
  outputs/train/<YOUR_DIFFUSION_OUTPUT_DIR>/checkpoints/last/pretrained_model
```

上传完成后，就可以通过下面这种形式加载模型：

```text
<YOUR_MODEL_REPO_ID>
```

例如：

```text
your_hf_username/your_diffusion_model_name
```

### 5. 真机推理与评估

Diffusion 训练完成后，可以使用 `lerobot-record` 挂载策略进行真机推理和评估。使用以下命令记录十轮推理结果，每轮 10s，复位时间 6s。（可根据自己实际情况修改）

```bash
lerobot-record \
  --robot.type=so101_follower \     #从动臂
  --robot.port=/dev/ttyACM0 \
  --robot.id=<YOUR_FOLLOWER_ID> \
  --robot.cameras='{ front: {type: opencv, index_or_path: "<YOUR_FRONT_CAMERA_PATH>", width: 640, height: 480, fps: 30, fourcc: "MJPG"}, side: {type: opencv, index_or_path: "<YOUR_SIDE_CAMERA_PATH>", width: 640, height: 480, fps: 30, fourcc: "MJPG"} }' \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=<YOUR_LEADER_ID> \
  --display_data=true \
  --dataset.repo_id=<YOUR_EVAL_DATASET_REPO_ID> \
  --dataset.num_episodes=10 \
  --dataset.single_task="<YOUR_TASK_DESCRIPTION>" \
  --dataset.episode_time_s=10 \
  --dataset.reset_time_s=6 \
  --dataset.reset_max_relative_target=10.0 \
  --dataset.push_to_hub=false \
  --policy.path=<YOUR_MODEL_REPO_ID> \
  --policy.device=cuda \
  --policy.n_action_steps=16 \
  --policy.noise_scheduler_type=DDIM \
  --policy.num_inference_steps=10
```

如果模型还没有上传到 Hugging Face，也可以直接把 `--policy.path` 改成本地路径：

```bash
outputs/train/<YOUR_DIFFUSION_OUTPUT_DIR>/checkpoints/last/pretrained_model
```

### 6. 这些占位符需要替换

- `<YOUR_DATASET_REPO_ID>`
  - 训练数据集仓库名
  - 例如：`your_hf_username/your_dataset_name`

- `<YOUR_DIFFUSION_OUTPUT_DIR>`
  - Diffusion 训练输出目录名

- `<YOUR_DIFFUSION_JOB_NAME>`
  - Diffusion 训练任务名

- `<YOUR_MODEL_REPO_ID>`
  - 训练完成后上传的模型仓库名
  - 例如：`your_hf_username/your_diffusion_model_name`

- `<YOUR_FOLLOWER_ID>`
  - follower 机械臂名称

- `<YOUR_FRONT_CAMERA_PATH>`
  - 前视相机路径

- `<YOUR_SIDE_CAMERA_PATH>`
  - 侧视相机路径

- `<YOUR_LEADER_ID>`
  - leader 机械臂名称

- `<YOUR_EVAL_DATASET_REPO_ID>`
  - 推理评估时保存结果的数据集名

- `<YOUR_TASK_DESCRIPTION>`
  - 当前任务描述
  - 建议与训练数据中的任务描述保持一致

### 7. 推理阶段的问题

#### 1.训练 loss 下降慢，机械臂震动明显。

解决方法：适当增大 `policy.horizon`。个人实验里，`horizon=32` 的表现通常会明显好于 `horizon=16`，动作窗口更长，真机控制也更平滑。

对应地，记得同步调整：

```bash
--policy.horizon=32 \
--policy.n_action_steps=16
```

#### 2. 推理阶段机械臂先快速运动一段，然后停顿一下，然后接着运动。

解决方法：模型默认是使用DDPM进行推理，为了加快模型推理速度，可以使用DDIM方法，并将反向推理步骤改写为10步。


```bash
--policy.noise_scheduler_type=DDIM \
--policy.num_inference_steps=10
```

一般来说：

- `num_inference_steps` 更大
  - 动作质量通常更好，但推理更慢。

- `num_inference_steps` 更小
  - 推理更快，但动作可能更粗糙，抓取稳定性也可能下降。

#### 3. Diffusion 虽然比 ACT 更擅长处理多模态分布，但如果数据覆盖范围不足，依然会过拟合。

解决方法：采集数据时尽量改变物体方向、摆放位置和抓取轨迹，不要让样本都集中在非常接近的位置上。

因为 Diffusion 的优势在于学习“一个合理动作分布”，如果数据本身只覆盖了很窄的一小块状态空间，那么模型学到的仍然只是局部模仿，泛化不会自动变好。
