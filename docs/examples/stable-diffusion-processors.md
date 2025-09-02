# Stable Diffusion Processors Usage Guide

This guide explains how to use three Stable Diffusion processor scripts for AIGC image generation:

- `usage/StableDiffusionProcessor.py` (SDXL-based)
- `usage/StableDiffusion15Processor.py` (SD 1.5-based)  
- `usage/StableDiffusion15Processor_COCO.py` (SD 1.5 COCO variant)

These scripts read task data (background images, segmentation masks, person instances), perform local inpainting using Stable Diffusion pipelines, and write generated results back to object storage (MinIO) and database (PostgreSQL).

## Prerequisites

### Environment Setup
- Python environment with project dependencies installed
- In repository root (Windows/cmd):
  ```bat
  cd /d d:\Code\project\aigc-gui
  pip install -r requirements.txt
  ```
- Available NVIDIA GPU (scripts use `torch.cuda.set_device(gpu_id)` by default)
- Accessible MinIO and PostgreSQL instances with corresponding tables and data
- Local SD/ControlNet/IP-Adapter model files with correct paths configured in scripts

> **Note:** All three scripts set `gpu_id = 7` at file header. Adjust to 0 or other available device ID based on your machine configuration.

## Data Requirements

All three scripts depend on the following data resources:

### MinIO Storage Buckets and File Mappings
- **Storage Buckets:**
  - `taskinfos`: Person-related resources, generation intermediate/final results
  - `sceneinfos`: Scene/background related resources (used by SDXL script)
- **Data Table `filepath`**: Stores `file_id -> storage_path` mappings for locating MinIO objects

### PostgreSQL Database Tables
- **Standard SD/SDXL:**
  - `task`: Task table (contains `mask_info_path`, `background_image_path`, `scene_mask_path`, `person_instance_ids_path` fields)
  - `person`: Person table (`instance_id`, `mask_path`, `rgb_path`, `depth_path`, etc.)
  - **Result tables**: `sd_scene` (scene-level output), `sd_person` (person-level results)
- **COCO Variant:**
  - `task_coco`: Task table (similar fields to `task`, but background also stored in task storage)
  - `person_coco`: Person table
  - **Result tables**: `sd_coco_scene`, `sd_coco_person`

### Mask/Depth Format Requirements
- **Masks** (`inner_mask`/`scene_mask`): Stored as PNG (or equivalent) in MinIO; read and converted to single-channel grayscale (>0 = True)
- **Depth Data** (`depth_path`): Stored as `np.float32` `.npy` files; converted to 0-255 `uint8` depth control images

## Model Path Configuration

All three scripts call `init_sdxl(...)` or `init_sd15(...)` in `__main__` requiring the following model paths (examples show Linux paths - adjust to your local paths):

### SDXL (`usage/StableDiffusionProcessor.py`)
```python
# Example model paths - adjust to your local installation
model_path = "/path/to/models/sdxl/realismEngineSDXL_v30VAE"
canny_ctrl_path = "/path/to/models/controlnet/control_sdxl_canny"  
depth_ctrl_path = "/path/to/models/controlnet/control_sdxl_depth"
ipa_model_path = "/path/to/models/ip-adapter/ip_adapter_sdxl/ip_adapter_sdxl.bin"
ipa_encoder_path = "/path/to/models/ip-adapter/ip_adapter_sdxl/image_encoder"
```

### SD 1.5 (Both SD 1.5 scripts)
```python
# Example model paths - adjust to your local installation  
model_path = "/path/to/models/sd15/realDream"  # or any SD1.5 model
canny_ctrl_path = "/path/to/models/controlnet/control_v11p_sd15_canny"
depth_ctrl_path = "/path/to/models/controlnet/control_v11f1p_sd15_depth" 
ipa_model_path = "/path/to/models/ip-adapter/ip_adapter_sd15/ip_adapter.bin"
ipa_encoder_path = "/path/to/models/ip-adapter/ip_adapter_sd15/image_encoder"
```

> **Windows Users:** Change paths to local drive paths, e.g., `D:\models\sdxl\...`

## Connection Configuration

Update MinIO/PostgreSQL connection parameters in `__main__` (defaults show `192.168.13.183` etc.):

```python
minio_config = {
    "host": "<minio_host>",
    "port": <minio_port>, 
    "username": "<minio_user>",
    "password": "<minio_pass>"
}

postgres_config = {
    "task": {
        "host": "<pg_host>", 
        "port": <pg_port>, 
        "username": "postgres", 
        "password": "example", 
        "database_name": "taskinfos"
    },
    "scene": {
        "host": "<pg_host>", 
        "port": <pg_port>, 
        "username": "postgres", 
        "password": "example", 
        "database_name": "sceneinfos"
    }
}
```

## Running the Scripts

Execute one of the following commands from repository root (Windows/cmd):

### SDXL Version
```bat
cd /d d:\Code\project\aigc-gui
python usage\StableDiffusionProcessor.py
```

### SD 1.5 Version  
```bat
cd /d d:\Code\project\aigc-gui
python usage\StableDiffusion15Processor.py
```

### SD 1.5 COCO Variant
```bat
cd /d d:\Code\project\aigc-gui
python usage\StableDiffusion15Processor_COCO.py
```

## Script Differences and Key Points

### SDXL: `StableDiffusionProcessor.py`
- **Pipeline Class**: `StableDiffusionXLBase`
- **Task Source**: `task` table (random selection of 1 record)
- **Person Source**: `person` table  
- **Result Storage**: `sd_scene`, `sd_person` tables
- **IP-Adapter Reference**: Uses full background image `scene_data.background_rgb` by default
- **Debug Output**: Script includes debug saving:
  ```python
  for person in persons_data:
      person.save_as_local_file('/data1/nieyujie/data/debug_data/debug_show')
  ```
  For Windows, modify to writable path or comment out to avoid `/data1/...` directory creation failures.

### SD 1.5: `StableDiffusion15Processor.py`  
- **Pipeline Class**: `StableDiffusionBase`
- **Task Source**: `task` table (currently locked to `id=11`, can change back to random)
- **Person Source**: `person` table
- **Result Storage**: `sd_scene`, `sd_person` tables
- **Additional Output**: Saves scene image to local directory (Linux example path):
  ```python
  cv2.imwrite('/data1/.../sd15images_768/{scene_image_uuid}.png', ...)
  ```
  Windows users should change to local writable path or comment out.

### COCO Variant: `StableDiffusion15Processor_COCO.py`
- **Pipeline Class**: `StableDiffusionBase`  
- **Task Source**: `task_coco` table
- **Person Source**: `person_coco` table
- **Result Storage**: `sd_coco_scene`, `sd_coco_person` tables
- **Background Image**: Read from `task_coco.background_image_path` (also stored in `taskinfos`/MinIO)
- **IP-Adapter Reference**: Randomly samples from local `'/data1/nieyujie/data/coco_samples'`
  - Windows users should change to existing local sample directory or use `scene_data.background_rgb`
- **Preview**: Uses `jpt.imshow` for image preview - comment out in non-interactive environments

## Configurable Parameters

All three scripts share similar adjustable parameters:

### Text Prompts
- `positive_prompts` and `negative_prompts` (supports dynamic prompt combinations via `utils/prompt_utils.py`)

### Sampling/Inpainting
- `inference_steps` (default 20)
- `denoise_strength` (local inpainting strength, default 0.7; lower during refinement)

### ControlNet
- `canny_weight`, `depth_weight`

### Blending
- `BlendingParam(GaussianBlurParam(radius=25), with_poisson_blending=False)`

### IP-Adapter  
- `ipadapter_images=[...]`
- `ipadapter_weights=[0.3 ~ 0.5]`

## Typical Workflow Summary

1. **Load Task Data**: Random/specified task reading from database - background image, scene masks, person instance lists and color mappings
2. **Per-Person Processing**:
   - Read `inner_mask`, `rgb`, `depth`, calculate `outer_mask`, depth control images
   - Paste person back onto background to get `combined_image`, perform inpainting within `outer_mask` region  
   - Upload person-level generation results (or intermediate images) to MinIO and record metadata in `sd_person`/`sd_coco_person`
3. **Scene Integration**: Blend all person results back into scene, optionally perform final scene-level refinement
4. **Final Storage**: Upload full scene image to MinIO and record association info in `sd_scene`/`sd_coco_scene`

## Troubleshooting

### Common Issues
- **Connection Failures** (MinIO/PG): Check IP/port/credentials, security groups/firewalls
- **Model Path Errors**: Ensure all directories exist with correct weight files; SDXL and 1.5 paths cannot be mixed
- **CUDA Device Unavailable**: Adjust `gpu_id`; use `nvidia-smi` to check memory; avoid concurrent GPU usage
- **Image Decoding Failures**: Confirm MinIO objects are valid PNG/JPG; `cv2.imdecode` returns `None` when data is corrupted
- **Windows Path Issues**: Change Linux paths in scripts to local writable paths, or comment out debug saving/display

## Quick Start Example (SDXL)

1. **Modify `usage/StableDiffusionProcessor.py`**:
   - Set `gpu_id = 0`
   - Set `model_root = r'D:\models'` (and update internal sub-paths)
   - Fill in correct `minio_config` and `postgres_config`

2. **Run**:
   ```bat
   cd /d d:\Code\project\aigc-gui
   python usage\StableDiffusionProcessor.py
   ```

3. **Monitor**: Check console logs and database/MinIO output records

For batch processing or custom task filtering, modify the SQL conditions in each script's `load_task(...)` method.

## Related Documentation

- [Stable Diffusion Workflow Architecture](../api/stable-diffusion-workflow.md) - Detailed technical documentation
- [Background Generation](background-generation.md) - SD 1.5 background generation examples
- [Text to Image](text-to-image.md) - Basic text-to-image generation patterns