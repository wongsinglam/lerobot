# 使用 LeRobot 的 PI0.5 训练自有机器人（数据已转 LeRobot 格式）

本文整理了使用 `pi05`（PI0.5）训练自有机器人时，主要需要配置的部分。每个部分都给出对应的 `config` 项和代码位置。

## 1) 训练入口与基础运行参数

- `config`：
  - `dataset.repo_id` / `dataset.root`
  - `policy.type=pi05`
  - `policy.pretrained_path`
  - `steps` / `batch_size` / `num_workers`
  - `output_dir` / `job_name`
- `代码`：
  - [`src/lerobot/configs/train.py#L37`](../../src/lerobot/configs/train.py#L37)（`TrainPipelineConfig`）
  - [`src/lerobot/configs/default.py#L24`](../../src/lerobot/configs/default.py#L24)（`DatasetConfig`）
  - [`src/lerobot/scripts/lerobot_train.py#L152`](../../src/lerobot/scripts/lerobot_train.py#L152)（训练主入口 `train`）

## 2) 策略与数据特征对齐（关键）

- `config`：
  - `rename_map`（观测键名不一致时使用）
  - `policy.max_state_dim`
  - `policy.max_action_dim`
  - `policy.empty_cameras`
- `代码`：
  - [`src/lerobot/policies/factory.py#L406`](../../src/lerobot/policies/factory.py#L406)（`make_policy`，特征对齐主逻辑）
  - [`src/lerobot/datasets/utils.py#L707`](../../src/lerobot/datasets/utils.py#L707)（`dataset_to_policy_features`）
  - [`src/lerobot/policies/pi05/configuration_pi05.py#L118`](../../src/lerobot/policies/pi05/configuration_pi05.py#L118)（`validate_features`）
  - [`src/lerobot/policies/utils.py#L226`](../../src/lerobot/policies/utils.py#L226)（视觉特征一致性校验）
- 说明：
  - `max_action_dim` 需要大于等于你数据真实 action 维度，否则在模型投影层会报维度错误。

## 3) PI0.5 输入必需项（state + task）

- `config`：
  - 无特殊开关，但数据侧必须具备 `observation.state` 与任务文本（`task`）。
- `代码`：
  - [`src/lerobot/policies/pi05/processor_pi05.py#L61`](../../src/lerobot/policies/pi05/processor_pi05.py#L61)（`state`/`task` 必需校验）
  - [`src/lerobot/datasets/lerobot_dataset.py#L1051`](../../src/lerobot/datasets/lerobot_dataset.py#L1051)（`__getitem__`）与 [`src/lerobot/datasets/lerobot_dataset.py#L1078`](../../src/lerobot/datasets/lerobot_dataset.py#L1078)（`task` 注入）

## 4) 归一化与统计（PI0.5 默认 quantiles）

- `config`：
  - `policy.normalization_mapping`
  - PI0.5 默认是 `STATE/ACTION -> QUANTILES`，视觉 `VISUAL -> IDENTITY`
  - 若数据没有 quantile 统计，可改为 `MEAN_STD` 或先补 quantile 统计
- `代码`：
  - [`src/lerobot/policies/pi05/configuration_pi05.py#L66`](../../src/lerobot/policies/pi05/configuration_pi05.py#L66)（默认 `normalization_mapping`）
  - [`src/lerobot/policies/pi05/processor_pi05.py#L136`](../../src/lerobot/policies/pi05/processor_pi05.py#L136)（`NormalizerProcessorStep` 在 tokenizer 前）
  - [`src/lerobot/datasets/v30/augment_dataset_quantile_stats.py#L182`](../../src/lerobot/datasets/v30/augment_dataset_quantile_stats.py#L182)（补 quantile 统计入口）

## 5) PI0.5 微调策略（显存/速度相关）

- `config`：
  - `policy.dtype`（`float32`/`bfloat16`）
  - `policy.device`
  - `policy.gradient_checkpointing`
  - `policy.compile_model` / `policy.compile_mode`
  - `policy.freeze_vision_encoder`
  - `policy.train_expert_only`
- `代码`：
  - [`src/lerobot/policies/pi05/configuration_pi05.py#L75`](../../src/lerobot/policies/pi05/configuration_pi05.py#L75)（dtype/checkpoint/compile/freeze 开关）
  - [`src/lerobot/policies/pi05/modeling_pi05.py#L413`](../../src/lerobot/policies/pi05/modeling_pi05.py#L413)（冻结逻辑）
  - [`src/lerobot/policies/pi05/modeling_pi05.py#L570`](../../src/lerobot/policies/pi05/modeling_pi05.py#L570)（`torch.compile`）
  - [`src/lerobot/policies/pi05/modeling_pi05.py#L586`](../../src/lerobot/policies/pi05/modeling_pi05.py#L586)（梯度检查点 enable）

## 6) 优化器与学习率计划

- `config`：
  - 默认 `use_policy_training_preset=true`（使用 PI0.5 预设 optimizer/scheduler）
  - 或显式传 `optimizer` / `scheduler`
- `代码`：
  - [`src/lerobot/configs/train.py#L132`](../../src/lerobot/configs/train.py#L132)（policy preset 装配）
  - [`src/lerobot/policies/pi05/configuration_pi05.py#L142`](../../src/lerobot/policies/pi05/configuration_pi05.py#L142)（`get_optimizer_preset`）与 [`src/lerobot/policies/pi05/configuration_pi05.py#L151`](../../src/lerobot/policies/pi05/configuration_pi05.py#L151)（`get_scheduler_preset`）
  - [`src/lerobot/optim/factory.py#L25`](../../src/lerobot/optim/factory.py#L25)（optimizer/scheduler 构建）
  - [`src/lerobot/optim/schedulers.py#L95`](../../src/lerobot/optim/schedulers.py#L95)（短训练步数 auto-scale）

## 7) 保存、恢复、推送

- `config`：
  - `save_checkpoint` / `save_freq` / `resume`
  - `policy.push_to_hub` + `policy.repo_id`
- `代码`：
  - [`src/lerobot/configs/train.py#L138`](../../src/lerobot/configs/train.py#L138)（push 前 `repo_id` 校验）
  - [`src/lerobot/scripts/lerobot_train.py#L441`](../../src/lerobot/scripts/lerobot_train.py#L441)（checkpoint 保存）
  - [`src/lerobot/scripts/lerobot_train.py#L516`](../../src/lerobot/scripts/lerobot_train.py#L516)（训练结束 push）

## 最小可用训练命令模板

```bash
python src/lerobot/scripts/lerobot_train.py \
  --dataset.repo_id=<your_dataset> \
  --dataset.root=<optional_local_root> \
  --policy.type=pi05 \
  --policy.pretrained_path=lerobot/pi05_base \
  --policy.push_to_hub=false \
  --policy.device=cuda \
  --policy.dtype=bfloat16 \
  --policy.max_state_dim=<state_dim_ceil> \
  --policy.max_action_dim=<action_dim_ceil> \
  --policy.gradient_checkpointing=true \
  --policy.compile_model=true \
  --steps=3000 \
  --batch_size=32 \
  --output_dir=./outputs/pi05_training \
  --job_name=pi05_training
```

## 常见检查清单

- 数据是否包含 `observation.state`、`action`、`task`。
- 相机键名是否与预训练模型期望一致；不一致时是否设置了 `rename_map`。
- `policy.max_action_dim` 与数据 action 维度是否匹配（不小于真实维度）。
- quantile 统计是否存在；若不存在，是否已切换 `normalization_mapping` 或补齐统计。
