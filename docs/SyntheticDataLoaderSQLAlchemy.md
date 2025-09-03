# SyntheticDataLoaderSQLAlchemy 模块文档

本模块基于 SQLAlchemy 与 MinIO，提供从多库（datasets、taskinfos、sceneinfos）检索样本元数据与实际二进制文件的能力，并将样本结构化为可直接下载的组件集合。

- 数据库：PostgreSQL（datasets / taskinfos / sceneinfos）
- 对象存储：MinIO（bucket：taskinfos、sceneinfos）
- 主要类：`DataServerHandle` → `PersonDatasetHandle` → `PersonSampleHandle`

> 目标：给定静态样本表（static table）与样本 ID，查询相关文件在 MinIO 的存储路径并下载到本地，或以字节形式获取。

---

## 安装与依赖

运行环境建议：Python 3.9+

第三方依赖（按实际导入）：
- sqlalchemy（含 `psycopg2-binary` 作为 PostgreSQL 驱动）
- omegaconf
- minio
- numpy
- opencv-python（cv2）
- imageio
- pgvector（用于 `pgvector.sqlalchemy`）
- matplotlib（仅示例/可选）

可选安装（仅供参考，按需执行）：

```bat
:: 可选示例命令（按需执行）
pip install sqlalchemy psycopg2-binary omegaconf minio numpy opencv-python imageio pgvector matplotlib
```

> 说明：请根据你现有的环境与公司源配置调整安装方式与版本。

---

## 配置

使用 `config/database_config.yaml` 进行数据库与 MinIO 连接配置，关键段：

```yaml
MongoDB:
  host: ...
  port: 27017
  username: ...
  password: ...

MinIO:
  host: 192.168.13.183
  port: 29000
  username: minioadmin
  password: minioadmin

RenderPostgres: { ... }
ScenePostgres:  { ... }
TaskPostgres:   { ... }
DatasetPostgres:{ ... }
```

- MinIO 连接默认 `secure=False`（非 TLS），如需启用 HTTPS 请在代码中调整。
- Bucket 名称在代码中写死：`taskinfos` 与 `sceneinfos`，需保证存在。

---

## 数据表要求与字段约定

模块当前只支持数据集名称为 `Person` 的数据集，且需要存在以下数据表与字段。

1) datasets（`DatasetPostgres`）
- 表：`dataset_info`
- 读取逻辑：`SELECT datasets_name FROM dataset_info WHERE id=1`，要求返回包含 `"Person"` 的名称列表（或兼容可迭代）。

2) taskinfos（`TaskPostgres`）
- 表：`person`、`sd_person`、`filepath`、以及你的静态样本表（`static_table_name`）
- `person` 表字段（代码中使用）：
  - `instance_id`, `person_id`
  - `rgb_path`, `mask_path`, `head_mask_path`, `face_mask_path`, `depth_path`
  - `occlusion_label_path`, `openpose_data_path`, `original_data_path`
- `sd_person` 表字段：
  - `person_image_id`, `person_instance_id`, `scene_image_id`
- `filepath` 表字段：
  - `file_id`, `storage_path`
- 静态样本表（`static_table_name`）字段（最小集合）：
  - `id`（作为样本主键，1-based 顺序用于检索）
  - `background_image_path`（scene 库文件 ID）
  - `scene_mask_path`, `scene_rgb_path`, `scene_head_mask_path`, `scene_face_mask_path`
  - `scene_depth_path`, `person_instance_ids_path`, `mask_info_path`, `scene_occlusion_label_path`
  - `scene_image_id`（注意：代码中既用于与 `sd_person.scene_image_id` 关联，也作为到 `taskinfos.filepath` 的 file_id 使用）

3) sceneinfos（`ScenePostgres`）
- 表：`filepath`
- 字段：`file_id`, `storage_path`

> 解析文件真实存储路径的规则：
> - 当 `database == 'task'` 时，到 `taskinfos.filepath` 用 `file_id` → `storage_path`。
> - 当 `database == 'scene'` 时，到 `sceneinfos.filepath` 用 `file_id` → `storage_path`。
> 再以对应 bucket（`taskinfos` / `sceneinfos`）+ `storage_path` 从 MinIO 获取数据。

---

## 核心类与职责

### DataServerHandle

- 初始化：`DataServerHandle(config: dict)`
  - `config` 可由 `OmegaConf.load('.../database_config.yaml')` 读入后传入。
  - 连接 datasets 数据库，反射获得 `dataset_info` 表。
- `list_all_datasets()`：读取 `id=1` 的 `datasets_name` 并返回。
- `get_dataset_handle(dataset_name: str, static_table_name: str)`：
  - 目前仅当 `dataset_name == 'Person'` 时返回 `PersonDatasetHandle`；否则抛出异常。
- `close()`：释放连接。

### PersonDatasetHandle

- 初始化参数：`database_config: DBConnectionConfig`, `static_table_name: str`
  - 连接 taskinfos、sceneinfos 数据库，反射所需表。
  - 初始化 MinIO 客户端。
  - 通过 `COUNT(static_table.scene_image_id)` 计算样本总数，作为 `length`。
- 属性：`length` 样本总量。
- `get_sample_handle(sample_id: int)`：
  - 1-based 检查：`1 <= sample_id <= length`。
  - 返回 `PersonSampleHandle`。
- 其余方法（如 `fetch_amount_of_sample_uuids_streaming`、`list_all_samples_uuid`、`count_samples_num`、`random_sample_n_uuids`）当前未实现（`NotImplementedError`）。
- `close()`：释放连接。

### PersonSampleHandle

- `get_sample_components() -> OmegaConf`
  - 查询静态样本表中 `id == sample_id` 的一行，拼装该样本的完整组件信息：
    - `scene`：场景级别组件
    - `persons`：按 `person_instance_id` 组织的人员组件
    - `extra_info`：额外信息（例如 `person_iid2pid_map`）
  - 并发起到 `person` 和 `sd_person` 的联合查询，收集人员图像与生成图。
  - 返回结构为 `OmegaConf`（可当作字典使用）。

  返回结构示例（字段名后缀 `_info` 表示一个“引用对象”，包含 MinIO file_id 和归属库标记）：
  
  ```yaml
  scene:
    background_image_info: { path: <file_id>, database: scene }
    scene_mask_info:       { path: <file_id>, database: task, BGR: true }
    scene_rgb_info:        { path: <file_id>, database: task, BGR: true }
    scene_head_mask_info:  { path: <file_id>, database: task, BGR: true }
    scene_face_mask_info:  { path: <file_id>, database: task, BGR: true }
    scene_depth_info:      { path: <file_id>, database: task }
    person_instance_ids_info: { path: <file_id>, database: task }
    mask_info_info:        { path: <file_id>, database: task }
    scene_occlusion_label_info: { path: <file_id>, database: task }
    scene_sd_image_info:   { path: <file_id>, database: task }
  persons:
    <person_instance_id>:
      person_image_info:   { path: <file_id>, database: task, BGR: true }
      mask_info:           { path: <file_id>, database: task }
      head_mask_info:      { path: <file_id>, database: task }
      face_mask_info:      { path: <file_id>, database: task }
      depth_info:          { path: <file_id>, database: task }
      occlusion_label_info:{ path: <file_id>, database: task }
      openpose_data_info:  { path: <file_id>, database: task }
      original_data_info:  { path: <file_id>, database: task }
      person_sd_image_info:{ path: <file_id>, database: task }
  extra_info:
    person_iid2pid_map: { <instance_id>: <person_id>, ... }
  ```

- `fetch_binary_data(component_info: OmegaConf) -> bytes`
  - 根据 `component_info.database`（`task`/`scene`）定位 `filepath` 表拿到 `storage_path`，从 MinIO 对应 bucket 下载为字节。
  - 若存在 `BGR: true`，会走 `_cvt_bgr_to_rgb` 路径：实现在当前代码中是 `cv2.imdecode` 后直接以 `imageio.imwrite('<bytes>', ...)` 重新编码为 PNG；未进行颜色通道 BGR→RGB 转换（函数名有“to_rgb”，但实现保持了 BGR 排列再编码 PNG）。

- `download_one_sample(sample_contents: OmegaConf, download_path: str)`
  - 将 `sample_contents` 中所有组件下载到 `download_path/<sample_id>/` 下：
    - 扩展名规则由类别决定：
      - `TYPE_IMAGE` → `.png`
      - `TYPE_PICKLE` → `.pkl`
      - `TYPE_NUMPY` → `.npy`
    - `extra_info.person_iid2pid_map` 将保存为 `person_iid2pid_map.pkl`。

---

## 类型与扩展名映射

模块内置三类组件名集合，用于确定输出文件后缀：
- `TYPE_IMAGE`：`['background_image_info', 'scene_mask_info', 'scene_rgb_info', 'scene_head_mask_info', 'scene_face_mask_info','scene_sd_image_info', 'person_image_info', 'mask_info', 'head_mask_info', 'face_mask_info', 'person_sd_image_info']`
- `TYPE_PICKLE`：`['person_instance_ids_info', 'mask_info_info', 'scene_occlusion_label_info', 'openpose_data_info', 'occlusion_label_info', 'original_data_info']`
- `TYPE_NUMPY`：`['scene_depth_info', 'depth_info']`

> 任何不在上述集合内的组件将按“原名（去掉 `_info` 后缀）”保存，不追加扩展名。

---

## 快速开始（示例）

以下示例演示如何加载配置、获取数据集句柄、读取样本并下载：

```python
from omegaconf import OmegaConf
from SyntheticDataLoaderSQLAlchemy import DataServerHandle

config_path = r'D:\\Code\\project\\SyntheticDataLoader\\config\\database_config.yaml'
config = OmegaConf.load(config_path)

# 1) 创建数据服务句柄
server = DataServerHandle(config)

# 2) 指定静态样本表名（例如：TestStaticTable_20240910_151126）并获取数据集句柄
person_ds = server.get_dataset_handle('Person', 'TestStaticTable_20240910_151126')

# 3) 选择样本 id（1-based）并获取样本句柄
sample = person_ds.get_sample_handle(sample_id=11)

# 4) 查询样本组件（返回 OmegaConf，可当作 dict 使用）
components = sample.get_sample_components()

# 5) 下载所有组件到目标目录
sample.download_one_sample(components, r'D:\\tmp\\synthetic_samples')

# 6) 释放资源
person_ds.close()
server.close()
```

> 注意：`sample_id` 从 1 开始，且必须 `<= person_ds.length`。

---

## 常见问题（FAQ）

- 连接失败/驱动缺失：
  - PostgreSQL 驱动需安装 `psycopg2-binary`；若企业环境限制，可使用编译版本或系统包。
  - `pgvector.sqlalchemy` 需要安装 `pgvector` 包，且数据库侧应已安装/启用 `vector` 扩展（如无必需功能，也可不使用相关列）。
- MinIO 取文件报错：
  - 请确认 `taskinfos`/`sceneinfos` bucket 存在；
  - 确认 `filepath` 表中 `file_id -> storage_path` 映射存在；
  - 账号/密码与网络连通是否正确。
- 颜色异常：
  - 标记了 `BGR: true` 的图像在当前实现中会被解码并直接编码为 PNG，未执行 BGR→RGB 通道转换；如需转换可在本地再行处理，或在模块中将注释的 `cv2.cvtColor` 打开。
- `get_dataset_handle` 抛出 `Dataset not found`：
  - 请检查 datasets 库 `dataset_info` 的 `id=1` 行 `datasets_name` 是否包含 `Person`；
  - 或根据实际需求扩展 `DataServerHandle.get_dataset_handle` 的逻辑以支持更多数据集名称。

---

## 变更建议（供参考）

- 将 `list_all_datasets()` 的固定 `id==1` 改为查询全部，并以更稳健的结构（如多行）返回。
- `PersonDatasetHandle` 中的若干 `NotImplementedError` 方法可按需求实现（随机采样、流式读取等）。
- `_cvt_bgr_to_rgb` 建议真正进行 BGR→RGB 转换，或更名以避免误导。
- 将 bucket 名称与表名从硬编码改为可配置项。

---

## 版权与许可

本仓库未包含明确的开源许可证说明。若需对外发布或复用，请在项目根目录补充 LICENSE 并遵循相关条款。
