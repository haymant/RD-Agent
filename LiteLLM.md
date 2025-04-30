# RD-Agent Local Setup

## LiteLLM Setup

### Pre-Requisites

```bash
docker pull ghcr.io/berriai/litellm:main-latest
docker pull pytorch/pytorch:2.2.1-cuda12.1-cudnn8-runtime
```

We will configure the docker container to proxy to a local LMStudio with *qwen3-14b* model. Follow LMStudio to setup.

### Configure LiteLLM Proxy

Add a yaml file such as litellm_config.yaml

```yaml
model_list:
  - model_name: qwen3-14b
    litellm_params:
      model: lm_studio/qwen3-14b
      api_base: os.environ/LM_STUDIO_API_BASE
      api_key: "os.environ/LM_STUDIO_API_KEY"
      api_version: "os.environ/LM_STUDIO_API_VERSION"
```

### Start LiteLLM Proxy

```bash
docker run \
    -v $(pwd)/litellm_config.yaml:/app/config.yaml \
    -e DATABASE_URL=postgres://neondbxxxxx \
    -e STORE_MODEL_IN_DB=False \
    -e LITELLM_MASTER_KEY=sk-xxxxxxxxxx \
    -e LM_STUDIO_API_KEY=d6*********** \
    -e LM_STUDIO_API_BASE=http://172.17.0.1:1234/v1 \
    -p 4000:4000 \
    ghcr.io/berriai/litellm:main-latest \
    --config /app/config.yaml --detailed_debug
```

1. Create a new team at http://localhost:4000/ui, 
2. Create a new virtual key
3. Copy the key to .env 

```bash
LM_STUDIO_API_KEY=sk-theNewVirtualKey
```

### Minor Fix in RDAgent

Please refer to same commit to fix some minor of RDAgent. Then create a *.env* file like:

```bash
BACKEND=rdagent.oai.backend.LiteLLMAPIBackend
# It can be modified to any model supported by LiteLLM.
CHAT_MODEL=lm_studio/qwen3-14b
EMBEDDING_MODEL=lm_studio/qwen3-14b
# The backend api_key fully follows the convention of litellm.
OPENAI_API_KEY=noKeyNeeded
LM_STUDIO_API_BASE=http://172.17.0.1:4000
LM_STUDIO_API_KEY=sk-theNewVirtualKey
LM_STUDIO_API_VERSION=noVersionNeeded
```

And then run:

```bash
rdagent fin_factor
```