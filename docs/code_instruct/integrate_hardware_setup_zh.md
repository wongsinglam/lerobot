# 在 LeRobot 中集成自定义硬件（Robot/Teleoperator/Camera）实操指引

本文基于官方文档 [`docs/source/integrate_hardware.mdx`](../source/integrate_hardware.mdx)，补充当前仓库里可直接定位的代码路径与行号，帮助你从“定义设备”到“命令行可用”再到“录数训练”完整打通。

## 0) 先理解 LeRobot 是如何识别你自定义硬件的

LeRobot 的硬件接入核心是三层：

- 抽象接口层（你要实现的基类）
  - Robot 基类：[`src/lerobot/robots/robot.py#L30`](../../src/lerobot/robots/robot.py#L30)
  - Teleoperator 基类：[`src/lerobot/teleoperators/teleoperator.py#L29`](../../src/lerobot/teleoperators/teleoperator.py#L29)
  - Camera 基类：[`src/lerobot/cameras/camera.py#L26`](../../src/lerobot/cameras/camera.py#L26)
- Config 注册层（`ChoiceRegistry`）
  - RobotConfig：[`src/lerobot/robots/config.py#L23`](../../src/lerobot/robots/config.py#L23)
  - TeleoperatorConfig：[`src/lerobot/teleoperators/config.py#L23`](../../src/lerobot/teleoperators/config.py#L23)
  - CameraConfig：[`src/lerobot/cameras/configs.py#L60`](../../src/lerobot/cameras/configs.py#L60)
- 实例化层（factory + 第三方插件动态导入）
  - 插件扫描与导入：[`src/lerobot/utils/import_utils.py#L144`](../../src/lerobot/utils/import_utils.py#L144)
  - 插件前缀约束：[`src/lerobot/utils/import_utils.py#L152`](../../src/lerobot/utils/import_utils.py#L152)
  - 动态类推断并实例化：[`src/lerobot/utils/import_utils.py#L79`](../../src/lerobot/utils/import_utils.py#L79)

## 1) 自建 Robot：必须实现哪些接口

### 1.1 Config 部分

- 继承 `RobotConfig`，并注册 type（文档示例是 `@RobotConfig.register_subclass("my_cool_robot")`）。
- `RobotConfig` 通用字段：
  - `id` / `calibration_dir`：[`src/lerobot/robots/config.py#L25`](../../src/lerobot/robots/config.py#L25)
- 如果你的 robot 带 camera，`RobotConfig.__post_init__` 会检查 camera 的 `width/height/fps` 是否完整：
  - [`src/lerobot/robots/config.py#L29`](../../src/lerobot/robots/config.py#L29)

### 1.2 代码部分（Robot 抽象契约）

你至少要实现这些抽象属性/方法：

- `observation_features`：[`src/lerobot/robots/robot.py#L90`](../../src/lerobot/robots/robot.py#L90)
- `action_features`：[`src/lerobot/robots/robot.py#L104`](../../src/lerobot/robots/robot.py#L104)
- `is_connected`：[`src/lerobot/robots/robot.py#L117`](../../src/lerobot/robots/robot.py#L117)
- `connect`：[`src/lerobot/robots/robot.py#L125`](../../src/lerobot/robots/robot.py#L125)
- `is_calibrated` / `calibrate`：[`src/lerobot/robots/robot.py#L137`](../../src/lerobot/robots/robot.py#L137)、[`src/lerobot/robots/robot.py#L142`](../../src/lerobot/robots/robot.py#L142)
- `configure`：[`src/lerobot/robots/robot.py#L174`](../../src/lerobot/robots/robot.py#L174)
- `get_observation`：[`src/lerobot/robots/robot.py#L182`](../../src/lerobot/robots/robot.py#L182)
- `send_action`：[`src/lerobot/robots/robot.py#L194`](../../src/lerobot/robots/robot.py#L194)
- `disconnect`：[`src/lerobot/robots/robot.py#L209`](../../src/lerobot/robots/robot.py#L209)

可以直接参考内置实现（最贴近官方教程逻辑）：

- `SOFollower`：[`src/lerobot/robots/so_follower/so_follower.py#L38`](../../src/lerobot/robots/so_follower/so_follower.py#L38)
- 特征定义：[`src/lerobot/robots/so_follower/so_follower.py#L77`](../../src/lerobot/robots/so_follower/so_follower.py#L77)
- 连接/标定/配置：[`src/lerobot/robots/so_follower/so_follower.py#L89`](../../src/lerobot/robots/so_follower/so_follower.py#L89)、[`src/lerobot/robots/so_follower/so_follower.py#L112`](../../src/lerobot/robots/so_follower/so_follower.py#L112)、[`src/lerobot/robots/so_follower/so_follower.py#L156`](../../src/lerobot/robots/so_follower/so_follower.py#L156)
- 观测/动作：[`src/lerobot/robots/so_follower/so_follower.py#L179`](../../src/lerobot/robots/so_follower/so_follower.py#L179)、[`src/lerobot/robots/so_follower/so_follower.py#L197`](../../src/lerobot/robots/so_follower/so_follower.py#L197)

## 2) 自建 Teleoperator：必须实现哪些接口

### 2.1 Config 部分

- 继承 `TeleoperatorConfig` 并注册 type。
- 通用字段同样是 `id` / `calibration_dir`：
  - [`src/lerobot/teleoperators/config.py#L23`](../../src/lerobot/teleoperators/config.py#L23)

### 2.2 代码部分（Teleoperator 抽象契约）

你至少要实现：

- `action_features`：[`src/lerobot/teleoperators/teleoperator.py#L89`](../../src/lerobot/teleoperators/teleoperator.py#L89)
- `feedback_features`：[`src/lerobot/teleoperators/teleoperator.py#L102`](../../src/lerobot/teleoperators/teleoperator.py#L102)
- `connect/calibrate/configure`：[`src/lerobot/teleoperators/teleoperator.py#L123`](../../src/lerobot/teleoperators/teleoperator.py#L123)、[`src/lerobot/teleoperators/teleoperator.py#L140`](../../src/lerobot/teleoperators/teleoperator.py#L140)、[`src/lerobot/teleoperators/teleoperator.py#L172`](../../src/lerobot/teleoperators/teleoperator.py#L172)
- `get_action/send_feedback`：[`src/lerobot/teleoperators/teleoperator.py#L180`](../../src/lerobot/teleoperators/teleoperator.py#L180)、[`src/lerobot/teleoperators/teleoperator.py#L191`](../../src/lerobot/teleoperators/teleoperator.py#L191)
- `disconnect`：[`src/lerobot/teleoperators/teleoperator.py#L206`](../../src/lerobot/teleoperators/teleoperator.py#L206)

可参考内置实现：

- `SOLeader`：[`src/lerobot/teleoperators/so_leader/so_leader.py#L34`](../../src/lerobot/teleoperators/so_leader/so_leader.py#L34)
- `get_action`：[`src/lerobot/teleoperators/so_leader/so_leader.py#L141`](../../src/lerobot/teleoperators/so_leader/so_leader.py#L141)

## 3) 自建 Camera（可选，但常见）

如果你不使用内置相机实现（opencv/realsense/zmq 等），就需要自建 camera 插件。

- Camera 抽象接口：[`src/lerobot/cameras/camera.py#L26`](../../src/lerobot/cameras/camera.py#L26)
- 必需方法：
  - `connect`：[`src/lerobot/cameras/camera.py#L100`](../../src/lerobot/cameras/camera.py#L100)
  - `read`：[`src/lerobot/cameras/camera.py#L111`](../../src/lerobot/cameras/camera.py#L111)
  - `async_read`：[`src/lerobot/cameras/camera.py#L122`](../../src/lerobot/cameras/camera.py#L122)
  - `disconnect`：[`src/lerobot/cameras/camera.py#L182`](../../src/lerobot/cameras/camera.py#L182)
- CameraConfig 基类：[`src/lerobot/cameras/configs.py#L60`](../../src/lerobot/cameras/configs.py#L60)

## 4) 插件包约定（决定 CLI 能否自动识别）

这部分和 `integrate_hardware.mdx` 的四条 convention 对应，仓库代码上由 `register_third_party_plugins` 与 `make_device_from_device_class` 实现：

- 包名前缀必须是其一：
  - `lerobot_robot_`
  - `lerobot_camera_`
  - `lerobot_teleoperator_`
  - 代码：[`src/lerobot/utils/import_utils.py#L152`](../../src/lerobot/utils/import_utils.py#L152)
- 类名规则：
  - `MyDeviceConfig` 对应 `MyDevice`（去掉 `Config` 后缀）
  - 代码：[`src/lerobot/utils/import_utils.py#L97`](../../src/lerobot/utils/import_utils.py#L97)、[`src/lerobot/utils/import_utils.py#L100`](../../src/lerobot/utils/import_utils.py#L100)
- 模块查找规则（同模块/同包推断模块）：
  - 代码：[`src/lerobot/utils/import_utils.py#L103`](../../src/lerobot/utils/import_utils.py#L103)、[`src/lerobot/utils/import_utils.py#L112`](../../src/lerobot/utils/import_utils.py#L112)
- 工厂 fallback 到动态实例化：
  - Robot：[`src/lerobot/robots/utils.py#L77`](../../src/lerobot/robots/utils.py#L77)
  - Teleoperator：[`src/lerobot/teleoperators/utils.py#L100`](../../src/lerobot/teleoperators/utils.py#L100)
  - Camera：[`src/lerobot/cameras/utils.py#L52`](../../src/lerobot/cameras/utils.py#L52)

另外，建议在插件包 `__init__.py` 导出 Config 类和设备类。LeRobot 自身模块也遵循这个模式：

- Robot 导出：[`src/lerobot/robots/__init__.py#L17`](../../src/lerobot/robots/__init__.py#L17)
- Teleoperator 导出：[`src/lerobot/teleoperators/__init__.py#L17`](../../src/lerobot/teleoperators/__init__.py#L17)
- Camera 导出：[`src/lerobot/cameras/__init__.py#L15`](../../src/lerobot/cameras/__init__.py#L15)

## 5) 命令行如何真正调用到你的设备

CLI 入口：

- `lerobot-calibrate`：[`pyproject.toml#L197`](../../pyproject.toml#L197)
- `lerobot-record`：[`pyproject.toml#L200`](../../pyproject.toml#L200)
- `lerobot-replay`：[`pyproject.toml#L201`](../../pyproject.toml#L201)
- `lerobot-teleoperate`：[`pyproject.toml#L203`](../../pyproject.toml#L203)

每个命令的 `main()` 都会先注册第三方插件：

- calibrate：[`src/lerobot/scripts/lerobot_calibrate.py#L97`](../../src/lerobot/scripts/lerobot_calibrate.py#L97)
- record：[`src/lerobot/scripts/lerobot_record.py#L570`](../../src/lerobot/scripts/lerobot_record.py#L570)
- replay：[`src/lerobot/scripts/lerobot_replay.py#L136`](../../src/lerobot/scripts/lerobot_replay.py#L136)
- teleoperate：[`src/lerobot/scripts/lerobot_teleoperate.py#L245`](../../src/lerobot/scripts/lerobot_teleoperate.py#L245)

对应配置 dataclass（也就是你在 CLI 里写的 `--robot.*` / `--teleop.*` / `--dataset.*`）：

- `CalibrateConfig`：[`src/lerobot/scripts/lerobot_calibrate.py#L67`](../../src/lerobot/scripts/lerobot_calibrate.py#L67)
- `TeleoperateConfig`：[`src/lerobot/scripts/lerobot_teleoperate.py#L108`](../../src/lerobot/scripts/lerobot_teleoperate.py#L108)
- `DatasetRecordConfig` / `RecordConfig`：[`src/lerobot/scripts/lerobot_record.py#L147`](../../src/lerobot/scripts/lerobot_record.py#L147)、[`src/lerobot/scripts/lerobot_record.py#L194`](../../src/lerobot/scripts/lerobot_record.py#L194)
- `ReplayConfig`：[`src/lerobot/scripts/lerobot_replay.py#L90`](../../src/lerobot/scripts/lerobot_replay.py#L90)

## 6) 运行时 I/O 调用链（最关键）

你的硬件方法是否被正确调用，可以直接看这几处：

- teleoperate 主循环：
  - `robot.get_observation()`：[`src/lerobot/scripts/lerobot_teleoperate.py#L163`](../../src/lerobot/scripts/lerobot_teleoperate.py#L163)
  - `teleop.get_action()`：[`src/lerobot/scripts/lerobot_teleoperate.py#L166`](../../src/lerobot/scripts/lerobot_teleoperate.py#L166)
  - `robot.send_action(...)`：[`src/lerobot/scripts/lerobot_teleoperate.py#L175`](../../src/lerobot/scripts/lerobot_teleoperate.py#L175)
- record 主循环：
  - `robot.get_observation()`：[`src/lerobot/scripts/lerobot_record.py#L331`](../../src/lerobot/scripts/lerobot_record.py#L331)
  - `teleop.get_action()`：[`src/lerobot/scripts/lerobot_record.py#L355`](../../src/lerobot/scripts/lerobot_record.py#L355)
  - `robot.send_action(...)`：[`src/lerobot/scripts/lerobot_record.py#L387`](../../src/lerobot/scripts/lerobot_record.py#L387)
- replay：
  - `robot.get_observation()`：[`src/lerobot/scripts/lerobot_replay.py#L123`](../../src/lerobot/scripts/lerobot_replay.py#L123)
  - `robot.send_action(...)`：[`src/lerobot/scripts/lerobot_replay.py#L127`](../../src/lerobot/scripts/lerobot_replay.py#L127)

## 7) 数据集特征如何从你的硬件定义中生成

录数时，数据集特征来自你实现的 `robot.action_features` 和 `robot.observation_features`：

- 从 robot 特征创建初始 feature：
  - action：[`src/lerobot/scripts/lerobot_record.py#L426`](../../src/lerobot/scripts/lerobot_record.py#L426)
  - observation：[`src/lerobot/scripts/lerobot_record.py#L433`](../../src/lerobot/scripts/lerobot_record.py#L433)
- 聚合 pipeline 后形成最终 dataset features：
  - [`src/lerobot/scripts/lerobot_record.py#L423`](../../src/lerobot/scripts/lerobot_record.py#L423)
  - [`src/lerobot/datasets/pipeline_features.py#L25`](../../src/lerobot/datasets/pipeline_features.py#L25)
  - [`src/lerobot/datasets/pipeline_features.py#L67`](../../src/lerobot/datasets/pipeline_features.py#L67)

这也是为什么你必须保证 `observation_features` / `action_features` 与 `get_observation` / `send_action` 结构严格一致。

## 8) 最小验证流程（建议按这个顺序）

1. 先只跑标定，确认 connect/calibrate/disconnect 没问题：

```bash
lerobot-calibrate --robot.type=<your_robot_type> --robot.id=<your_id> ...
```

2. 再跑遥操作，确认实时 I/O 稳定：

```bash
lerobot-teleoperate --robot.type=<your_robot_type> --teleop.type=<your_teleop_type> ...
```

3. 再录一个最小数据集，确认 features 与写盘正确：

```bash
lerobot-record --robot.type=<your_robot_type> --dataset.repo_id=<user>/<dataset> --dataset.single_task="..." ...
```

4. 最后用 replay 回放一段动作，确认执行一致性：

```bash
lerobot-replay --robot.type=<your_robot_type> --dataset.repo_id=<user>/<dataset> --dataset.episode=0
```

## 9) 常见问题快速定位

1. `type` 识别不到
- 先看是否调用了插件注册：[`src/lerobot/utils/import_utils.py#L144`](../../src/lerobot/utils/import_utils.py#L144)
- 再看包名前缀是否符合约定：[`src/lerobot/utils/import_utils.py#L152`](../../src/lerobot/utils/import_utils.py#L152)

2. Config 能识别，但实例化设备类失败
- 检查 `SomethingConfig`/`Something` 命名是否对应：
  - [`src/lerobot/utils/import_utils.py#L97`](../../src/lerobot/utils/import_utils.py#L97)
- 检查文件布局是否在候选模块路径中：
  - [`src/lerobot/utils/import_utils.py#L103`](../../src/lerobot/utils/import_utils.py#L103)

3. 录数时报 feature 不匹配
- 重点核对 `observation_features` / `action_features` 与运行时返回值字典键是否一致：
  - 定义：[`src/lerobot/robots/robot.py#L90`](../../src/lerobot/robots/robot.py#L90)、[`src/lerobot/robots/robot.py#L104`](../../src/lerobot/robots/robot.py#L104)
  - 消费：[`src/lerobot/scripts/lerobot_record.py#L426`](../../src/lerobot/scripts/lerobot_record.py#L426)、[`src/lerobot/scripts/lerobot_record.py#L433`](../../src/lerobot/scripts/lerobot_record.py#L433)

### Looking for an example ?

Check out these two packages from the community:

- https://github.com/SpesRobotics/lerobot-robot-xarm
- https://github.com/SpesRobotics/lerobot-teleoperator-teleop

## Wrapping Up

Once your robot class is complete, you can leverage the LeRobot ecosystem:

- Control your robot with available teleoperators or integrate directly your teleoperating device
- Record training data and visualize it
- Integrate it into RL or imitation learning pipelines

Don't hesitate to reach out to the community for help on our [Discord](https://discord.gg/s3KuuzsPFb) 🤗
