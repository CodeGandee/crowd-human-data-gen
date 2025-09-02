# AIGC GUI Documentation

Welcome to the **AIGC GUI** project - an AI-Generated Content platform built on top of Stable Diffusion models for advanced image generation capabilities.

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

## API Reference

For detailed API documentation, see our [API Reference](api/models.md) section covering models, pipelines, and utilities.

## Contributing

We welcome contributions! Please see our [Contributing Guide](development/contributing.md) for details on how to get started.