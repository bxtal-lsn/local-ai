# Fully Containerized & Persistent DeepSeek R1 with Podman

## 1. `Create podman-compose.yml`

```
version: '3.8'

services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    command: >
      sh -c "ollama pull deepseek-r1:1.5b &&
             ollama pull deepseek-r1:7b &&
             ollama serve"
    devices:
      - "/dev/dri:/dev/dri"
    security_opt:
      - "label=disable"
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ollama_data:/root/.ollama

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    depends_on:
      - ollama
    ports:
      - "3000:8080"
    volumes:
      - open-webui:/app/backend/data
    extra_hosts:
      - "host.docker.internal:host-gateway"

volumes:
  ollama_data:
  open-webui:
```

---

## 2. Start Everything
```
podman-compose -f podman-compose.yml up -d
```

---

## 3. API Test

```
curl http://localhost:11434/api/generate -d '{
  "model": "deepseek-r1",
  "stream": false,
  "prompt": "Hello!"
}'
```

---

## 4. Web UI

Visit `http://localhost:3000`

---

## 5. Stop Everything
```
podman-compose -f podman-compose.yml down
```