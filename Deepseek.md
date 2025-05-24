# RD-Agent Local Setup

## LiteLLM Setup

### Pre-Requisites

```bash
docker pull ghcr.io/berriai/litellm:main-latest
docker pull pytorch/pytorch:2.2.1-cuda12.1-cudnn8-runtime
```

We will configure the docker container to proxy to a local LMStudio with *qwen3-14b* model. Follow LMStudio to setup. In this example, these two models are downloaded and loaded:

1. deepseek-chat
2. text-embedding-nomic-embed-text-v1.5

After models are loaded, you could verify them using curl cmd provided in Developer tab of LM Studio.

### Configure LiteLLM Proxy

Add a yaml file such as litellm_config.yaml

```yaml
model_list:
  - model_name: deepseek-chat
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: os.environ/DEEPSEEK_API_KEY
      input_cost_per_token: 0.00
      output_cost_per_token: 0.00
  - model_name: text-embedding-nomic-embed-text-v1.5
    litellm_params:
      model: lm_studio/text-embedding-nomic-embed-text-v1.5
      api_base: os.environ/LM_STUDIO_API_BASE
      api_key: os.environ/LM_STUDIO_API_KEY
      api_version: os.environ/LM_STUDIO_API_VERSION      
      input_cost_per_token: 0.00
      output_cost_per_token: 0.00        
```

### Start LiteLLM Proxy

```bash
docker run \
    -v $(pwd)/litellm_config.yaml:/app/config.yaml \
    -e DATABASE_URL=postgres://neondbxxxxx \
    -e STORE_MODEL_IN_DB=False \
    -e LITELLM_MASTER_KEY=sk-xxxxxxxxxx \
    -e DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxx \
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

#### Verification

Chat Model:

```bash

```

Embedding Model:

```bash
curl --location 'http://0.0.0.0:4000/embeddings' \
--header 'Authorization: Bearer sk-theNewVirtualKey' \
--header 'Content-Type: application/json' \
--data '{"input": ["Academia.edu uses"], "model": "text-embedding-nomic-embed-text-v1.5"}'
```

#### Add models to litellm model list

Update *rdagent/app/cli.py*

```python
def app():
    litellm.register_model(
    model_cost={ "qwen3-14b":{
        "max_tokens": 32768,
        "max_input_tokens": 32768,
        "max_output_tokens": 0,
        "input_cost_per_token": 0.0,
        "output_cost_per_token": 0.0,
        "supports_function_calling": True,
        "supports_prompt_caching": True,
        "supports_system_messages": True,
        "supports_response_schema": True,
        "custom_llm_provider": "lm_studio",
        "supports_tool_choice": True,
        "litellm_provider": "lm_studio",
        "mode": "moderation"
    }
    })
```

#### Minor Fix in RDAgent

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
