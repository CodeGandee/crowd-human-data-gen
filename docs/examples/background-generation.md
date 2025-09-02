# Background Generation with SD 1.5

This guide explains how to use `usage/StableDiffusion15_gen_bg.py` to automatically generate "person-free" background images using SD 1.5 + IP-Adapter, upload results to MinIO, and register metadata in PostgreSQL.

## Overview

- **Text-to-Image Generation**: Based on `StableDiffusionBase` pipeline
- **IP-Adapter Style Guidance**: Uses reference images from local directory to guide style
- **Person Detection**: Uses `PersonDetector` to detect human presence; discards and regenerates if people are detected until achieving "person-free" backgrounds
- **Storage Integration**: Uploads generated background images to MinIO (bucket: `taskinfos`) and records dimensions/unique IDs in `sd_background` table, with `file_id -> storage_path` mappings in `filepath` table

## Prerequisites

### Environment Setup
- Install dependencies (from repository root):
  ```bat
  cd /d d:\Code\project\aigc-gui
  pip install -r requirements.txt
  ```
- Available NVIDIA GPU (recommended)
- Configure MinIO and PostgreSQL:
  - **MinIO bucket**: `taskinfos`
  - **PostgreSQL database** (example: `taskinfos`) with tables:
    - `sd_background(background_image_id TEXT PRIMARY KEY, height INT, width INT, ...)`
    - `filepath(file_id TEXT PRIMARY KEY, storage_path TEXT UNIQUE, ...)`

### Model and Reference Image Setup
- **SD 1.5 model directory** (e.g., `.../sd15/mixrealV2`)
- **IP-Adapter model and encoder directories**
- **Reference image directory** `ipa_images_dir` (containing JPG/PNG etc. images)

## Configuration File: background.yaml

The script reads configuration via `OmegaConf` from `usage/config/background.yaml`:

```yaml
minio_config:
  host: <minio_host>
  port: <minio_port>
  username: <minio_user>
  password: <minio_pass>

postgres_config:
  host: <pg_host>
  port: <pg_port>
  username: postgres
  password: example
  database_name: taskinfos

ipa_images_dir: D:/datasets/coco_samples   # Local reference image directory

StableDiffusion15:
  model: D:/models/sd15/mixrealV2
  ipa_model: D:/models/ip-adapter/ip_adapter_sd15/ip_adapter.bin
  ipa_encoder: D:/models/ip-adapter/ip_adapter_sd15/image_encoder

person_detector_threshold: 0.3
```

**Key Parameters:**
- **`ipa_images_dir`**: Directory from which to randomly select reference images for IP-Adapter
- **`person_detector_threshold`**: Person detection threshold - higher values are stricter

## Important: Update Script Configuration Path

`StableDiffusion15_gen_bg.py` loads configuration with a hardcoded path at module level:

```python
config_path = '/data1/.../usage/config/background.yaml'
```

**For Windows, change to your local path:**

```python
config_path = r'D:\Code\project\aigc-gui\usage\config\background.yaml'
```

Consider using relative paths or environment variables instead of hardcoded absolute paths for better portability.

## Quick Start (Windows/cmd)

1. **Edit `usage/config/background.yaml`** with your MinIO/PostgreSQL/model/reference image directory settings

2. **Modify `usage/StableDiffusion15_gen_bg.py`** - update `config_path` to your local path

3. **Run the script:**
   ```bat
   cd /d d:\Code\project\aigc-gui
   python usage\StableDiffusion15_gen_bg.py
   ```

4. **The script will generate 10 background images in a loop**, preview them, and upload to MinIO with database synchronization

## Workflow Details

### Prompt Generation
- **`StableDiffusionBGGenerator.sample_prompt()`** generates base positive/negative prompts:
  - **Positive**: Contains "high quality, background image, realistic style" etc., combined with random components from `random_gen.generate_background_prompt()`
  - **Negative**: Contains "low quality, blurry, people" etc., helping suppress person generation

### Reference Image Sampling
- **`sample_image(ipa_images_dir)`** randomly reads one image from local reference directory for IP-Adapter guidance

### Image Generation and Validation
- **`sd15.text_to_image(...)`** generates target size images (default 540x960); resizes with `cv2.resize` if dimensions don't match
- **`PersonDetector.detect(...)`** detects people presence; discards and retries if people detected
- **Upon achieving "person-free" image:**
  - Upload RGB image to MinIO (bucket: `taskinfos`)
  - Insert `background_image_id`, `height`, `width` into `sd_background` table  
  - Insert `file_id -> storage_path` mapping into `filepath` table

## Configurable Parameters

Adjustable in code or configuration:

### Generation Parameters
- **`target_hw`**: Default `(540, 960)` - change to `(H, W)` as needed for your use case
- **`inference_steps`**: Default `20`
- **`ipa_weight`**: Default `0.2` - adjust IP-Adapter influence (0~1)

### Prompt Customization
- Modify `base_prompt`/`negative_prompt` or `random_gen.generate_background_prompt()` strategy

### Person Detection
- Adjust `person_detector_threshold` or change detector threshold and model

### Batch Size
- Modify `for i in range(10)` at script end for different quantities

## Output and Verification

### MinIO Storage
- **`taskinfos` bucket** will contain new randomly-named objects (images)
- Corresponding `file_id` stored in `sd_background.background_image_id`

### PostgreSQL Records  
- **`sd_background`**: New records with dimensions and IDs
- **`filepath`**: New `file_id -> storage_path` mappings

### Preview
- Script uses `jpt.imshow(...)` for display - comment out related lines in headless/GUI-free environments

## Common Issues

### Configuration Problems
- **Config path error**: `OmegaConf.load` can't find `background.yaml` - verify `config_path` points to valid file
- **Empty reference directory**: `sample_image` will error if `ipa_images_dir` contains no images

### Connection Issues  
- **MinIO/PG connection failure**: Check IP/port/authentication, network connectivity, and database table existence

### Resource Constraints
- **Insufficient VRAM**: Reduce resolution, steps, or run on idle GPU
- **Windows path separators**: Use raw strings (`r''` prefix) or double backslashes in configuration and code paths

## Advanced Suggestions

### Portability Improvements
- Change `config_path` to read environment variable or command-line argument for cross-environment deployment
- Wrap generation and upload logic into functions for unit testing and reusability  
- Add failure retry and timeout controls for improved stability during long batch generation sessions

## Code Example: Basic Usage

```python
# Basic workflow (simplified)
from StableDiffusion15_gen_bg import StableDiffusionBGGenerator

# Initialize with config
generator = StableDiffusionBGGenerator(config)

# Generate single background
for attempt in range(max_attempts):
    # Generate image with random prompt and reference
    image = generator.generate_background_image(
        target_hw=(540, 960),
        inference_steps=20, 
        ipa_weight=0.2
    )
    
    # Check for people
    if not generator.person_detector.detect(image):
        # Upload and store if person-free
        generator.upload_and_store(image)
        break
```

This script provides a robust pipeline for generating high-quality, person-free background images suitable for downstream scene composition tasks.

## Related Documentation

- [Stable Diffusion Processors](stable-diffusion-processors.md) - Main SD processing workflows
- [Text to Image](text-to-image.md) - Basic text-to-image generation patterns
- [Stable Diffusion Workflow](../api/stable-diffusion-workflow.md) - Technical architecture details