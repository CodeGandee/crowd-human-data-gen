# TaskManager_coco_like_data 使用说明

本说明介绍如何使用 `TaskManager_coco_like_data.py` 生成接近 COCO 风格的标注产物（实例分割、头/脸掩码、深度、可见性统计、OpenPose 关节等），并保存到本地或数据库。

- 主要组件
  - 类：`TaskManager`、`TaskConfig`、`PersonLabel`
  - 关键方法：
    - `TaskManager.genarate_task_by_position()`：按规则在场景可行走区域采样人物并生成一次任务
    - `TaskManager.create_task(scene_id, person_info)`：指定场景与人物位置生成任务
    - `TaskConfig.save_to_local_path(path)`：保存整任务及每个人的产物到本地
    - `TaskConfig.save_data_to_database(bucket, minio, postgres)`：保存到 MinIO + Postgres

- 配置文件
  - 数据库连接与存储桶：`config/database_config.yaml`

## 1. 运行环境与依赖

- 准备 MinIO 与 Postgres，并在 `config/database_config.yaml` 中填写连接信息（scene/render/task 库、存储桶如 `sceneinfos`、`renderinfos`、`taskinfos` 等）。
- 常见依赖（按你的环境为准）：numpy、opencv-python、trimesh、pyrender、psycopg2、minio、distinctipy、matplotlib、pyglet。
- 注意：脚本内包含若干 `plt.show()` 可视化窗口，服务器/无显示环境可注释掉以避免阻塞。

## 2. 快速开始（命令行）

```cmd
cd /d d:\Code\project\RenderToolbox
python TaskManager_coco_like_data.py
```

默认行为：
- 随机抽取一个场景，从其 3D 可行走区域中采样 2 人左右（近/远混合）。
- 渲染整场景 RGB/Depth/Seg、头/脸掩码与可见性统计，并为每人生成单人产物与 OpenPose 关节。
- 将结果保存到本地目录（脚本中默认示例为 `E:\tmp\syn_image_visualize_demo`）。
- 如需直接入库，取消 main 中相应注释并配置存储桶名。

## 3. 以编程方式调用

```python
from TaskManager_coco_like_data import TaskManager
import numpy as np

# 1) 初始化（传入配置文件路径）
tm = TaskManager(db_config_path='./config/database_config.yaml')

# 2) 自动按规则采样并生成任务（返回 TaskConfig）
task = tm.genarate_task_by_position()
if task:
    # 本地保存（会以 task_uuid 建目录）
    task.save_to_local_path(r'./output/coco_like')
    
    # 或保存到 MinIO + Postgres（需要有效的存储桶与表结构）
    task.save_data_to_database(bucket_name='taskinfos', minio=tm.minio, postgres=tm.task_postgres)

# 3) 手动指定场景与人物位置
scene_id = 'your_scene_id'
person_info = [  # (character_id, world_position: np.ndarray[3])
    ('character_001', np.array([1.2, 0.0, 6.5])),
    ('character_002', np.array([-0.8, 0.0, 4.2])),
]
task2 = tm.create_task(scene_id=scene_id, person_info=person_info)
if task2:
    task2.save_to_local_path(r'./output/manual')
    task2.save_data_to_database(bucket_name='taskinfos', minio=tm.minio, postgres=tm.task_postgres)
```

说明：
- `genarate_task_by_position()` 会：
  - 随机选场景 -> 读取背景图与相机参数 -> 在 3D 区域内按近/远规则采样若干人物（去重、边界与碰撞检测） -> 渲染并产出任务。
- `create_task(scene_id, person_info)` 需传入已存在的场景 ID 与人物在该场景世界坐标的位置。

## 4. 产物与目录结构

调用 `TaskConfig.save_to_local_path(path)` 后会创建：
- `path/<task_uuid>/`
  - `depth_data.npy`：整场景深度矩阵
  - `depth_image.png`：归一化深度图（仅对有深度区域归一化）
  - `seg_data.png`：整场景实例分割（RGB 伪色）
  - `head_mask.png` / `face_mask.png`：整场景头/脸掩码（与 seg 对应颜色）
  - `rgb_data.png`：人物贴到背景后的 RGB 结果
  - `occlusion_label.pkl`：场景级可见性统计，按 person_id 汇总
  - `<person_id>/`（每人一个子目录）
    - `rgb.png`、`depth_data.npy`、`depth_image.png`、`seg_data.png`、`head_mask.png`、`face_mask.png`

额外：`TaskConfig`/`PersonLabel` 在入库路径下还会生成 openpose、原始 3D 标注等数据（保存到 MinIO/数据库时为 pickle 或 JSON 字节流）。

## 5. 保存到数据库

- 一键：`TaskConfig.save_data_to_database(bucket, minio, postgres)`
  - 内部依次：`save_data_to_minio`（上传二进制/图片/数组/标注） → `save_info_to_postgres`（写入任务索引 + 文件映射表） → 每个 `PersonLabel.save_data_to_database`。
- 访问的库与表（示例）：
  - 读取：`scene/character/mesh/filepath` 等表（场景、人物、网格与文件映射）
  - 写入：`task`、`person`、`filepath`（使用 `conflict` upsert）
- 存储桶：
  - 读取：`sceneinfos`、`renderinfos`；写入：如 `taskinfos`（可在配置与调用处调整）。

## 6. 常见问题

- 运行时弹出多个图像窗口：注释 `plt.show()`，或在无头环境配置非交互后端。
- 无法连通 MinIO/Postgres：检查 `config/database_config.yaml`；确认桶/表结构存在且权限正确。
- 采样不到人物或完全被遮挡：调整 `genarate_task_by_position` 中的深度阈值、期望人数，或改用 `create_task` 手动指定位置。
- Windows 路径写入：`save_to_local_path` 目标路径需存在可写盘符，例如 `E:\tmp\...`；或改为项目相对目录。
