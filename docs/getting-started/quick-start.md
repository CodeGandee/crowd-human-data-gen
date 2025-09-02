# Quick Start

Get up and running with AIGC GUI in just a few minutes!

## Starting the Server

Once you have [installed](installation.md) the dependencies, start the FastAPI server:

```bash
# Using the run script
./run-server.bat  # On Windows
# or
bash run-bento-service.sh  # On Linux/Mac

# Or start directly with Python
python main.py
```

The server will start on `http://localhost:8000` by default.

## Your First Image Generation

### Text-to-Image

Generate an image from a text prompt:

```python
import requests

# Basic text-to-image request
response = requests.post("http://localhost:8000/api/text-to-image", json={
    "prompt": "A beautiful landscape with mountains and a lake",
    "model": "stable_diffusion_v15",
    "width": 512,
    "height": 512,
    "steps": 20,
    "guidance_scale": 7.5
})

# Save the generated image
with open("output.png", "wb") as f:
    f.write(response.content)
```

### Available Endpoints

The API provides several endpoints for different generation modes:

- `POST /api/text-to-image` - Generate images from text
- `POST /api/image-to-image` - Transform existing images
- `POST /api/inpaint` - Fill missing parts of images
- `POST /api/person-generation` - Generate person images
- `GET /api/models` - List available models

## Testing Examples

The project includes several example scripts to help you get started:

```bash
# Test basic Stable Diffusion functionality
python example-sd-basic.py

# Test image-to-image conversion
python example-local-edit.py

# Test IP adapters
python example-ip-adapters.py

# Test LoRA adapters
python example-lora-adapters.py
```

## Model Selection

Choose from various pre-configured models:

```python
# SD 1.5 models
models_sd15 = [
    "stable_diffusion_v15",
    "dreamshaper_sd15", 
    "realistic_vision_sd15"
]

# SDXL models
models_sdxl = [
    "stable_diffusion_xl",
    "dreamshaper_xl",
    "juggernaut_xl",
    "proto_vision_xl",
    "night_vision_xl"
]
```

## Configuration

Basic configuration can be done through environment variables or by modifying `my_config.py`.

## What's Next?

- Explore [Examples](../examples/text-to-image.md) for detailed usage patterns
- Check [API Reference](../api/models.md) for comprehensive endpoint documentation
- Learn about [Configuration](configuration.md) options