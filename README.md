# Redhat-AI-Dev Llama Stack

[![Apache2.0 License](https://img.shields.io/badge/license-Apache2.0-brightgreen.svg)](LICENSE)
[![Llama Stack Version](https://img.shields.io/badge/Llama%20Stack-0.4.3-blue)](#version-table)
[![RHDH Release](https://img.shields.io/badge/RHDH%20Release-1.10-blueviolet)](#version-table)

> [!IMPORTANT]
> If you are looking for releases prior to RHDH 1.10, please see the [release-v0.1.x](https://github.com/redhat-ai-dev/llama-stack/tree/release-v0.1.x) branch.
>
> If you are looking for library mode, plase see the [dev](https://github.com/redhat-ai-dev/llama-stack/tree/dev) branch.

- [Version Table](#version-table)
- [Release Process](#release-process)
- [Provider Configuration](#provider-configuration)
- [Running Locally](#running-locally)
- [Configuring RAG Content](#configuring-rag-content)
- [Configuring Safety Guards](#configuring-safety-guards)
- [Running on a Cluster](#running-on-a-cluster)
- [Makefile Commands](#makefile-commands)
- [Updating/Formatting YAML Files](#updatingformatting-yaml-files)
- [Troubleshooting](#troubleshooting)

## Version Table

| Llama Stack Version | Llama Stack Image | Lightspeed Core Image | RAG Image |
| ---- | ---- | ---- | ---- |
| `0.4.3` | `registry.redhat.io/rhoai/odh-llama-stack-core-rhel9:v3.3` | `quay.io/lightspeed-core/lightspeed-stack:dev-20260226-ca21850` | `quay.io/redhat-ai-dev/rag-content:release-1.9-lls-0.4.3` |

`main` tracks one active release at a time. Historical releases are preserved in `rhdh-x.x` branches and Git tags.

## Release Process

Release and hotfix workflow is documented in [docs/RELEASE_PROCESS.md](./docs/RELEASE_PROCESS.md).

## Provider Configuration

Provider-specific setup and environment variable details live in [docs/PROVIDERS.md](./docs/PROVIDERS.md).

## Running Locally
> [!NOTE]
> You should ensure you have [Podman](https://podman.io/docs/installation) v5.4.1+ (recommended) or [Docker](https://docs.docker.com/engine/) v28.1.0+ with Compose support
>

Run `make get-rag` first so `./rag-content` exists locally.
Also ensure `./env/values.env` exists (copy from `./env/default-values.env`).

Start the local API stack for `0.4.3`:

```sh
make local-up
```

Disable Ollama and use `run-no-guard.yaml` instead:

```sh
make local-up WITH_OLLAMA=false
```

Stop services:

```sh
make local-down
```

By default (`WITH_OLLAMA=true`), `make local-up` uses:

- `llama-stack-configs/run.yaml`
- an Ollama container in compose (required for serving the safety model)
- compose enforces startup order: Ollama healthy -> Llama Stack healthy -> Lightspeed Core starts
- Ollama/Safety env vars from `env/values.env`

With `WITH_OLLAMA=false`, `make local-up` uses the base compose only:

- `llama-stack-configs/run-no-guard.yaml`
- no Ollama container (safety guards disabled)

## Configuring RAG Content

Pull the embeddings model and vector database locally:

```sh
make get-rag
```

This command fully replaces `./rag-content` on each run.

By default, `get-rag` uses the `0.4.3` RAG image.

You can also provide a full image reference directly:

```sh
make get-rag RAG_CONTENT_IMAGE=quay.io/redhat-ai-dev/rag-content:<tag>
```

## Configuring Safety Guards

In `llama-stack-configs/run.yaml`, Llama Guard is enabled by default.

> [!IMPORTANT]
> To skip safety guards for development, use `llama-stack-configs/run-no-guard.yaml`.

Start an Ollama container and pull Llama Guard:

```sh
podman run -d --name ollama -p 11434:11434 docker.io/ollama/ollama:latest
podman exec ollama ollama pull llama-guard3:8b
```

Set these environment variables as needed:

- `SAFETY_MODEL`: Llama Guard model name. Defaults to `llama-guard3:8b`
- `SAFETY_URL`: Endpoint URL. Defaults to `http://host.docker.internal:11434/v1`
- `SAFETY_API_KEY`: API key, not required for local

## Makefile Commands

| Command | Description |
| ---- | ---- |
| `get-rag` | Pull and unpack RAG content into `./rag-content` (replaces existing contents). Optional: `RAG_CONTENT_IMAGE=<image>`. |
| `local-up` | Start local compose services for `0.4.3`. Default: `WITH_OLLAMA=true` (uses `run.yaml`). Set `WITH_OLLAMA=false` to use `run-no-guard.yaml`. |
| `local-down` | Stop local compose services. |
| `validate-yaml` | Validate YAML formatting/syntax in config directories. |
| `format-yaml` | Format YAML files in config directories. |
| `update-question-validation` | Update question-validation content in `config/providers.d`. Optional: `QUESTION_VALIDATION_TAG=<tag>`. |
| `validate-prompt-templates` | Validate prompt values against upstream templates. |
| `update-prompt-templates` | Update prompt values from upstream templates. |

## Updating/Formatting YAML Files

Use make targets (these are also used by CI for validation):

```sh
make format-yaml
make validate-yaml
```

For the question-validation sync target, you can override the upstream tag:

```sh
make update-question-validation QUESTION_VALIDATION_TAG=0.1.17
```

## Troubleshooting

Enable debug logs:

```sh
LLAMA_STACK_LOGGING=all=DEBUG
```

If you hit a permission error for `vector_db`, such as:

```sh
sqlite3.OperationalError: attempt to write a readonly database
```

fix permissions with:

```sh
chmod -R 777 rag-content/vector_db
```