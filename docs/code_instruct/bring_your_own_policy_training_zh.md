# 在 LeRobot 中自建 Policy 并训练（中文实操）

本文基于官方文档 [`docs/source/bring_your_own_policies.mdx`](../source/bring_your_own_policies.mdx) ，补充了按当前仓库代码可直接落地的实现与训练流程。

## 0. 先理解 LeRobot 如何识别你的自定义 policy

LeRobot 不是手动注册每个第三方 policy，而是通过“插件发现 + 命名约定 + 动态导入”完成：

- 插件发现：训练入口会先调用 [`src/lerobot/scripts/lerobot_train.py#L531`](../../src/lerobot/scripts/lerobot_train.py#L531) 的 `register_third_party_plugins()`
- 插件前缀：只会自动导入包名以 `lerobot_policy_` 开头的包，见 [`src/lerobot/utils/import_utils.py#L149`](../../src/lerobot/utils/import_utils.py#L149) 与 [`src/lerobot/utils/import_utils.py#L152`](../../src/lerobot/utils/import_utils.py#L152)
- policy 类动态推断规则：
  - 从 `Config` 类名推断 `Policy` 类名，见 [`src/lerobot/policies/factory.py#L550`](../../src/lerobot/policies/factory.py#L550) 与 [`src/lerobot/policies/factory.py#L556`](../../src/lerobot/policies/factory.py#L556)
  - 从 `configuration_xxx.py` 推断 `modeling_xxx.py`，见 [`src/lerobot/policies/factory.py#L557`](../../src/lerobot/policies/factory.py#L557)
- processor 动态推断规则：
  - 函数名必须是 `make_{policy_type}_pre_post_processors`，见 [`src/lerobot/policies/factory.py#L582`](../../src/lerobot/policies/factory.py#L582)
  - 模块名由 `configuration_xxx.py` 推断 `processor_xxx.py`，见 [`src/lerobot/policies/factory.py#L583`](../../src/lerobot/policies/factory.py#L583)

这几条是最容易踩坑的地方。

## 1. 创建插件包骨架

建议目录：

```bash
lerobot_policy_my_custom_policy/
├── pyproject.toml
└── src/
    └── lerobot_policy_my_custom_policy/
        ├── __init__.py
        ├── configuration_my_custom_policy.py
        ├── modeling_my_custom_policy.py
        └── processor_my_custom_policy.py
```

`pyproject.toml` 最小示例：

```toml
[project]
name = "lerobot_policy_my_custom_policy"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
  "torch",
]
```

## 2. 写配置类（必须继承 PreTrainedConfig）

基类约束见：
- [`src/lerobot/configs/policies.py#L41`](../../src/lerobot/configs/policies.py#L41) `PreTrainedConfig`
- 抽象方法要求：[`src/lerobot/configs/policies.py#L104`](../../src/lerobot/configs/policies.py#L104)、[`src/lerobot/configs/policies.py#L118`](../../src/lerobot/configs/policies.py#L118)

示例（可作为起点）：

```python
from dataclasses import dataclass, field

from lerobot.configs.policies import PreTrainedConfig
from lerobot.configs.types import FeatureType, NormalizationMode, PolicyFeature
from lerobot.optim.optimizers import AdamWConfig
from lerobot.optim.schedulers import CosineDecayWithWarmupSchedulerConfig
from lerobot.utils.constants import ACTION, OBS_STATE


@PreTrainedConfig.register_subclass("my_custom_policy")
@dataclass
class MyCustomPolicyConfig(PreTrainedConfig):
    hidden_dim: int = 256
    chunk_size: int = 1
    n_action_steps: int = 1
    normalization_mapping: dict[str, NormalizationMode] = field(
        default_factory=lambda: {
            "VISUAL": NormalizationMode.IDENTITY,
            "STATE": NormalizationMode.MEAN_STD,
            "ACTION": NormalizationMode.MEAN_STD,
        }
    )

    def validate_features(self) -> None:
        if OBS_STATE not in self.input_features:
            self.input_features[OBS_STATE] = PolicyFeature(type=FeatureType.STATE, shape=(8,))
        if ACTION not in self.output_features:
            self.output_features[ACTION] = PolicyFeature(type=FeatureType.ACTION, shape=(8,))

    def get_optimizer_preset(self):
        return AdamWConfig(lr=1e-4, weight_decay=1e-2, grad_clip_norm=1.0)

    def get_scheduler_preset(self):
        return CosineDecayWithWarmupSchedulerConfig(
            num_warmup_steps=1000,
            num_decay_steps=100000,
            peak_lr=1e-4,
            decay_lr=1e-5,
        )

    @property
    def observation_delta_indices(self):
        return None

    @property
    def action_delta_indices(self):
        return list(range(self.chunk_size))

    @property
    def reward_delta_indices(self):
        return None
```

## 3. 写模型类（必须继承 PreTrainedPolicy）

基类约束见：
- [`src/lerobot/policies/pretrained.py#L45`](../../src/lerobot/policies/pretrained.py#L45) `PreTrainedPolicy`
- 子类必须声明 `config_class` 和 `name`，见 [`src/lerobot/policies/pretrained.py#L63`](../../src/lerobot/policies/pretrained.py#L63)
- 必须实现 `get_optim_params/reset/forward/predict_action_chunk/select_action`，见 [`src/lerobot/policies/pretrained.py#L160`](../../src/lerobot/policies/pretrained.py#L160)

最小可训练示例：

```python
from collections import deque

import torch
from torch import Tensor, nn

from lerobot.policies.pretrained import PreTrainedPolicy
from lerobot.utils.constants import ACTION, OBS_STATE
from .configuration_my_custom_policy import MyCustomPolicyConfig


class MyCustomPolicy(PreTrainedPolicy):
    config_class = MyCustomPolicyConfig
    name = "my_custom_policy"

    def __init__(self, config: MyCustomPolicyConfig, **kwargs):
        super().__init__(config)
        self.config.validate_features()
        in_dim = self.config.input_features[OBS_STATE].shape[0]
        out_dim = self.config.output_features[ACTION].shape[0]
        self.net = nn.Sequential(
            nn.Linear(in_dim, self.config.hidden_dim),
            nn.ReLU(),
            nn.Linear(self.config.hidden_dim, out_dim),
        )
        self._action_queue = deque(maxlen=self.config.n_action_steps)

    def get_optim_params(self):
        return self.parameters()

    def reset(self):
        self._action_queue.clear()

    def forward(self, batch: dict[str, Tensor]):
        pred = self.net(batch[OBS_STATE])
        loss = ((pred - batch[ACTION]) ** 2).mean()
        return loss, {"loss": float(loss.detach().cpu())}

    @torch.no_grad()
    def predict_action_chunk(self, batch: dict[str, Tensor], **kwargs) -> Tensor:
        action = self.net(batch[OBS_STATE])  # (B, A)
        return action[:, None, :]  # (B, T=1, A)

    @torch.no_grad()
    def select_action(self, batch: dict[str, Tensor], **kwargs) -> Tensor:
        if len(self._action_queue) == 0:
            chunk = self.predict_action_chunk(batch)[:, : self.config.n_action_steps]
            self._action_queue.extend(chunk.transpose(0, 1))
        return self._action_queue.popleft()
```

## 4. 写 processor（训练必须）

训练脚本会构建 pre/post processor，见 [`src/lerobot/scripts/lerobot_train.py#L280`](../../src/lerobot/scripts/lerobot_train.py#L280)。

`processor_my_custom_policy.py` 至少要提供：

```python
from typing import Any
import torch

from lerobot.processor import (
    DeviceProcessorStep,
    NormalizerProcessorStep,
    PolicyAction,
    PolicyProcessorPipeline,
    RenameObservationsProcessorStep,
    UnnormalizerProcessorStep,
)
from lerobot.processor.converters import policy_action_to_transition, transition_to_policy_action
from lerobot.utils.constants import POLICY_POSTPROCESSOR_DEFAULT_NAME, POLICY_PREPROCESSOR_DEFAULT_NAME

from .configuration_my_custom_policy import MyCustomPolicyConfig


def make_my_custom_policy_pre_post_processors(
    config: MyCustomPolicyConfig,
    dataset_stats: dict[str, dict[str, torch.Tensor]] | None = None,
) -> tuple[
    PolicyProcessorPipeline[dict[str, Any], dict[str, Any]],
    PolicyProcessorPipeline[PolicyAction, PolicyAction],
]:
    pre = PolicyProcessorPipeline(
        steps=[
            RenameObservationsProcessorStep(rename_map={}),
            DeviceProcessorStep(device=config.device),
            NormalizerProcessorStep(
                features={**config.input_features, **config.output_features},
                norm_map=config.normalization_mapping,
                stats=dataset_stats,
            ),
        ],
        name=POLICY_PREPROCESSOR_DEFAULT_NAME,
    )
    post = PolicyProcessorPipeline(
        steps=[
            UnnormalizerProcessorStep(
                features=config.output_features,
                norm_map=config.normalization_mapping,
                stats=dataset_stats,
            ),
            DeviceProcessorStep(device="cpu"),
        ],
        name=POLICY_POSTPROCESSOR_DEFAULT_NAME,
        to_transition=policy_action_to_transition,
        to_output=transition_to_policy_action,
    )
    return pre, post
```

## 5. 包初始化与导出

`__init__.py` 需要导入配置类、模型类、processor 构建函数，保证插件 import 时能完成注册：

```python
from .configuration_my_custom_policy import MyCustomPolicyConfig
from .modeling_my_custom_policy import MyCustomPolicy
from .processor_my_custom_policy import make_my_custom_policy_pre_post_processors

__all__ = [
    "MyCustomPolicyConfig",
    "MyCustomPolicy",
    "make_my_custom_policy_pre_post_processors",
]
```

## 6. 安装插件并验证是否被发现

```bash
cd lerobot_policy_my_custom_policy
pip install -e .
```

验证：

```bash
python -c "from lerobot.configs.policies import PreTrainedConfig; print('my_custom_policy' in PreTrainedConfig.get_known_choices())"
```

## 7. 启动训练

训练入口：
- [`src/lerobot/scripts/lerobot_train.py#L152`](../../src/lerobot/scripts/lerobot_train.py#L152)
- 训练配置校验逻辑：[`src/lerobot/configs/train.py#L81`](../../src/lerobot/configs/train.py#L81)

示例命令（假设数据已是 LeRobot 格式）：

```bash
lerobot-train \
  --policy.type=my_custom_policy \
  --dataset.repo_id=<your_dataset_repo_id> \
  --batch_size=64 \
  --steps=20000 \
  --policy.device=cuda \
  --output_dir=outputs/train/my_custom_policy \
  --job_name=my_custom_policy_run \
  --policy.push_to_hub=false
```

如果你要从已训练 checkpoint 继续训练，可用 `--policy.path=<checkpoint_or_hub_model>`。对应加载逻辑见 [`src/lerobot/configs/train.py#L83`](../../src/lerobot/configs/train.py#L83) 与 [`src/lerobot/configs/policies.py#L168`](../../src/lerobot/configs/policies.py#L168)。

## 8. 常见报错排查

1. 报 `Policy type 'my_custom_policy' is not available`
- 检查包名是否以 `lerobot_policy_` 开头（[`src/lerobot/utils/import_utils.py#L152`](../../src/lerobot/utils/import_utils.py#L152)）。
- 检查是否真的安装了插件 (`pip show lerobot_policy_my_custom_policy`)。
- 检查配置类是否注册了 `@PreTrainedConfig.register_subclass("my_custom_policy")`。

2. 报找不到 policy 类或导入失败
- 配置类名必须以 `Config` 结尾，见 [`src/lerobot/policies/factory.py#L552`](../../src/lerobot/policies/factory.py#L552)。
- 模型文件名要和 `configuration_*.py` 对应，见 [`src/lerobot/policies/factory.py#L557`](../../src/lerobot/policies/factory.py#L557)。

3. 报找不到 processor 函数
- 函数名必须是 `make_{policy_type}_pre_post_processors`，见 [`src/lerobot/policies/factory.py#L582`](../../src/lerobot/policies/factory.py#L582)。

4. 训练时提示 `Policy is not configured`
- 你必须传 `--policy.type` 或 `--policy.path`，见 [`src/lerobot/configs/train.py#L108`](../../src/lerobot/configs/train.py#L108)。

## 9. 推荐最小迭代顺序

1. 先做“仅 state -> action”的最小 MLP policy，跑通 100~500 steps。
2. 再加图像输入和更复杂 processor。
3. 最后再加 chunking、多模态 tokenizer、PEFT 等高级功能。

## Examples and Community Contributions

Check out these example policy implementations:

- [DiTFlow Policy](https://github.com/danielsanjosepro/lerobot_policy_ditflow) - Diffusion Transformer policy with flow-matching objective. Try it out in this example: [DiTFlow Example](https://github.com/danielsanjosepro/test_lerobot_policy_ditflow)

Share your policy implementations with the community! 🤗