# Contributing

- [Prerequisites](#prerequisites)
- [Running Locally](#running-locally)
- [Configuring RAG Content](#configuring-rag-content)
- [Configuring Safety Guards](#configuring-safety-guards)
- [Syncing Configs](#syncing-configs)
  - [Syncing Images](#syncing-images)
  - [Syncing Compose Config](#syncing-compose-config)
- [Formatting and Validating YAML](#formatting-and-validating-yaml)
- [Makefile Commands](#makefile-commands)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- [Podman](https://podman.io/docs/installation) v5.4.1+ (recommended) or [Docker](https://docs.docker.com/engine/) v28.1.0+ with Compose support
- [yq](https://github.com/mikefarah/yq) v4.52.4+ for image and config sync/validation

## Running Locally

1. Copy `./env/default-values.env` to `./env/values.env` and fill in any provider-specific values (see [docs/PROVIDERS.md](./PROVIDERS.md)).

2. Pull the RAG content:

```sh
make get-rag
```

3. Start the local API stack:

```sh
make local-up
```

This starts Llama Stack, Lightspeed Core, and Ollama (for Llama Guard). Compose enforces startup order: Ollama healthy -> Llama Stack healthy -> Lightspeed Core starts.

Lightspeed Core uses two mounted config files in local compose:

- `compose/lightspeed-stack.compose.yaml` -> `/app-root/lightspeed-stack.yaml`
- `lightspeed-core-configs/rhdh-profile.py` -> `/app-root/rhdh-profile.py`

To run without safety guards:

```sh
make local-up WITH_SAFETY=false
```

See [Configuring Safety Guards](#configuring-safety-guards) for all available options including remote safety endpoints.

4. Stop services:

```sh
make local-down
```

## Configuring RAG Content

`make get-rag` pulls the embeddings model and vector database from the RAG content image into `./rag-content`. It fully replaces the directory on each run.

To use a different RAG image:

```sh
make get-rag RAG_CONTENT_IMAGE=quay.io/redhat-ai-dev/rag-content:<tag>
```

> [!IMPORTANT]
> The vector_store ID value changes whenever the RAG content is updated in the image. This means that you only need to do the below update once per image.

With Llama Stack `0.4.3` the way Vector Stores are created has changed. This means that the RAG content you download locally by running `make get-rag` contains a generated Vector Store ID. In order for RAG to work properly you need to navigate to `rag-content/vector_db/rhdh_product_docs/<docs number>/llama-stack.yaml` and find the `vector_stores` section, it should look like:

```
vector_stores:
  - embedding_dimension: 768
    embedding_model: sentence-transformers//rag-content/embeddings_model
    provider_id: rhdh-product-docs-1_8
    vector_store_id: vs_3d47e06c-ac95-49b6-9833-d5e6dd7252dd
```

You will need the `vector_store_id` value. After copying that value you will need to update `config.yaml`. The `vector_store_id` you copied will replace the `vector_store_id` in that file.



## Configuring Safety Guards

Safety is controlled by the `ENABLE_SAFETY` environment variable in the Llama Stack config (`config.yaml`). When set, it activates the safety provider, registers the guard model, and creates the shield. When empty, all safety entries are filtered out at startup.

There are three ways to run depending on your safety needs:

### 1. Without safety guards

No Ollama container, no safety provider. Useful for fast local development.

```sh
make local-up WITH_SAFETY=false
```

No additional environment variables are required. `ENABLE_SAFETY` stays empty in `env/default-values.env` and the safety entries in `config.yaml` are skipped.

### 2. With a local safety guard (Ollama)

Starts a local Ollama container that pulls and serves the Llama Guard model automatically.

```sh
make local-up
```

`WITH_SAFETY` defaults to `true`, which includes the `compose.ollama.yaml` overlay. This overlay starts the Ollama service and sets `ENABLE_SAFETY=true` on the Llama Stack container, overriding the empty default from your env file.

| Variable | Default | Description |
| ---- | ---- | ---- |
| `ENABLE_SAFETY` | Set to `true` automatically by the compose overlay | Activates safety providers in `config.yaml` |
| `SAFETY_MODEL` | `llama-guard3:8b` | Llama Guard model name pulled by Ollama |
| `SAFETY_URL` | `http://ollama:11434/v1` | Points to the Ollama compose service |
| `SAFETY_API_KEY` | *(empty)* | Not required for local Ollama |

### 3. With a remote safety guard

Use a Llama Guard model hosted elsewhere (e.g. vLLM on OpenShift) without running a local Ollama container.

```sh
make local-up WITH_SAFETY=false
```

Set the following in `env/values.env`:

| Variable | Required | Description |
| ---- | ---- | ---- |
| `ENABLE_SAFETY` | Yes, set to `true` | Activates safety providers in `config.yaml` |
| `SAFETY_URL` | Yes | URL of the remote safety model endpoint |
| `SAFETY_MODEL` | If different from default | Model name. Defaults to `llama-guard3:8b` |
| `SAFETY_API_KEY` | If the endpoint requires auth | API key for the remote service |

> [!NOTE]
> In this scenario `WITH_SAFETY=false` is used because there is no need for a local Ollama container -- the safety provider in `config.yaml` is activated by `ENABLE_SAFETY=true` in your `values.env` and points to the remote URL.

## Syncing Configs

This repository has sync scripts that keep derived files consistent with their sources. CI validates these on every PR -- if they drift, the PR will fail.

### Syncing Images

[images.yaml](./images.yaml) is the source of truth for sprint images. It is also consumed by an external service for a different environment. The image values in `env/default-values.env` must stay in sync with it.

After updating `images.yaml`:

```sh
make sync-images
```

This reads the `image` field for each service in `images.yaml` and updates the corresponding env vars (`LIGHTSPEED_CORE_IMAGE`, `LLAMA_STACK_IMAGE`, `RAG_CONTENT_IMAGE`) in `env/default-values.env`.

### Syncing Compose Config

`compose/lightspeed-stack.compose.yaml` is derived from `lightspeed-core-configs/lightspeed-stack.yaml`. The only difference is the `llama_stack.url` -- the compose version uses the Docker Compose service name (`http://llama-stack:8321`) instead of localhost.

After updating `lightspeed-core-configs/lightspeed-stack.yaml`:

```sh
make sync-compose-config
```

`make sync-compose-config` only regenerates `compose/lightspeed-stack.compose.yaml`. It does not manage compose volume mounts in `compose/compose.yaml`.

`lightspeed-core-configs/rhdh-profile.py` is maintained directly in this repository (not synced from upstream). Keep `customization.profile_path` in `lightspeed-core-configs/lightspeed-stack.yaml` aligned with the mount path configured in `compose/compose.yaml` (`/app-root/rhdh-profile.py`).

## Formatting and Validating YAML

Format and validate YAML files (also used by CI):

```sh
make format-yaml
make validate-yaml
```

Update the question-validation provider config from upstream:

```sh
make update-question-validation QUESTION_VALIDATION_TAG=0.1.17
```

## Makefile Commands

| Command | Description |
| ---- | ---- |
| `get-rag` | Pull and unpack RAG content into `./rag-content` (replaces existing contents). Optional: `RAG_CONTENT_IMAGE=<image>`. |
| `local-up` | Start local compose services. Default: `WITH_SAFETY=true` (starts Ollama and enables safety). Set `WITH_SAFETY=false` to skip the local Ollama container. |
| `local-down` | Stop local compose services. |
| `sync-images` | Sync image values from `images.yaml` into `env/default-values.env`. Requires `yq`. |
| `validate-images` | Validate that `images.yaml` and `env/default-values.env` are in sync. Requires `yq`. |
| `sync-compose-config` | Sync `compose/lightspeed-stack.compose.yaml` from `lightspeed-core-configs/lightspeed-stack.yaml`. Requires `yq`. |
| `validate-compose-config` | Validate that the compose config is in sync with its source. Requires `yq`. |
| `validate-yaml` | Validate YAML formatting/syntax. |
| `format-yaml` | Format YAML files. |
| `update-question-validation` | Update question-validation content in `config/providers.d`. Optional: `QUESTION_VALIDATION_TAG=<tag>`. |
| `validate-prompt-templates` | Validate prompt values against upstream templates. |
| `update-prompt-templates` | Update prompt values from upstream templates. |

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

If `podman compose` delegates to `docker-compose` and you get a registry auth error like:

```sh
unable to retrieve auth token: invalid username/password: unauthorized
```

it means `docker-compose` cannot find your credentials. Even if you are logged in via `podman login`, `docker-compose` looks for credentials at `~/.docker/config.json`. Write your credentials there with:

```sh
mkdir -p ~/.docker
podman login --authfile ~/.docker/config.json registry.redhat.io
```
