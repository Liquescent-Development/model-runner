# Docker Model CLI

A powerful command-line interface for managing and interacting with AI/ML models. This CLI plugin works with the model-runner server to provide a seamless experience for model management, inference, and distribution.

## Features
- **Run Models**: Execute models with prompts or in interactive chat mode with streaming support
- **List Models**: View all models available locally with detailed information
- **Package Models**: Convert GGUF files into OCI artifacts and push them to registries
- **Configure Models**: Set runtime flags and context sizes for models
- **Model Lifecycle**: Pull, push, tag, remove, and unload models
- **Status & Monitoring**: Check model status and view detailed information

## Installation

### macOS (Apple Silicon)

Visit the [releases page](https://github.com/Liquescent-Development/model-runner/releases) and download the latest `docker-model` binary from a `cli-latest-*` release.

Then install as a Docker CLI plugin:

```bash
# Create plugin directory
mkdir -p ~/.docker/cli-plugins

# Move downloaded binary to plugin directory
mv ~/Downloads/docker-model ~/.docker/cli-plugins/docker-model
chmod +x ~/.docker/cli-plugins/docker-model

# Verify installation
docker model --help
```

The plugin can also be used standalone (without Docker installed):
```bash
# Rename and use as standalone binary
mv ~/Downloads/docker-model model-cli
chmod +x model-cli

# Use directly
./model-cli --help
```

### Building from Source

1. **Clone the repo:**
   ```bash
   git clone https://github.com/Liquescent-Development/model-runner.git
   cd model-runner/cmd/cli
   ```
2. **Build and install:**
   ```bash
   make install
   ```

## Prerequisites

The CLI requires a running model-runner server. You can:

1. **Use Docker Desktop** (if installed) - model-runner is included
2. **Run standalone model-runner** - Download from [releases](https://github.com/Liquescent-Development/model-runner/releases) and start with:
   ```bash
   # Requires llama.cpp: brew install llama.cpp
   LLAMA_SERVER_PATH=/opt/homebrew/bin MODEL_RUNNER_PORT=12434 ./model-runner-darwin-arm64
   ```

## Usage
Run `docker model --help` or `./model-cli --help` to see all commands and options.

### Common Commands

**Model Management:**
- `docker model list` — List available models
- `docker model pull MODEL` — Pull a model from a registry
- `docker model push MODEL` — Push a model to a registry
- `docker model rm MODEL` — Remove a model
- `docker model tag SOURCE TARGET` — Tag a model
- `docker model inspect MODEL` — Show detailed model information

**Running Models:**
- `docker model run MODEL [PROMPT]` — Run a model with a prompt or enter chat mode
- `docker model unload MODEL` — Unload a model from memory
- `docker model configure MODEL [flags]` — Configure model runtime settings

**Packaging & Distribution:**
- `docker model package --gguf <path> --push <target>` — Package and push a GGUF model

**Status & Monitoring:**
- `docker model status` — Check runner and model status
- `docker model df` — Show disk usage

## Examples

### Interactive Chat
```bash
# Run with a single prompt
docker model run gemma3 "What is the capital of France?"

# Enter interactive chat mode
docker model run gemma3
> Tell me a joke.
```

### Package and Distribute a Model
```bash
# Package a GGUF file and push to registry
docker model package --gguf ./my-model.gguf --push myorg/my-model:latest

# With license and context size
docker model package --gguf ./model.gguf \
  --license "Apache-2.0" \
  --context-size 8192 \
  --push myorg/model:v1
```

### Model Management
```bash
# Pull a model
docker model pull ai/gemma3

# List all models
docker model list

# Remove a model
docker model rm ai/gemma3:latest
```

## Development
- **Run unit tests:**
  ```bash
  make unit-tests
  ```
- **Generate docs:**
  ```bash
  make docs
  ```

## License
[Apache 2.0](LICENSE)

