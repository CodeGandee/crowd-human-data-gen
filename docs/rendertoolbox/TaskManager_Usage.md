# TaskManager 使用说明

本说明文档介绍如何使用 `TaskManager.py` 生成渲染任务、保存结果到本地与数据库（MinIO + Postgres）。

- 主要类与方法
  - `TaskManager.TaskManager`：任务入口类
    - `TaskManager.TaskManager.create_task(scene_id, person_info)`：按给定 scene_id 与人员位置生成任务
    - `TaskManager.TaskManager.genarate_task_by_position()`：自动采样场景中人员并生成任务
  - `TaskManager.TaskConfig`：单次任务的结果载体
    - `TaskManager.TaskConfig.save_to_local_path(path)`：将任务输出保存到本地
    - `TaskManager.TaskConfig.save_data_to_minio(bucket, minio)`：将任务数据写入 MinIO
    - `TaskManager.TaskConfig.save_info_to_postgres(postgres)`：将任务索引写入 Postgres
    - `TaskManager.TaskConfig.save_data_to_database(bucket, minio, postgres)`：一键保存到 MinIO + Postgres
  - `TaskManager.PersonLabel`：单人标注结果
    - `TaskManager.PersonLabel.save_to_local_path(path)`：单人结果本地落盘

- 配置文件
  - 数据库与存储配置：`config/database_config.yaml`

## 1. 环境与依赖

- 配置 MinIO 与 Postgres 连接信息于 `config/database_config.yaml`。
- 安装依赖（按你的项目环境为准，示例：numpy、opencv-python、trimesh、pyrender、psycopg2、minio、distinctipy 等）。

## 2. 快速开始

- 直接运行脚本主入口（会自动采样人员并生成任务）：

```cmd
cd /d d:\Code\project\RenderToolbox
python TaskManager.py
```

- 以编程方式调用：

```python
from TaskManager import TaskManager
import numpy as np

# 1) 初始化（传入配置文件路径）
tm = TaskManager(db_config_path='./config/database_config.yaml')

# 2) 方式A：自动采样人员并生成任务
task_cfg = tm.genarate_task_by_position()  # 返回 TaskConfig
if task_cfg:
    # 本地保存
    task_cfg.save_to_local_path(r'./output/auto')
    # 或保存到 MinIO + Postgres
    task_cfg.save_data_to_database(bucket_name='taskinfos', minio=tm.minio, postgres=tm.task_postgres)

# 3) 方式B：指定场景与人员位置生成任务
scene_id = 'your_scene_id'
# person_info: List[Tuple[str, np.ndarray(3,)]] (character_id, world_position)
person_info = [
    ('character_001', np.array([1.2, 0.0, 6.5])),
    ('character_002', np.array([-0.8, 0.0, 4.2])),
]
task_cfg2 = tm.create_task(scene_id=scene_id, person_info=person_info)
if task_cfg2:
    task_cfg2.save_to_local_path(r'./output/manual')
    task_cfg2.save_data_to_database(bucket_name='taskinfos', minio=tm.minio, postgres=tm.task_postgres)
```

说明：
- `create_task` 需要已有的 `scene_id` 与人员世界坐标；内部会加载场景、渲染人物与标注。
- `genarate_task_by_position` 会读取场景 3D 可行走区域并按深度规则采样近/远距离位置。

## 3. 主要 API 说明

- 初始化
  - `TaskManager(db_config_path: str)`：读取 `config/database_config.yaml`，建立 MinIO 与 Postgres 连接。

- 生成任务
  - `TaskManager.create_task(scene_id: str, person_info: List[Tuple[str, np.ndarray]]) -> TaskConfig`
  - `TaskManager.genarate_task_by_position() -> TaskConfig`

- 任务结果保存
  - 本地：`TaskConfig.save_to_local_path(path: str)`
  - 数据库：`TaskConfig.save_data_to_database(bucket: str, minio, postgres)`

## 4. 输出内容

调用 `TaskConfig.save_to_local_path` 后的目录结构示例：
- 任务根目录（以 `task_uuid` 命名）
  - depth_data.npy：深度矩阵
  - depth_image.png：深度图（归一化）
  - seg_data.png：实例分割
  - head_mask.png / face_mask.png：头部/人脸掩码
  - rgb_data.png：叠加人物后的 RGB 图
  - occlusion_label.pkl：遮挡标注（场景级）
  - 每个人物一个子目录（以人物 ID 命名），含该人物 rgb、mask、depth、head_mask、face_mask 等

单人结果也可通过 `PersonLabel.save_to_local_path` 单独保存。

## 5. 示例入口

可参考 `TaskManager.py` 末尾示例：
- 初始化 `TaskManager`
- 调用 `genarate_task_by_position`
- 调用 `save_to_local_path` 与 `save_data_to_database`

## 6. 常见问题

- 无法连接 MinIO 或 Postgres：请检查 `config/database_config.yaml`。
- 生成空任务：可能是人物被完全遮挡或采样不到有效位置；可调整阈值与人数，或使用 `create_task` 指定位置。
