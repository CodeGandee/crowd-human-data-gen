# RenderToolbox Documentation

RenderToolbox is a Python-based synthetic data generation toolkit for creating annotated training datasets from 3D scenes and character assets. It specializes in multi-character rendering with comprehensive annotations including RGB, depth, instance segmentation, head/face masks, occlusion visibility statistics, and OpenPose-style 2D keypoints.

## Getting Started

This section provides comprehensive guides for using RenderToolbox's TaskManager components:

### Core Documentation

- **[TaskManager Usage Guide](TaskManager_Usage.md)** - Complete guide for basic task generation and management using `TaskManager.py`
  - Environment setup and dependencies
  - Quick start examples
  - API reference for TaskManager, TaskConfig, and PersonLabel classes
  - Local and database storage options
  - Output structure and data formats

- **[COCO-style Data Generation](TaskManager_coco_like_data_Usage.md)** - Generate COCO-format annotated datasets using `TaskManager_coco_like_data.py`
  - COCO-style annotations with instance segmentation
  - OpenPose keypoint generation
  - Head/face mask extraction
  - Visibility statistics and occlusion handling
  - Database integration with MinIO + PostgreSQL

## Key Features

- **Multi-character 3D rendering** with realistic occlusion handling
- **Comprehensive annotations**: RGB, depth, instance segmentation, keypoints
- **Database integration**: PostgreSQL + MinIO object storage support  
- **COCO-format compatibility** for standard computer vision pipelines
- **Flexible deployment**: Local file output or cloud storage
- **Modular architecture** with clear separation of concerns

## Quick Reference

### Main Classes
- `TaskManager` - Main orchestrator for task generation and management
- `TaskConfig` - Container for complete task results and metadata  
- `PersonLabel` - Individual person annotation data

### Key Methods
- `TaskManager.create_task(scene_id, person_info)` - Generate task with specific scene and positions
- `TaskManager.genarate_task_by_position()` - Auto-sample scene positions and generate task
- `TaskConfig.save_to_local_path(path)` - Save complete task to local directory
- `TaskConfig.save_data_to_database(bucket, minio, postgres)` - Save to MinIO + PostgreSQL

## Configuration

Both TaskManager variants use the same configuration file:
```
config/database_config.yaml
```

This file contains connection settings for:
- MinIO object storage
- PostgreSQL databases (multiple instances for different data types)
- MongoDB (optional metadata storage)

---

For more information about the broader crowd-human data generation project, see the [main documentation](../index.md).