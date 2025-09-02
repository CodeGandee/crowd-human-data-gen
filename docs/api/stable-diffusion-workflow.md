# Stable Diffusion Workflow Architecture

This document explains the code working mechanism of `StableDiffusionProcessor.py`, detailing the complete data flow from "retrieving data from database/object storage → constructing inpaint inputs → calling SDXL pipeline → blending and writing back".

## Overview

**Objective:** Perform local inpainting on person regions in scenes using SDXL-based models, generating clothing/styles that match prompts while maintaining background consistency and natural blending. Write both person-level and scene-level results back to database and MinIO.

**Dependencies:**
- **MinIO** (Object Storage): Stores input materials and generation results
- **PostgreSQL**: Stores file mappings and task/person/result metadata  
- **Models**: `StableDiffusionXLBase`, ControlNet (Canny/Depth), IP-Adapter (reference image encoder and weights)
- **CUDA Environment**: Script defaults to `gpu_id = 7` (configurable)

## Core Classes and Responsibilities

### `SceneData`
- **Background Image** (RGB): Scene background
- **`mask_info`**: Person ID → color mapping dictionary  
- **Scene Mask** (`scene_mask`): Colored mask with unique color per person
- **Person Instance List** (`person_list`): List of person instances in scene

### `PersonData`  
- **`inner_mask`**: Single-person segmentation mask (grayscale)
- **`outer_mask`**: Expanded mask based on inner mask, aligned and adjusted to square bounding box
- **`depth_image`**: uint8 control image generated from depth `.npy` file
- **`combined_image`**: RGB image with person pasted back onto background (used as inpaint source)
- **`occ_mask_bool`**: Scene-level occupancy mask for blending person inpaint results back into full image

### `StableDiffusionProcessor`
- **Resource Initialization**: MinIO, PostgreSQL, `StableDiffusionXLBase`
- **Data Loading**: Query tasks from DB, retrieve images/masks/depth from MinIO via `filepath` table
- **Data Preparation**: Generate `outer_mask`, depth control images, construct inpaint inputs
- **Processing**: Call `sdxl.inpaint(...)` for each `PersonData`
- **Blending & Storage**: Merge person results and store both person/scene results to MinIO/DB

## Data Flow and Key Steps

The following steps are implemented by the `generate()` method:

### 1. Load Random Task
- `load_task()`: Randomly select one record from `task` table

### 2. Assemble Scene Data  
- `load_scene_data_from_task(task)`:
  - `mask_info_path` → MinIO: Deserialize to dict (person ID → RGB color)
  - `background_image_path` → MinIO: Decode to `background_rgb` 
  - `scene_mask_path` → MinIO: Colored mask image (unique color per person)
  - `person_instance_ids_path` → MinIO: Deserialize to instance ID list

### 3. Build PersonData for Each Person
- `get_person_data_by_instance_id(instance_id, scene_data)`:
  - Query `person` table: Get `mask_path`, `rgb_path`, `depth_path` for instance
  - `occ_mask_bool`: Find person pixels in `scene_mask` using `mask_info` colors → boolean mask
  - `inner_mask`: Read from MinIO, convert to grayscale, threshold >0 = True
  - `outer_mask`: Call `_calculate_outer_mask(inner_mask)` for expansion and alignment
  - `depth_image`: Generate control image from `.npy` (float32) file
  - `combined_image`: Copy `background_rgb`, paste person's RGB back to corresponding position

### 4. SDXL Inpainting
For each `PersonData`:
- **Dynamic Prompts**: 
  - `parse_dynamic_prompts` + `random_sample_dynamic_prompts`
  - Fixed negative prompts suppress "low quality/blurry/exposed" content
- **`sdxl.inpaint(...)` Core Parameters**:
  - `source_image`: `combined_image`
  - `inner_mask`: Person's precise mask (boolean)
  - `outer_mask`: Expanded/aligned bounding box (boolean)  
  - `depth_image`: Depth control image
  - **ControlNet**: `canny_weight`, `depth_weight`
  - **IP-Adapter**: `ipadapter_images=[background_rgb]`, `ipadapter_weights=[0.5]`
  - **Sampling**: `inference_steps=20`, `denoise_strength=0.7`
- **Returns**: `sd_image` (full-size RGB after inpainting), `sd_res` (person-level result/process image)
- **Blending**: `output_image = _merge_image_by_mask(sd_image, output_image, occ_mask_bool)`
- **Store Person Result**: `_upload_person_image_to_database(sd_res, 'taskinfos', image_info)` with metadata (prompt, steps, weights, etc.)

### 5. Scene-Level Refinement
- Re-iterate through each person, call `sdxl.inpaint(...)` on `output_image` for light refinement:
  - `denoise_strength=0.2`, `canny/depth_weight=0.2`, `ipadapter_weights=[0.3]`

### 6. Store Scene Results
- `_upload_scene_image_to_database(output_image, 'taskinfos', scene_image_info)`

## Key Algorithm Details

### Outer Mask Calculation
**Function:** `_calculate_outer_mask(inner_mask, x_expand_ratio=1/3, y_expand_ratio=1/3, divided_by=8)`

**Steps:**
1. Calculate bounding rectangle (xywh) of `inner_mask`
2. Expand by given ratios in X/Y directions for looser region  
3. Adjust rectangle width/height to be divisible by `divided_by` (for model size alignment)
4. Center and expand to square (take longer side) for consistent generation
5. Draw rectangle region on blank canvas to get `outer_mask`

**Rationale:** Expansion and alignment improve inpaint stability and quality, avoiding conflicts between person edges and context.

### Depth Control Image Generation
**Function:** `_generate_depth_ctrl_map(depth_data, mask)`

**Steps:**
1. Load `depth_data` from `.npy` (float32); replace 0 values with minimum valid value (avoid missing data)
2. Linear normalization to [0, 255] and convert to uint8
3. Invert values (white foreground 255 = closer, for ControlNet compatibility)  
4. Apply bitwise AND with `inner_mask` to keep only person region depth

## SDXL Pipeline Integration

Core `StableDiffusionXLBase.inpaint(...)` parameters used in this script:

- **`source_image`**: Full RGB image to be inpainted
- **`inner_mask`**: Strong constraint region for person foreground (boolean)
- **`outer_mask`**: Outer bounding box encompassing inner mask (boolean), guides network attention to broader context
- **`depth_image`**: Depth control image (uint8)
- **`positive_prompts`** / **`negative_prompts`**: Text prompts
- **`inference_steps`** / **`denoise_strength`**: Sampling and inpainting strength
- **`canny_weight`** / **`depth_weight`**: ControlNet influence
- **`ipadapter_images`** / **`ipadapter_weights`**: Reference images and weights (uses background image for style consistency)

**Returns:** Full-size `sd_image` and person-level `sd_res` (defined by pipeline).

## Data Storage and Tracking

### Person-Level Storage
- Upload `sd_res` to MinIO (bucket: `taskinfos`) and record in `sd_person` table:
  - `person_image_id`, `scene_image_id`, `task_id`, `person_instance_id`, `metadata` (JSON parameter summary)
- Insert `file_id -> storage_path` mapping in `filepath` table

### Scene-Level Storage  
- Upload final `output_image` to MinIO and record in `sd_scene` table linking `scene_image_id` with `task_id/scene_id`

Use `scene_image_id` and `person_image_id` for person-scene association tracking.

## Code Considerations and Suggestions

### GPU Selection
- File header sets fixed `gpu_id = 7` - adjust to 0/1/... based on your machine's available CUDA devices

### Color Space
- Background image processing includes `cv2.cvtColor(background_image_bgr, cv2.COLOR_RGB2BGR, background_image_bgr)` - this should typically be `COLOR_BGR2RGB` for OpenCV conventions. Verify based on actual display results.

### Refinement Stage Variable Scope
In the refinement loop:
```python
for person in persons_data:
    output_image = self.sdxl.inpaint(
        source_image=output_image,
        outer_mask=outer_mask_boolean,
        inner_mask=inner_mask_boolean,
        depth_image=person.depth_image,
        ...)
```
`inner_mask_boolean/outer_mask_boolean` are not redefined in this loop and may reference variables from previous loop. Consider using `person.inner_mask > 0` and `person.outer_mask > 0` (reference SD15 implementation).

### Debug Output
Code includes debug image saving to Linux paths `/data1/...`. For Windows environments, change to local writable paths or comment out.

## Main Entry and Model Configuration

Script bottom (`__main__`) example:
- **MinIO/PG connection parameters**: Replace IP/port/credentials as needed
- **Model paths**:
  - `sdxl/realismEngineSDXL_v30VAE`
  - `control_sdxl_canny`, `control_sdxl_depth`  
  - IP-Adapter weights and encoder
- **Execution**:
  1. `sdpr = StableDiffusionProcessor(minio_config, postgres_config)`
  2. `sdpr.init_sdxl(...)`
  3. `sdpr.generate()`
  4. Close connections

## Failure Cases and Edge Conditions

- **Empty Masks**: `_get_bounding_box_xywh` throws exception ("no object found") - ensure `inner_mask` is valid
- **Missing Data**: Person/task not found in database will throw errors - ensure `task/person` tables and MinIO file mappings are complete
- **Zero Depth Values**: Already handled with minimum valid value filling - consider improved preprocessing for noisy depth data
- **Memory/Performance**: 
  - Reduce resolution, steps, or disable control branches (Canny/Depth/IP-Adapter) to reduce resource usage

## Typical Execution Timeline

1. Load task and scene data
2. Iterate through persons: prepare inner/outer/depth/combined images
3. SDXL inpaint (with ControlNet and IP-Adapter)
4. Blend person results into full image using `occ_mask_bool`
5. Optional refinement (light artifact removal, style unification)  
6. Upload person and scene results to MinIO and write to database

---

For more detailed setup instructions, Windows commands, and configuration examples, see [Stable Diffusion Processors Guide](../examples/stable-diffusion-processors.md).