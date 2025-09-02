# Installation

This guide will help you set up the AIGC GUI development environment on your local machine.

## Prerequisites

- Python 3.13.5+ (managed via Pixi)
- Git
- Sufficient disk space for model storage (20GB+ recommended)

## Environment Setup

### Using Pixi (Recommended)

The project uses Pixi for dependency management. Pixi will handle Python version and environment setup automatically.

```bash
# Clone the repository
git clone <your-repo-url>
cd aigc-gui

# Install Pixi (if not already installed)
# Visit: https://pixi.sh/latest/

# Install dependencies
pixi install
```

### Manual Installation

If you prefer manual setup:

```bash
# Ensure Python 3.13.5+ is installed
python --version

# Install dependencies
pip install -r requirements.txt
```

## Model Setup

The application requires pre-trained Stable Diffusion models. Configure your model paths in the application:

1. Download your preferred models to a local directory
2. Update the `basedir` path in `main.py` to point to your model storage location
3. Ensure the model directory structure matches the expected format

### Supported Models

**SD 1.5 Models:**
- Stable Diffusion v1.5
- DreamShaper SD1.5
- Realistic Vision SD1.5

**SDXL Models:**
- SDXL Base v1.0
- DreamShaper XL
- Juggernaut XL
- Proto Vision XL
- Night Vision XL

## Database Setup

The application uses MongoDB for data storage:

```bash
# Install MongoDB locally or use a cloud service
# Update connection settings in your environment configuration

# Create a .env file with your MongoDB connection string
echo "MONGODB_URL=mongodb://localhost:27017/aigc_gui" > .env
```

## Verification

Verify your installation by running the test suite:

```bash
# Run unit tests
python run-unit-test.py

# Test basic model functionality
python test-anything.py
```

## Next Steps

- [Quick Start Guide](quick-start.md) - Get started with your first image generation
- [Configuration](configuration.md) - Customize your setup
- [API Reference](../api/models.md) - Explore the API documentation