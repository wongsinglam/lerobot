## Evaluate SmolVLA

```
pip install -e ".[smolvla]"
pip install numpy==1.26.0 # revert numpy to version 1.26
```
```
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

### Evaluate PI0.5
```
pip install -e ".[pi]"
pip install numpy==1.26.0 # revert numpy to version 1.26
```
```
PI0.5 requires disabling torch compile for evaluation:
```
```
TORCH_COMPILE_DISABLE=1 TORCHINDUCTOR_DISABLE=1 lerobot-eval \
    --policy.path=nvidia/pi05-arena-gr1-microwave \
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
    --env.video_length=15 \
    --env.video_interval=15 \
    --env.state_keys=robot_joint_pos \
    --env.camera_keys=robot_pov_cam_rgb \
    --trust_remote_code=True \
    --eval.batch_size=1
```
```
To change the number of parallel environments, use the ```--eval.batch_size``` flag.
```