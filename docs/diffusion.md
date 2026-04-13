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
  --policy.n_action_steps=16 \
  --policy.drop_n_last_frames=15 \
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

- `--policy.drop_n_last_frames=15`
  - 末尾不进行采样的帧数，避免采样动作序列的时候因为未来动作不够长而出现大量padding，采样动作序列的值就是horizon。
  - 一般通过drop_n_last_frames = horizon - n_action_steps - n_obs_steps + 1 设置


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

Diffusion 训练完成后，可以使用 `lerobot-record` 挂载策略进行真机推理和评估，这里是使用horizon为32进行训练的。使用以下命令记录十轮推理结果，每轮 10s，复位时间 6s。（可根据自己实际情况修改）

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
  --policy.device=cuda \
  --policy.n_action_steps=16 \                         #一次推理后动作执行步数，不能大于预测步数
  --policy.noise_scheduler_type=DDIM \
  --policy.num_inference_steps=10
```

### 6. 推理阶段的问题

#### 1.训练 loss 下降慢，机械臂震动明显。

解决方法：适当增大 `policy.horizon`。horizon值太小重复推理次数过多，机械臂抖动明显而且效果较差。个人实验里，`horizon=32` 的表现通常会明显好于 `horizon=16`，动作窗口更长，真机控制也更平滑。

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

实机测试下来发现使用DDIM及10步反向推理平滑性明显增强，抓取效果变化不大。

#### 3. Diffusion 虽然比 ACT 更擅长处理多模态分布，但如果数据覆盖范围不足，依然会过拟合。

解决方法：采集数据时尽量改变物体方向、摆放位置和抓取轨迹，不要让样本都集中在非常接近的位置上。

因为 Diffusion 的优势在于学习“一个合理动作分布”，如果数据本身只覆盖了很窄的一小块状态空间，那么模型学到的仍然只是局部模仿，泛化不会自动变好。
