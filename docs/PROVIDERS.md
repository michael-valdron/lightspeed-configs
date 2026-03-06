# Provider Configuration

Each inference has its own environment variables. You can include all of these in an env file and pass it to your container. Use [default-values.env](./env/default-values.env) as a template, and copy it to `values.env` for local edits.

> [!IMPORTANT]
> These are `.env` values, so avoid wrapping values in quotes unless required by the provider.
>
> `VLLM_API_KEY=token` (recommended)
>
> `VLLM_API_KEY="token"` (can cause parsing issues)

## vLLM

Required:

```env
ENABLE_VLLM=true
VLLM_URL=<your-server-url>/v1
VLLM_API_KEY=<your-api-key>
```

Optional:

```env
VLLM_MAX_TOKENS=<defaults to 4096>
VLLM_TLS_VERIFY=<defaults to true>
```

## Ollama

Required:

```env
ENABLE_OLLAMA=true
OLLAMA_URL=<your-ollama-url>
```

`OLLAMA_URL` guidance:

- If Llama Stack runs directly on your host, use `http://localhost:11434`.
- If Llama Stack runs in a container, use `http://host.containers.internal:11434`.
- On Linux, you may need to open firewall access to the Podman network or run with `--network host`.

## OpenAI

Required:

```env
ENABLE_OPENAI=true
OPENAI_API_KEY=<your-api-key>
```

Get your API key from [platform.openai.com](https://platform.openai.com/settings/organization/api-keys).

## Vertex AI (Gemini)

Required:

```env
ENABLE_VERTEX_AI=true
VERTEX_AI_PROJECT=
VERTEX_AI_LOCATION=
GOOGLE_APPLICATION_CREDENTIALS=
```

Provider details: [Llama Stack Vertex AI docs](https://llamastack.github.io/v0.2.18/providers/inference/remote_vertexai.html).
