# SmolVLA 在 IsaacLab Arena 中的评估启动逻辑

本文记录以下命令在 LeRobot 中的实际执行链路，以及 SmolVLA 模型如何接入 IsaacLab Arena 环境。

```bash
lerobot-eval \
    --policy.path=nvidia/smolvla-arena-gr1-microwave \
    --env.type=isaaclab_arena \
    --env.hub_path=nvidia/isaaclab-arena-envs \
    --rename_map='{"observation.images.robot_pov_cam_rgb": "observation.images.robot_pov_cam"}' \
    --policy.device=cuda \
    --env.environment=gr1_microwave \
    --env.embodiment=gr1_pink \
    --env.object=mustard_bottle \
    --env.headless=false \
    --env.enable_cameras=true \
    --env.video=true \
    --env.video_length=10 \
    --env.video_interval=15 \
    --env.state_keys=robot_joint_pos \
    --env.camera_keys=robot_pov_cam_rgb \
    --trust_remote_code=True \
    --eval.batch_size=1
```

## 1. CLI 入口

- `lerobot-eval` 的入口定义在 [`pyproject.toml#L204`](../../pyproject.toml#L204)，指向 `lerobot.scripts.lerobot_eval:main`。
- 主流程从 `eval_main(cfg)` 开始，见 [`src/lerobot/scripts/lerobot_eval.py#L506`](../../src/lerobot/scripts/lerobot_eval.py#L506)。

## 2. 配置阶段：先解析 policy.path

- `EvalPipelineConfig.__post_init__` 会读取 `--policy.path`。
- 通过 `PreTrainedConfig.from_pretrained(...)` 从本地目录或 Hugging Face Hub 加载策略配置 `config.json`，并生成具体策略配置类型（这里是 `SmolVLAConfig`）。
- 对应代码：
- [`src/lerobot/configs/eval.py#L44`](../../src/lerobot/configs/eval.py#L44)（`EvalPipelineConfig.__post_init__`）
- [`src/lerobot/configs/policies.py#L168`](../../src/lerobot/configs/policies.py#L168)（`PreTrainedConfig.from_pretrained`）

## 3. 环境加载：isaaclab_arena 通过 Hub 动态导入

- `eval_main` 里先执行 `make_env(...)` 创建环境。
- 因为 `isaaclab_arena` 继承 `HubEnvConfig`，并带 `hub_path=nvidia/isaaclab-arena-envs`，所以会走 Hub 远程环境分支：
- 检查 `trust_remote_code`。
- 下载 hub 仓库中的 `env.py`（或指定 path）。
- 动态 import 该模块并调用其中的 `make_env(n_envs, use_async_envs, cfg)`。
- 对应代码：
- [`src/lerobot/envs/configs.py#L72`](../../src/lerobot/envs/configs.py#L72)（`HubEnvConfig`）
- [`src/lerobot/envs/factory.py#L102`](../../src/lerobot/envs/factory.py#L102)（`make_env`）
- [`src/lerobot/envs/factory.py#L137`](../../src/lerobot/envs/factory.py#L137)（`HubEnvConfig` 分支）
- [`src/lerobot/envs/factory.py#L145`](../../src/lerobot/envs/factory.py#L145)（下载/导入 hub env）

## 4. 策略加载：SmolVLA 权重如何进入评估流程

- `eval_main` 再执行 `make_policy(cfg=cfg.policy, env_cfg=cfg.env, rename_map=cfg.rename_map)`。
- `make_policy` 会根据 `cfg.type` 解析到 `SmolVLAPolicy`。
- 因为有 `cfg.pretrained_path`，会调用 `SmolVLAPolicy.from_pretrained(...)`。
- `PreTrainedPolicy.from_pretrained` 会下载/读取 `model.safetensors` 并加载到模型，然后 `policy.to(device)` 和 `policy.eval()`。
- 对应代码：
- [`src/lerobot/policies/factory.py#L406`](../../src/lerobot/policies/factory.py#L406)（`make_policy`）
- [`src/lerobot/policies/factory.py#L488`](../../src/lerobot/policies/factory.py#L488)（pretrained 分支）
- [`src/lerobot/policies/pretrained.py#L76`](../../src/lerobot/policies/pretrained.py#L76)（`from_pretrained`）
- [`src/lerobot/policies/smolvla/modeling_smolvla.py#L225`](../../src/lerobot/policies/smolvla/modeling_smolvla.py#L225)（`SmolVLAPolicy`）

## 5. 关键点：模型不是“塞进环境对象”，而是在 rollout 中逐步交互

在 `rollout()` 中，每一步都是：

1. `env.reset/step` 产出原始观测。
2. `preprocess_observation` 把 IsaacLab 观测字段标准化（例如 `policy`、`camera_obs`）。
3. `add_envs_task` 注入任务文本（`task`）。
4. `env_preprocessor`（`IsaaclabArenaProcessorStep`）按 `state_keys`/`camera_keys` 抽取状态与图像，得到 `observation.state` 和 `observation.images.*`。
5. `preprocessor`（策略预处理）执行重命名、tokenize、归一化、搬运到 device。
6. `policy.select_action(observation)` 推理动作。
7. `postprocessor` 反归一化动作并转到 CPU。
8. `env.step(action)` 执行动作进入下一步。

对应代码：

- 主循环：[`src/lerobot/scripts/lerobot_eval.py#L95`](../../src/lerobot/scripts/lerobot_eval.py#L95)（`rollout`）
- rollout 中关键调用：[`src/lerobot/scripts/lerobot_eval.py#L164`](../../src/lerobot/scripts/lerobot_eval.py#L164)（`preprocess_observation`）、[`src/lerobot/scripts/lerobot_eval.py#L170`](../../src/lerobot/scripts/lerobot_eval.py#L170)（`add_envs_task`）、[`src/lerobot/scripts/lerobot_eval.py#L173`](../../src/lerobot/scripts/lerobot_eval.py#L173)（`env_preprocessor`）、[`src/lerobot/scripts/lerobot_eval.py#L175`](../../src/lerobot/scripts/lerobot_eval.py#L175)（policy preprocessor）、[`src/lerobot/scripts/lerobot_eval.py#L177`](../../src/lerobot/scripts/lerobot_eval.py#L177)（`policy.select_action`）、[`src/lerobot/scripts/lerobot_eval.py#L189`](../../src/lerobot/scripts/lerobot_eval.py#L189)（`env.step`）
- 观测标准化：[`src/lerobot/envs/utils.py#L49`](../../src/lerobot/envs/utils.py#L49)
- 任务注入：[`src/lerobot/envs/utils.py#L156`](../../src/lerobot/envs/utils.py#L156)
- IsaacLab 环境预处理：[`src/lerobot/processor/env_processor.py#L158`](../../src/lerobot/processor/env_processor.py#L158)
- SmolVLA 预后处理构建：[`src/lerobot/policies/smolvla/processor_smolvla.py#L39`](../../src/lerobot/policies/smolvla/processor_smolvla.py#L39)

## 6. rename_map 在这条命令中的作用

- 你的映射：
- `observation.images.robot_pov_cam_rgb -> observation.images.robot_pov_cam`
- 生效位置是策略预处理 pipeline 里的 `RenameObservationsProcessorStep`。
- 目的：把环境输出键名改为模型 checkpoint 期望的输入键名。
- 对应代码：
- [`src/lerobot/processor/rename_processor.py#L27`](../../src/lerobot/processor/rename_processor.py#L27)（`RenameObservationsProcessorStep`）
- [`src/lerobot/processor/rename_processor.py#L46`](../../src/lerobot/processor/rename_processor.py#L46)（重命名实际生效行）
- [`src/lerobot/scripts/lerobot_eval.py#L539`](../../src/lerobot/scripts/lerobot_eval.py#L539)（将 `rename_map` 注入 preprocessor overrides）
