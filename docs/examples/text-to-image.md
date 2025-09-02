# Text-to-Image Generation

Generate stunning images from text descriptions using various Stable Diffusion models.

## Basic Usage

### Simple Text-to-Image

```python
import requests

# Basic generation request
response = requests.post("http://localhost:8000/api/text-to-image", json={
    "prompt": "A serene mountain lake at sunset",
    "model": "stable_diffusion_v15",
    "width": 512,
    "height": 512,
    "steps": 20,
    "guidance_scale": 7.5,
    "seed": 42
})

if response.status_code == 200:
    with open("mountain_lake.png", "wb") as f:
        f.write(response.content)
    print("Image saved as mountain_lake.png")
```

### Advanced Parameters

```python
# Advanced generation with more control
advanced_request = {
    "prompt": "A photorealistic portrait of a person, highly detailed, professional photography",
    "negative_prompt": "blurry, low quality, distorted, cartoonish",
    "model": "realistic_vision_sd15",
    "width": 768,
    "height": 768,
    "steps": 30,
    "guidance_scale": 8.0,
    "sampler": "DPM++ 2M Karras",
    "seed": 12345,
    "batch_size": 2
}

response = requests.post("http://localhost:8000/api/text-to-image", json=advanced_request)
```

## Model Comparison

### SD 1.5 Models

```python
models_sd15 = {
    "stable_diffusion_v15": {
        "description": "Original Stable Diffusion model",
        "best_for": "General purpose, balanced results",
        "recommended_size": "512x512"
    },
    "dreamshaper_sd15": {
        "description": "Enhanced artistic model",
        "best_for": "Creative and artistic images",
        "recommended_size": "512x512"
    },
    "realistic_vision_sd15": {
        "description": "Photorealistic model",
        "best_for": "Realistic photos and portraits",
        "recommended_size": "512x768"
    }
}
```

### SDXL Models

```python
models_sdxl = {
    "stable_diffusion_xl": {
        "description": "High-resolution base model",
        "best_for": "High quality general images",
        "recommended_size": "1024x1024"
    },
    "dreamshaper_xl": {
        "description": "Artistic SDXL variant",
        "best_for": "Creative high-res artwork",
        "recommended_size": "1024x1024"
    },
    "juggernaut_xl": {
        "description": "Realistic SDXL model",
        "best_for": "Photorealistic high-res images",
        "recommended_size": "1024x1024"
    }
}
```

## Prompt Engineering

### Effective Prompts

```python
# Good prompt structure
good_prompt = (
    "A majestic eagle soaring over snow-capped mountains, "
    "golden hour lighting, highly detailed, professional wildlife photography, "
    "sharp focus, dramatic composition"
)

# Negative prompt to avoid common issues
negative_prompt = (
    "blurry, low quality, distorted, ugly, bad anatomy, "
    "extra limbs, watermark, signature"
)
```

### Style Modifiers

```python
style_examples = {
    "photorealistic": "photorealistic, highly detailed, professional photography, 8k uhd",
    "artistic": "digital art, concept art, trending on artstation, highly detailed",
    "cinematic": "cinematic lighting, dramatic composition, movie still, professional",
    "anime": "anime style, cel shading, vibrant colors, detailed anime art",
    "oil_painting": "oil painting, classical art style, renaissance, masterpiece"
}
```

## Parameter Guidelines

### Resolution Guidelines

```python
resolution_guide = {
    "SD 1.5": {
        "square": "512x512",
        "portrait": "512x768", 
        "landscape": "768x512",
        "max_recommended": "768x768"
    },
    "SDXL": {
        "square": "1024x1024",
        "portrait": "768x1344",
        "landscape": "1344x768", 
        "max_recommended": "1536x1536"
    }
}
```

### Steps and Guidance Scale

```python
parameter_guide = {
    "steps": {
        "fast": 15,      # Quick generation
        "balanced": 20,  # Good quality/speed balance
        "quality": 30,   # Higher quality
        "overkill": 50   # Diminishing returns beyond this
    },
    "guidance_scale": {
        "creative": 5.0,    # More creative freedom
        "balanced": 7.5,    # Standard setting
        "strict": 12.0,     # Strict prompt following
        "max": 20.0         # May cause artifacts
    }
}
```

## Batch Processing

```python
import asyncio
import aiohttp

async def generate_batch(prompts, model="stable_diffusion_v15"):
    """Generate multiple images concurrently"""
    async with aiohttp.ClientSession() as session:
        tasks = []
        for i, prompt in enumerate(prompts):
            task = generate_single(session, prompt, model, i)
            tasks.append(task)
        
        results = await asyncio.gather(*tasks)
        return results

async def generate_single(session, prompt, model, index):
    """Generate a single image"""
    request_data = {
        "prompt": prompt,
        "model": model,
        "width": 512,
        "height": 512,
        "steps": 20,
        "guidance_scale": 7.5
    }
    
    async with session.post("http://localhost:8000/api/text-to-image", 
                           json=request_data) as response:
        if response.status == 200:
            content = await response.read()
            filename = f"batch_image_{index}.png"
            with open(filename, "wb") as f:
                f.write(content)
            return filename
```

## Common Issues and Solutions

### Memory Issues

```python
# Reduce memory usage
low_memory_config = {
    "width": 512,
    "height": 512,
    "steps": 15,
    "batch_size": 1,
    "enable_vae_slicing": True,
    "enable_attention_slicing": True
}
```

### Quality Issues

```python
# Improve image quality
quality_config = {
    "steps": 30,
    "guidance_scale": 8.0,
    "sampler": "DPM++ 2M Karras",
    "use_karras_sigmas": True,
    "eta": 0.0
}
```

### Speed Optimization

```python
# Faster generation
speed_config = {
    "steps": 15,
    "guidance_scale": 6.0,
    "sampler": "LCM",
    "enable_model_cpu_offload": False
}
```

## Example Scripts

The project includes several example scripts:

- `example-sd-basic.py` - Basic text-to-image generation
- `example-sd15-model.py` - SD 1.5 specific examples
- `example-sdxl-model.py` - SDXL specific examples

## Next Steps

- [Image-to-Image](image-to-image.md) - Transform existing images
- [Inpainting](inpainting.md) - Fill missing parts of images
- [LoRA Adapters](lora.md) - Use fine-tuned model adaptations