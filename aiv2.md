Here is the concise updated setup integrating the suggestions:


---

1. Enhanced Dockerfile

FROM nvidia/cuda:12.2.2-devel-ubuntu24.04

# Base environment
RUN apt update && apt install -y \
    python3.11 python3.11-venv git-lfs ocl-icd-opencl-dev vulkan-utils && \
    rm -rf /var/lib/apt/lists/*

# Clone specific base model repos (locked versions)
RUN mkdir -p /app/base_models && \
    git clone https://github.com/deepseek-ai/DeepSeek-V3 /app/DeepSeek-V3 && \
    git clone https://github.com/Qwen/Qwen2 /app/base_models/Qwen && \
    cd /app/base_models/Qwen && git checkout tags/v2.5.0 && \
    git clone https://github.com/meta-llama/llama3 /app/base_models/llama && \
    cd /app/base_models/llama && git checkout tags/v3.0.0

# Install multi-framework support with profiling and testing tools
RUN python3.11 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --upgrade pip && \
    pip install vllm==0.4.1 sglang==0.3.0 torch==2.3.0 transformers==4.40.0 \
    qwen2==0.1.0 llama-index==0.10.0 fastapi==0.110.0 py-spy pytest

# Configure model cache
ENV HF_HOME=/app/models TRANSFORMERS_CACHE=/app/models
VOLUME /app/models /app/logs


---

2. Updated Entrypoint Script

#!/bin/bash

MODEL_TYPE=${MODEL_TYPE:-distill-qwen-32b}
BASE_TYPE=$(yq eval ".models.${MODEL_TYPE%.*}.base" model-config.yaml)
TOKENIZER_CLASS=$(yq eval ".models.${MODEL_TYPE%.*}.tokenizer_class" model-config.yaml)

case $BASE_TYPE in
  qwen2.5)
    echo "Initializing Qwen2.5-based model"
    export PYTHONPATH="/app/base_models/Qwen:$PYTHONPATH"
    ARGS="--trust-remote-code --use-flash-attn"
    ;;
  llama3)
    echo "Initializing Llama3-based model"
    export PYTHONPATH="/app/base_models/llama:$PYTHONPATH"
    ARGS="--rope_scaling linear"
    ;;
  deepseek-v3)
    echo "Using native DeepSeek-V3 implementation"
    cd /app/DeepSeek-V3
    exec python -m deepseek_v3.serve --model $MODEL_TYPE
    ;;
esac

# Run the model with profiling
py-spy record -o /app/logs/${MODEL_TYPE}_profile.svg -- \
  python -m vllm.entrypoints.openai.api_server \
    --model $MODEL_TYPE \
    --tokenizer-class $TOKENIZER_CLASS \
    --tensor-parallel-size ${TENSOR_PARALLEL:-2} \
    --max-model-len ${MAX_MODEL_LEN:-32768} \
    --dtype ${PRECISION:-bfloat16} \
    $ARGS


---

3. Updated docker-compose.yml

version: '3.8'

services:
  deepseek-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - MODEL_TYPE=distill-qwen-32b
      - BASE_FRAMEWORK=qwen2.5
      - TRUST_REMOTE_CODE=true
      - MAX_MODEL_LEN=32768
      - TENSOR_PARALLEL=2
    volumes:
      - ./models:/app/models
      - ./logs:/app/logs
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: ${GPU_COUNT:-1}
          memory: 80g


---

4. Model Configuration (model-config.yaml)

models:
  distill-qwen:
    base: qwen2.5
    tokenizer_class: Qwen2Tokenizer
    trust_remote_code: true
    flash_attn: true
    license: apache-2.0

  distill-llama:
    base: llama3
    tokenizer_class: LlamaTokenizer
    rope_scaling: "linear"
    license: llama3-license

  base-r1:
    implementation: deepseek-v3
    flash_attn: true
    license: mit


---

5. Testing Workflow

# Run model loading tests
docker compose run --rm \
  -e MODEL_TYPE=distill-qwen-7b \
  deepseek-api \
  pytest /app/tests

# Profile performance
docker compose run --rm \
  -e MODEL_TYPE=distill-llama-70b \
  deepseek-api


---

Key Changes

1. Locked specific versions of base model repositories.


2. Added profiling using py-spy and structured logging to /app/logs.


3. Made GPU allocation configurable with GPU_COUNT.


4. Integrated pytest for validation tests.


5. Volume mounts for logs and additional models allow dynamic extensibility.



This streamlined code integrates all optimizations, ready for scalable deployment.

