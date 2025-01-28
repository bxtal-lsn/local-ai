Here’s the optimized local setup for an Ubuntu 24.04 development machine, with the same features as the Dockerized setup but adapted for a local environment:


---

1. System Setup

Run these commands to prepare your local Ubuntu 24.04 machine:

# Update and install dependencies
sudo apt update && sudo apt install -y \
    python3.11 \
    python3.11-venv \
    git-lfs \
    ocl-icd-opencl-dev \
    vulkan-utils \
    wget \
    build-essential

# Install NVIDIA CUDA drivers (if not already installed)
sudo apt install -y nvidia-cuda-toolkit


---

2. Clone Repositories

Lock versions for consistency:

# Base directory for models
mkdir -p ~/base_models

# Clone DeepSeek-V3
git clone https://github.com/deepseek-ai/DeepSeek-V3 ~/DeepSeek-V3

# Clone Qwen and lock version
git clone https://github.com/Qwen/Qwen2 ~/base_models/Qwen
cd ~/base_models/Qwen && git checkout tags/v2.5.0

# Clone Llama3 and lock version
git clone https://github.com/meta-llama/llama3 ~/base_models/llama
cd ~/base_models/llama && git checkout tags/v3.0.0


---

3. Create Python Environment

Set up a virtual environment for dependency isolation:

# Create and activate virtual environment
python3.11 -m venv ~/venvs/deepseek
source ~/venvs/deepseek/bin/activate

# Install dependencies
pip install --upgrade pip
pip install vllm==0.4.1 sglang==0.3.0 torch==2.3.0 transformers==4.40.0 \
    qwen2==0.1.0 llama-index==0.10.0 fastapi==0.110.0 py-spy pytest


---

4. Model Configuration

Create model-config.yaml in your working directory:

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

5. Entrypoint Script

Save this as run_model.sh and make it executable (chmod +x run_model.sh):

#!/bin/bash

# Configurable environment variables
MODEL_TYPE=${MODEL_TYPE:-distill-qwen-32b}
BASE_TYPE=$(yq eval ".models.${MODEL_TYPE%.*}.base" model-config.yaml)
TOKENIZER_CLASS=$(yq eval ".models.${MODEL_TYPE%.*}.tokenizer_class" model-config.yaml)

case $BASE_TYPE in
  qwen2.5)
    echo "Initializing Qwen2.5-based model"
    export PYTHONPATH=~/base_models/Qwen:$PYTHONPATH
    ARGS="--trust-remote-code --use-flash-attn"
    ;;
  llama3)
    echo "Initializing Llama3-based model"
    export PYTHONPATH=~/base_models/llama:$PYTHONPATH
    ARGS="--rope_scaling linear"
    ;;
  deepseek-v3)
    echo "Using native DeepSeek-V3 implementation"
    cd ~/DeepSeek-V3
    exec python -m deepseek_v3.serve --model $MODEL_TYPE
    ;;
esac

# Run with profiling
py-spy record -o ~/logs/${MODEL_TYPE}_profile.svg -- \
  python -m vllm.entrypoints.openai.api_server \
    --model $MODEL_TYPE \
    --tokenizer-class $TOKENIZER_CLASS \
    --tensor-parallel-size ${TENSOR_PARALLEL:-2} \
    --max-model-len ${MAX_MODEL_LEN:-32768} \
    --dtype ${PRECISION:-bfloat16} \
    $ARGS


---

6. Logging and Compliance

Create directories for logs and compliance documents:

mkdir -p ~/logs ~/compliance

# Download licenses
wget -O ~/compliance/DeepSeek-R1-LICENSE https://opensource.org/license/mit/
wget -O ~/compliance/Qwen-LICENSE https://www.apache.org/licenses/LICENSE-2.0.txt
wget -O ~/compliance/Llama3-LICENSE https://llama.meta.com/llama3/license/


---

7. Testing and Profiling

Run tests and profile models:

# Activate virtual environment
source ~/venvs/deepseek/bin/activate

# Run model loading test for Qwen
MODEL_TYPE=distill-qwen-7b ./run_model.sh

# Run model loading test for Llama
MODEL_TYPE=distill-llama-70b ./run_model.sh


---

Key Features

1. Virtual Environment: Python dependencies are isolated in ~/venvs/deepseek.


2. Locked Repositories: Specific tags (v2.5.0, v3.0.0) ensure consistent builds.


3. Dynamic Entrypoint: Switch between Qwen, Llama, or DeepSeek by changing MODEL_TYPE.


4. Profiling: py-spy generates performance profiles in ~/logs.


5. Compliance: Licenses are stored locally in ~/compliance.




---

This setup avoids Docker, while retaining modularity, performance profiling, and compliance features for your local development environment. Let me know if you'd like further refinement!

