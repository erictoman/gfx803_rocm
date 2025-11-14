# CI/CD for Docker Images

This repository uses GitHub Actions to automatically build and push Docker images for ROCm GFX803 support.

## Workflows

### 1. Build and Push Docker Images (`docker-build.yml`)

This workflow automatically builds and pushes all Docker images to GitHub Container Registry (GHCR).

#### Triggers
- **Push** to `main` or `develop` branches
- **Pull requests** to `main` branch
- **Version tags** (v*)
- **Manual dispatch** (workflow_dispatch)

#### Build Strategy

The workflow builds images in the correct order to handle dependencies:

1. **Base Image** (`Dockerfile_rocm64_base`)
   - Built first as all other images depend on it
   - Contains ROCm 6.4.0, compiled rocBLAS for GFX803
   - Tags: `$REGISTRY/$REPOSITORY-base:latest`

2. **Dependent Images** (Parallel build after base succeeds)
   - `Dockerfile_rocm64_ollama` - Ollama LLM server
   - `Dockerfile_rocm64_comfyui` - ComfyUI for image generation
   - `Dockerfile_rocm64_pytorch` - PyTorch with GFX803 support
   - `Dockerfile_rocm64_whisperx` - WhisperX for speech recognition
   - `Dockerfile_rocm64_llamacpp` - llama.cpp for local inference

#### Manual Builds

You can manually trigger builds from the Actions tab with the following options:

- **all** - Build all images (default)
- **base** - Build only the base image
- **ollama** - Build only Ollama image
- **comfyui** - Build only ComfyUI image
- **pytorch** - Build only PyTorch image
- **whisperx** - Build only WhisperX image
- **llamacpp** - Build only llama.cpp image

#### Image Tagging

Images are tagged with:
- `latest` - Latest build on default branch
- `<branch>-<sha>` - Short commit SHA prefix
- `<version>` - Full semantic version (e.g., v1.2.3)
- `<major>.<minor>` - Major.minor version (e.g., 1.2)

#### Registry

All images are pushed to GitHub Container Registry:
- Registry: `ghcr.io`
- Repository: `ghcr.io/<owner>/<repo>-<imagename>`

### 2. Validate Dockerfiles (`validate.yml`)

This workflow performs validation checks on Dockerfiles.

#### Checks Performed
- **Hadolint** - Dockerfile linting and best practices
- **Dependency verification** - Ensures base image exists for dependent Dockerfiles
- **Port verification** - Checks exposed ports documentation

#### Triggers
- **Push** to any branch
- **Pull requests** to `main` branch

## Prerequisites for Local Testing

Before building images locally:

1. Ensure you have Docker installed
2. Have at least 16GB RAM available
3. Have at least 50GB free disk space
4. Allow 1-3 hours for full build depending on hardware

## Local Build Commands

```bash
# Build base image first (takes 30-60 minutes)
docker build -f Dockerfile_rocm64_base . -t 'rocm6_gfx803_base:6.4'

# Build specific applications
docker build -f Dockerfile_rocm64_ollama . -t 'rocm64_gfx803_ollama:0.9.0'
docker build -f Dockerfile_rocm64_comfyui . -t 'rocm64_gfx803_comfyui:latest'
docker build -f Dockerfile_rocm64_pytorch . -t 'rocm64_gfx803_pytorch:2.6'
docker build -f Dockerfile_rocm64_whisperx . -t 'rocm64_gfx803_whisperx:latest'
```

## Pulling Pre-built Images

After CI builds complete, you can pull images from GHCR:

```bash
# Login to GHCR
echo $GITHUB_TOKEN | docker login ghcr.io -u <username> --password-stdin

# Pull images
docker pull ghcr.io/<owner>/<repo>-base:latest
docker pull ghcr.io/<owner>/<repo>-Dockerfile_rocm64_ollama:latest
# etc.
```

## Build Time Expectations

| Image | Build Time | Storage Required |
|-------|------------|------------------|
| Base | 30-60 min | ~20GB |
| Ollama | 60-90 min | ~25GB |
| ComfyUI | 2-3 hours | ~40GB |
| PyTorch | 2 hours | ~35GB |
| WhisperX | 2.5 hours | ~40GB |
| llama.cpp | ~10 min | ~5GB |

## Important Notes

⚠️ **Building these images requires significant resources:**
- 16GB+ RAM
- 50GB+ free disk space
- Fast internet connection (several GB download)
- 1-3 hours per image build time

The CI workflow uses GitHub-hosted runners which may time out for these long-running builds. For production use, consider:
1. Building images locally on powerful hardware
2. Using self-hosted runners with GPU support
3. Building selectively (only images you need)

## Troubleshooting

### Build Timeouts
If GitHub Actions times out during build:
- Use workflow_dispatch to build individual images
- Consider self-hosted runners for faster builds
- Build locally for development

### Storage Issues
- Enable buildkit cache: `DOCKER_BUILDKIT=1 docker build ...`
- Use `--no-cache` only when necessary
- Clean up old images regularly

### GPU Access
ROCm requires specific hardware support. GitHub hosted runners don't have AMD GPU access, so:
- Images are built without GPU testing
- Final testing should be done on local hardware with GFX803 GPU
- Consider using ROCm compatible cloud instances