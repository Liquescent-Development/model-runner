# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a fork of Docker's model-runner repository with modifications to improve model name handling and display. The model-runner is a Go-based system that manages, runs, and serves AI models (primarily LLMs) using Docker. It includes both a server component (model-runner) and a CLI client (model-cli).

## Architecture

### Core Components

**Model Runner (Server)** - `main.go`
- Entry point that initializes the HTTP server (Unix socket or TCP)
- Sets up the inference backend (llama.cpp), model manager, and scheduler
- Configurable via environment variables (see Environment Variables section)

**Model Manager** - `pkg/inference/models/`
- Handles model lifecycle: pull, store, list, remove, tag operations
- Integrates with OCI registries for model distribution
- Uses `distribution` package for pulling/pushing models
- Manages model metadata and storage in `~/.docker/models` (or `MODELS_PATH`)

**Inference Backend** - `pkg/inference/backends/llamacpp/`
- Wraps llama.cpp server for model inference
- Handles GPU detection and server binary management
- Supports auto-updating llama.cpp server binaries from Docker Hub
- Runs inference processes in sandboxed environments (Darwin only)

**Scheduler** - `pkg/inference/scheduling/`
- Manages model loading/unloading and request routing
- Handles concurrent inference requests
- Integrates with backends and memory estimation

**Distribution System** - `pkg/distribution/`
- Handles packaging models as OCI artifacts
- Supports GGUF and safetensors formats
- Bundle format: models stored in `model/` subdirectory with config at bundle root
- Parallel downloads for large files

**CLI Client** - `cmd/cli/`
- Docker CLI plugin (`docker model ...`) or standalone (`model-cli`)
- Communicates with model-runner server via HTTP/Unix socket
- Commands: run, pull, push, list, rm, tag, inspect, configure, etc.

### Key Packages

- `pkg/sandbox/` - Darwin-specific process sandboxing using sandbox-exec
- `pkg/gpuinfo/` - GPU detection (Metal on macOS, CUDA/ROCm on Linux)
- `pkg/metrics/` - Prometheus metrics aggregation from llama.cpp servers
- `pkg/middleware/` - CORS and other HTTP middleware

## Building and Testing

### Build Commands

**Main model-runner server:**
```bash
make build                    # Build server binary
make run                      # Build and run server
make test                     # Run all tests
make clean                    # Remove build artifacts
```

**CLI client:**
```bash
cd cmd/cli
make build                    # Build CLI binary as 'model-cli'
make install                  # Build and link to ~/.docker/cli-plugins/docker-model
make unit-tests               # Run CLI tests with race detection
```

**Model distribution tool:**
```bash
make model-distribution-tool  # Build mdltool binary
```

### Running Tests

**All tests:**
```bash
go test -v ./...
```

**Single test:**
```bash
go test -v ./path/to/package -run TestName
```

**CLI tests with race detection:**
```bash
cd cmd/cli && go test -race -v ./...
```

**Single package:**
```bash
go test -v ./pkg/inference/models
```

### Docker Build

```bash
make docker-build                                    # Build Docker image
make docker-run PORT=8080 MODELS_PATH=/path/to/models  # Run in container
```

## Environment Variables

**Server Configuration:**
- `MODEL_RUNNER_PORT` - TCP port to listen on (default: Unix socket)
- `MODEL_RUNNER_SOCK` - Unix socket path (default: model-runner.sock)
- `MODELS_PATH` - Model storage directory (default: ~/.docker/models)
- `LLAMA_SERVER_PATH` - Path to llama.cpp server binaries (default: /Applications/Docker.app/Contents/Resources/model-runner/bin)
  - **For Homebrew installations:** Set to `/opt/homebrew/bin` (Apple Silicon) or `/usr/local/bin` (Intel)
  - **For Docker Desktop:** Use default or set to Docker.app path
- `LLAMA_ARGS` - Additional arguments for llama.cpp (e.g., "--verbose --ctx-size 2048")
- `LLAMA_SERVER_VERSION` - Desired llama.cpp server version (default: latest)
- `DISABLE_SERVER_UPDATE` - Disable auto-update of llama.cpp server
- `DISABLE_METRICS` - Disable Prometheus metrics endpoint
- `MODEL_RUNNER_RUNTIME_MEMORY_CHECK` - Enable runtime memory checking (default: off)
- `DEBUG` - Enable debug logging
- `DO_NOT_TRACK` - Disable telemetry

**CLI Configuration:**
- `MODEL_RUNNER_HOST` - Server URL (default: Unix socket or http://localhost:12434)

## Model Name Normalization

This fork implements comprehensive model name normalization to improve UX:

**Normalization Rules** (`pkg/inference/models/manager.go:NormalizeModelName`):
- Bare names → `ai/{name}:latest` (e.g., `gemma3` → `ai/gemma3:latest`)
- Names with tag → `ai/{name}:{tag}` (e.g., `gemma3:v1` → `ai/gemma3:v1`)
- Names with org → `{org}/{name}:latest` (e.g., `myorg/model` → `myorg/model:latest`)
- HuggingFace models → lowercase (e.g., `hf.co/Bartowski/Model` → `hf.co/bartowski/model:latest`)
- Fully qualified names → unchanged

**Display Stripping** (`cmd/cli/commands/utils.go:stripDefaultsFromModelName`):
- `ai/{name}:latest` → `{name}` for display
- `ai/{name}:{tag}` → `{name}:{tag}` for display
- `{org}/{name}:latest` → `{org}/{name}` for display

All CLI commands that accept model names apply normalization before making API calls.

## API Structure

**Model Management** - `/models`
- `GET /models` - List models
- `POST /models/create` - Pull/create model
- `GET /models/{name}` - Inspect model
- `DELETE /models/{name}` - Remove model
- `POST /models/{name}/push` - Push model

**Inference** - `/engines/{backend}/v1`
- `POST /engines/llama.cpp/v1/chat/completions` - Chat completions (OpenAI-compatible)
- `GET /engines/llama.cpp/v1/models` - List loaded models

**Metrics** - `/metrics`
- Prometheus-format metrics from llama.cpp servers

## Development Notes

### Port Configuration
- Default Docker Desktop port: 12434
- Test/development port: 13434 (avoids conflicts)
- Configure via `MODEL_RUNNER_PORT` environment variable

### Testing Against Local Server
```bash
# Terminal 1: Start server
MODEL_RUNNER_PORT=13434 ./model-runner

# Terminal 2: Use CLI
MODEL_RUNNER_HOST=http://localhost:13434 ./model-cli list
```

### Running with Homebrew llama.cpp

If you have llama.cpp installed via Homebrew, you need to set the path:

```bash
# Install llama.cpp via Homebrew
brew install llama.cpp

# Run model-runner pointing to Homebrew installation
LLAMA_SERVER_PATH=/opt/homebrew/bin MODEL_RUNNER_PORT=12434 ./model-runner
```

**Verification:**
The logs should show:
- `LLAMA_SERVER_PATH: /opt/homebrew/bin`
- `No digest file found - assuming custom llama.cpp installation, skipping version check`
- `installed llama-server with gpuSupport=true` (if Metal is available)

### Bundle Format (Distribution)
Models are packaged with files in `model/` subdirectory:
- `model/model.gguf` or `model/model.safetensors` - Model weights
- `model/model.mmproj` - Multi-modal projector (optional)
- `model/template.jinja` - Chat template (optional)
- `config.json` - Runtime config (bundle root)

Old bundles without `model/` subdirectory are automatically recreated.

### GPU Support
- **macOS**: Metal (via IOKit, requires specific sandbox permissions)
- **Linux**: CUDA/ROCm (llama.cpp variant selection)
- **Windows**: CPU only (currently)

### Sandboxing (macOS only)
llama.cpp processes run in Darwin sandbox (`pkg/sandbox/sandbox_darwin.go`):
- Network access restricted to IPC sockets
- File access limited to model storage and working directory
- GPU/Metal access explicitly allowed via IOKit
- Camera/microphone access denied

## CI/CD

### GitHub Actions

**build-macos.yml** - Builds model-runner for macOS ARM64
- Triggers on push/PR to main when Go files change
- Builds native macOS ARM64 binary with CGO enabled
- Uploads artifacts for 30 days
- Creates pre-release on main branch commits with binary attached
- Download from: GitHub Releases → latest-{commit-sha}

**cli-build.yml** - Builds model-cli for multiple platforms
- Triggers on push/PR to main when CLI files change
- Builds for: darwin-arm64, windows-amd64, windows-arm64, linux-amd64, linux-arm64
- Uses `make release` target
- Artifacts retained for 2 days

**ci.yml** - Main CI pipeline
- Runs tests with race detection
- Validates go mod tidy
- Runs shellcheck validation
- Builds distribution tools

**release.yml** - Manual Docker image release (workflow_dispatch)
- Builds multi-platform Docker images for CE
- Supports CPU and CUDA variants
- Tags as latest (optional)
