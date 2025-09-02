# Configuration

Learn how to configure AIGC GUI for your specific needs.

## Environment Configuration

### Environment Variables

Create a `.env` file in the project root:

```env
# Database Configuration
MONGODB_URL=mongodb://localhost:27017/aigc_gui
MONGODB_DB_NAME=aigc_gui

# Model Configuration
MODEL_BASE_PATH=/path/to/your/models
DEFAULT_SD_MODEL=stable_diffusion_v15
DEFAULT_SDXL_MODEL=stable_diffusion_xl

# Server Configuration
HOST=0.0.0.0
PORT=8000
DEBUG=false

# Generation Defaults
DEFAULT_WIDTH=512
DEFAULT_HEIGHT=512
DEFAULT_STEPS=20
DEFAULT_GUIDANCE_SCALE=7.5
```

## Model Configuration

### Base Model Paths

Configure model locations in `main.py`:

```python
basedir = r'/your/model/directory'
C.ModelConfigs.BaseModelPaths = {
    # SD 1.5 Models
    C.SDModelKeys.stable_diffusion_v15: f'{basedir}/stable-diffusion/stable-diffusion-v1-5',
    C.SDModelKeys.dreamshaper_sd15: f'{basedir}/stable-diffusion/dreamshaper',
    C.SDModelKeys.realistic_vision_sd15: f'{basedir}/stable-diffusion/realistic-vision-v40',
    
    # SDXL Models  
    C.SDXLModelKeys.stable_diffusion_xl: f'{basedir}/stable-diffusion/sdxl-1.0-base-diffusers',
    C.SDXLModelKeys.dreamshaper_xl: f'{basedir}/stable-diffusion/DreamshaperXL',
    C.SDXLModelKeys.juggernaut_xl: f'{basedir}/stable-diffusion/JuggernautXL',
}
```

### Custom Model Keys

Add your own model configurations:

```python
class MySDModelKeys(C.SDModelKeys):
    stable_diffusion_v15 = "StableDiffusion_v15"
    dreamshaper_sd15 = "Dreamshaper_sd15"
    my_custom_model = "MyCustomModel_sd15"

C.SDModelKeys = MySDModelKeys
```

## Pipeline Configuration

### Generation Parameters

Customize default generation parameters:

```python
# In your configuration file
DEFAULT_CONFIG = {
    "width": 512,
    "height": 512, 
    "steps": 20,
    "guidance_scale": 7.5,
    "scheduler": "DPMSolverMultistepScheduler",
    "safety_checker": True
}
```

### LoRA Configuration

Configure LoRA adapters:

```python
LORA_CONFIG = {
    "enabled": True,
    "default_scale": 0.8,
    "max_adapters": 5,
    "cache_size": "2GB"
}
```

### IP Adapter Configuration

Set up IP adapter settings:

```python
IP_ADAPTER_CONFIG = {
    "enabled": True,
    "models": ["ip-adapter_sd15", "ip-adapter-plus_sd15"],
    "default_scale": 0.6
}
```

## Database Configuration

### MongoDB Settings

```python
MONGO_CONFIG = {
    "connection_timeout": 30000,
    "max_pool_size": 50,
    "gridfs_bucket": "images",
    "indexes": {
        "generations": ["user_id", "created_at"],
        "models": ["name", "version"]
    }
}
```

## Performance Configuration

### Memory Management

```python
PERFORMANCE_CONFIG = {
    "torch_dtype": "float16",  # or "float32" for better quality
    "enable_memory_efficient_attention": True,
    "enable_vae_slicing": True,
    "enable_vae_tiling": True,
    "max_batch_size": 4
}
```

### GPU Configuration

```python
GPU_CONFIG = {
    "device": "cuda",  # or "cpu", "mps"
    "mixed_precision": True,
    "compile_model": False  # Set True for PyTorch 2.0+
}
```

## Security Configuration

### API Security

```python
SECURITY_CONFIG = {
    "enable_cors": True,
    "allowed_origins": ["http://localhost:3000"],
    "rate_limiting": {
        "enabled": True,
        "requests_per_minute": 10
    }
}
```

## Logging Configuration

```python
import logging

LOGGING_CONFIG = {
    "level": logging.INFO,
    "format": "%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    "handlers": ["console", "file"],
    "log_file": "logs/aigc-gui.log"
}
```

## Development Configuration

### Debug Settings

```python
DEBUG_CONFIG = {
    "save_intermediate_images": True,
    "log_generation_params": True,
    "profile_performance": False,
    "mock_slow_operations": False
}
```

## Production Configuration

### Deployment Settings

```python
PRODUCTION_CONFIG = {
    "workers": 4,
    "max_requests": 1000,
    "timeout": 300,
    "keep_alive": 2,
    "preload_models": ["stable_diffusion_v15", "stable_diffusion_xl"]
}
```

For production deployment, consider using:

```bash
# Using Gunicorn
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker

# Using Docker
docker build -t aigc-gui .
docker run -p 8000:8000 aigc-gui
```

## Configuration Validation

The application automatically validates configuration on startup. Check logs for any configuration warnings or errors.

## Next Steps

- [API Reference](../api/models.md) - Explore available endpoints
- [Examples](../examples/text-to-image.md) - See configuration in action