# Pi0 复现

## Pi0 训练与推理

> [!IMPORTANT]
> 请先把下面命令中的所有 `<...>` 占位符替换成你自己的配置，再运行。

> [!IMPORTANT]
> 如果你使用的是 `lerobot/pi0_base`，那么要注意它默认期望的相机输入名通常是 `base_0_rgb` / `left_wrist_0_rgb` / `right_wrist_0_rgb`。  
> 如果你的数据集相机名称是 `front` / `side`，那么训练和推理时都建议使用同一套 `rename_map`，例如把 `front -> base_0_rgb`、`side -> left_wrist_0_rgb`，同时再配合 `--policy.empty_cameras=1` 补上缺失的第三路相机。

> [!IMPORTANT]
> 你当前这个仓库里的 `src/lerobot/scripts/lerobot_record.py` 已经把 `dataset.rename_map` 继续传给 `make_policy(...)` 了，所以一般不需要再手动补这部分代码。  
> 如果你换到旧版本 LeRobot，再遇到 `feature mismatch`，可以参考下面的兼容修改方法。

### 1. 模型训练

Pi0 在这个项目里更适合作为一个偏“大模型路线”的 Vision-Language-Action baseline。相比smolvla模型规模更大，相同配置下推理延迟也更高。

Pi0 更推荐从官方基础模型 `lerobot/pi0_base` 继续微调，而不是从零开始训练。

训练前先安装依赖：

```bash
pip install -e ".[pi]"
```

然后登录 wandb：

```bash
wandb login
```

训练命令如下：

```bash
lerobot-train \
  --policy.path=lerobot/pi0_base \
  --dataset.repo_id=<YOUR_DATASET_REPO_ID> \
  --output_dir=outputs/train/<YOUR_PI0_OUTPUT_DIR> \
  --job_name=<YOUR_PI0_JOB_NAME> \
  --policy.device=cuda \
  --policy.dtype=bfloat16 \
  --policy.gradient_checkpointing=true \
  --policy.compile_model=false \
  --policy.train_expert_only=true \
  --batch_size=4 \
  --steps=20000 \
  --wandb.enable=true \
  --rename_map='{"observation.images.front": "observation.images.base_0_rgb", "observation.images.side": "observation.images.left_wrist_0_rgb"}' \
  --policy.empty_cameras=1 \
  --policy.push_to_hub=false
```

如果你的数据集本来就是三路相机，并且名字已经和 `pi0_base` 一致，例如：

```text
observation.images.base_0_rgb
observation.images.left_wrist_0_rgb
observation.images.right_wrist_0_rgb
```

那么就可以不加 `rename_map`，也不需要 `empty_cameras`。

如果显存足够，想做更完整的微调，可以把：

```bash
--policy.train_expert_only=true
```

改成：

```bash
--policy.train_expert_only=false
```

### 2. 训练参数说明

- `<YOUR_DATASET_REPO_ID>`
  - 训练所使用的数据集。
  - 例如：`your_hf_username/your_dataset_name`

- `<YOUR_PI0_OUTPUT_DIR>`
  - 本地训练输出目录名称。
  - 例如：`pi0_pick_place_dataset`

- `<YOUR_PI0_JOB_NAME>`
  - 当前训练任务名称。
  - 建议与输出目录保持一致。

- `--policy.path=lerobot/pi0_base`
  - 使用官方提供的 Pi0 基础模型继续微调。
  - 一般不建议从零开始训练。

- `--policy.dtype=bfloat16`
  - 可以明显降低显存压力。
  - 如果你的硬件环境对 `bfloat16` 支持不好，再考虑改回 `float32`。

- `--policy.gradient_checkpointing=true`
  - 用来进一步降低训练显存占用。
  - 代价是训练速度会更慢一些。

- `--policy.compile_model=false`
  - 第一次复现时建议先关掉。
  - 这样更容易排查问题，也能避免编译带来的额外启动开销。

- `--policy.train_expert_only=true`
  - 只训练动作专家和相关投影层，不全量更新 VLM 主体。
  - 这通常是单卡先跑通 Pi0 的更稳妥做法。

- `--batch_size=4`
  - 这是一个比较保守的起步值。
  - 如果仍然 OOM，可以先降到 `2` 甚至 `1`。

- `--steps=30000`
  - 作为第一轮验证用的训练步数。
  - 如果效果还不稳定，可以继续增加。

- `--rename_map='{"observation.images.front": "observation.images.base_0_rgb", "observation.images.side": "observation.images.left_wrist_0_rgb"}'`
  - 如果你的数据集相机名是 `front` / `side`，而 `pi0_base` 期望的是 `base_0_rgb` / `left_wrist_0_rgb` / `right_wrist_0_rgb`，就需要加这个映射。
  - 训练和推理阶段建议保持同一套映射，不要训练时用一套、推理时再换另一套名字。

- `--policy.empty_cameras=1`
  - `pi0_base` 默认按 3 路相机组织输入。
  - 如果你当前只有两路相机，可以补 1 路空相机，对齐缺失的 `right_wrist_0_rgb`。

### 3. 训练结果保存位置

训练完成后，模型一般会保存在下面这个目录：

```bash
outputs/train/<YOUR_PI0_OUTPUT_DIR>/checkpoints/last/pretrained_model
```

如果训练中断，也可以从已有 checkpoint 继续恢复训练：

```bash
lerobot-train \
  --config_path=outputs/train/<YOUR_PI0_OUTPUT_DIR>/checkpoints/last/pretrained_model/train_config.json \
  --resume=true
```

### 4. 上传模型到 Hugging Face

如果本地训练确认没有问题，可以把训练好的模型上传到 Hugging Face，方便后续推理直接加载。

```bash
huggingface-cli upload <YOUR_MODEL_REPO_ID> \
  outputs/train/<YOUR_PI0_OUTPUT_DIR>/checkpoints/last/pretrained_model
```

上传完成后，就可以通过下面这种形式加载模型：

```text
<YOUR_MODEL_REPO_ID>
```

例如：

```text
your_hf_username/your_pi0_model_name
```

### 4.1 `lerobot_record.py` 中的 `rename_map` 兼容说明

当前这个仓库中的 `src/lerobot/scripts/lerobot_record.py` 已经是兼容写法，一般不需要额外修改。

如果你在别的旧版本 LeRobot 里看到的是下面这行：

```python
policy = None if cfg.policy is None else make_policy(cfg.policy, ds_meta=dataset.meta)
```

那么建议改成：

```python
policy = None if cfg.policy is None else make_policy(
    cfg.policy,
    ds_meta=dataset.meta,
    rename_map=cfg.dataset.rename_map if cfg.dataset.rename_map else None,
)
```

修改文件位置：

```text
src/lerobot/scripts/lerobot_record.py
```

这样做的作用是：在推理阶段加载策略时，也允许 `front -> base_0_rgb`、`side -> left_wrist_0_rgb` 这种映射继续生效。

### 5. 真机推理与评估

Pi0 可以像 SmolVLA 一样选择同步推理或异步推理，这里不再赘述。

因为配置原因我无法本地部署pi0,所以只能在云端，在autodl租卡进行推理，本地只进行图像采集和动作执行，但是动作执行非常慢，同时无法开启双相机，如果有条件的小伙伴可以尝试一下云端推理本地执行能否成功。

以下是完整命令

在服务器上运行

```bash
python -m lerobot.async_inference.policy_server \
  --host=127.0.0.1 \
  --port=18080 \
  --fps=10 \
  --inference_latency=0.3
```
本地连接服务器
```bash
ssh -p 48772 -N -L 18080:127.0.0.1:18080 root@connect.westc.seetacloud.com   #服务器根据自己的ip和名称修改
```

本地推理
```bash
python -m lerobot.async_inference.robot_client \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM0 \
  --robot.id=my_awesome_follower_arm \
  --robot.cameras='{ base_0_rgb: {type: opencv, index_or_path: "/dev/v4l/by-path/pci-0000:08:00.4-usb-0:2.1:1.0-video-index0", width: 640, height: 480, fps: 30, fourcc: "MJPG"}  }' \
  --server_address=127.0.0.1:18080 \
  --policy_type=pi0 \
  --pretrained_name_or_path=/root/autodl-tmp/lerobot/models/pi0_lerobot_grasp_dataset \
  --policy_device=cuda \
  --client_device=cpu \
  --actions_per_chunk=50 \
  --chunk_size_threshold=0.6 \            #在动作队列中剩余多少比例开始推理
  --fps=30 \
  --task="Grab the rectangular box and place it into the paper box"
```
如果双相机出现问题可以先尝试单相机，不过还是建议局域网下云端推理或者直接本地推理。

### 6. Pi0 常见问题

#### 1. 训练时很容易 OOM，batch size 拉不上去。

解决方法：Pi0 本身就是比较重的 VLA 模型，训练显存压力明显大于 ACT 和 SmolVLA。第一轮复现建议优先使用下面这组配置：

```bash
--policy.dtype=bfloat16
--policy.gradient_checkpointing=true
--policy.train_expert_only=true
--batch_size=4
```

如果还是 OOM，就继续把 `batch_size` 降到 `2` 或 `1`。


#### 2. 加载模型时提示 `google/paligemma-3b-pt-224` 下载失败，或者 tokenizer 找不到。

解决方法：Pi0 的预处理器里会用到 `PaliGemma` tokenizer。如果你的环境不能直接访问 Hugging Face，推理时就可能在加载 `policy_preprocessor.json` 的 tokenizer 配置时失败。
同时该模型下载前需要先到其huggingface网页下进行授权，才能正常下载。

