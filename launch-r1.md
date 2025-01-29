Run DeepSeek R1 Locally (Ultra-Compact Guide)

1. Install Ollama

Linux:

curl -fsSL https://ollama.com/install.sh | sh

Mac & Windows: Ollama Website

2. Run Model

ollama run deepseek-r1:1.5b    # CPU (1.1GB)  
ollama run deepseek-r1         # GPU (4.7GB)  
ollama run deepseek-r1:70b     # High-end GPU (24GB)

3. Serve API

ollama serve

curl http://localhost:11434/api/generate -d '{
  "model": "deepseek-r1",
  "stream": false,
  "prompt": "Hello!"
}'

4. Chat UI

docker run -p 3000:8080 --rm --name open-webui \
  --add-host=host.docker.internal:host-gateway \
  -v open-webui:/app/backend/data \
  ghcr.io/open-webui/open-webui:main

Go to http://localhost:3000

