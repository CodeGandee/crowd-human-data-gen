# Crowd Human Data Generation Documentation

This repository contains documentation for three complementary projects for synthetic data generation and AI-powered image generation:

## AIGC GUI
An AI-Generated Content platform built on top of Stable Diffusion models for advanced image generation capabilities.

## RenderToolbox
A Python-based synthetic data generation toolkit for creating annotated training datasets from 3D scenes and character assets.

## SyntheticDataLoader
A comprehensive PostgreSQL + MinIO data loading toolkit designed for efficient retrieval and management of synthetic human datasets, supporting both traditional UUID-based and modern SQLAlchemy ID-based loading approaches.

## Overview

AIGC GUI is a comprehensive FastAPI-based web application that provides various AI-powered image generation features including:

- **Text-to-Image Generation**: Create images from textual descriptions
- **Image-to-Image Transformation**: Transform existing images based on prompts
- **Inpainting**: Fill in missing parts of images intelligently
- **LoRA Adapters**: Fine-tuned model adaptations for specific styles
- **IP Adapters**: Image prompt adapters for style transfer
- **Multiple Model Support**: Support for SD 1.5 and SDXL models

## Key Features

### 🎨 Multiple Generation Modes
- Text-to-image generation with various models
- Image-to-image transformation
- Inpainting and outpainting capabilities
- Person generation and refinement pipelines

### 🔧 Model Support
- **Stable Diffusion v1.5**: `stable_diffusion_v15`, `dreamshaper_sd15`, `realistic_vision_sd15`
- **SDXL Models**: `stable_diffusion_xl`, `dreamshaper_xl`, `juggernaut_xl`, `proto_vision_xl`, `night_vision_xl`
- LoRA and IP adapter support for model customization

### 🚀 Performance Optimized
- Built with FastAPI for high-performance API endpoints
- MongoDB integration with GridFS for large file storage
- Background task processing for long-running operations
- BentoML integration for model serving

### 🔌 Developer Friendly
- RESTful API design
- Comprehensive testing suite
- Modular pipeline architecture
- Easy model configuration and switching

### 🗄️ Data Management Integration
- **PostgreSQL + MinIO Architecture**: Seamless integration with multi-database environments
- **Flexible Loading Patterns**: Support for both UUID-based and ID-based sample retrieval
- **Rich Metadata Support**: Comprehensive scene and person-level component handling
- **PyTorch Integration**: Native Dataset/DataLoader wrappers for ML workflows

## Quick Start

```bash
# Install dependencies
pip install -r requirements.txt

# Start the server
python main.py
```

## Project Structure

```
aigc-gui/
├── main.py                 # FastAPI application entry point
├── models.py              # Data models and schemas
├── pyaigc/               # Core AIGC package
├── base_sd_model.py      # SD 1.5 base models
├── base_sdxl_model.py    # SDXL base models
├── pipeline.py           # Image generation pipelines
├── utils/                # Utility functions
├── test/                 # Test suites
├── sample-data/          # Sample images and data
└── docs/                 # Documentation (you are here!)
```

## Getting Started

Ready to dive in? Check out our [Installation Guide](getting-started/installation.md) to set up your development environment, or jump straight to the [Quick Start](getting-started/quick-start.md) for a hands-on introduction.

## Examples

Explore our comprehensive examples:

- [Text to Image Generation](examples/text-to-image.md)
- [Image to Image Transformation](examples/image-to-image.md) 
- [Inpainting Examples](examples/inpainting.md)
- [LoRA Adapter Usage](examples/lora.md)
- [IP Adapter Implementation](examples/ip-adapter.md)

## RenderToolbox Documentation

RenderToolbox provides comprehensive 3D scene rendering and synthetic data generation capabilities for computer vision training datasets:

**📖 [Complete RenderToolbox Guide](rendertoolbox/README.md)** - Comprehensive overview and getting started guide

### Quick Access
- [TaskManager Usage Guide](rendertoolbox/TaskManager_Usage.md) - Basic task generation and management (`TaskManager.py`)
- [COCO-style Data Generation](rendertoolbox/TaskManager_coco_like_data_Usage.md) - COCO-format datasets (`TaskManager_coco_like_data.py`)

### Key Features
- **Multi-character 3D rendering** with realistic occlusion handling
- **COCO-format compatibility** with instance segmentation and keypoints  
- **Database integration** using PostgreSQL + MinIO object storage
- **Comprehensive annotations**: RGB, depth, head/face masks, visibility stats
- **SyntheticDataLoader Integration**: Efficient data loading with SQLAlchemy and traditional UUID-based approaches

## SyntheticDataLoader

**📖 [SyntheticDataLoader Complete Guide](SyntheticDataLoaderSQLAlchemy.md)** - Comprehensive documentation for the data loading toolkit

### Key Features
- **Multi-Database Architecture**: Connects to datasets, taskinfos, sceneinfos, and render PostgreSQL databases
- **Flexible Loading Approaches**: 
  - SQLAlchemy-based loading with 1-based ID indexing (recommended)
  - Traditional UUID-based loading for legacy compatibility
- **MinIO Integration**: Seamless binary file retrieval from object storage
- **Rich Component Structure**: Scene and person-level data with comprehensive metadata
- **PyTorch Dataset Support**: Native integration with PyTorch training pipelines
- **Static Sample Tables**: Optimized aggregated tables for efficient sequential access

## API Reference

For detailed API documentation, see our [API Reference](api/models.md) section covering models, pipelines, and utilities.

## Contributing

We welcome contributions! Please see our [Contributing Guide](development/contributing.md) for details on how to get started.