# Local AI Server (Ollama + Open WebUI + ComfyUI + Speaches + Home Assistant)

A self-hosted, GPU-accelerated AI stack running entirely on a single machine via Docker Compose:

- **Ollama** — local LLM inference
- **Open WebUI** — chat UI, RAG, image generation front-end, voice input/output
- **ComfyUI** — Stable Diffusion image generation
- **Speaches** — OpenAI-compatible STT (Whisper) and TTS (Kokoro) server
- **Home Assistant** — smart home hub with an LLM-powered voice assistant ("Assist")
- **Caddy** — reverse proxy providing HTTPS for secure browser mic access
- **Tailscale** — private remote access without exposing ports to the internet

This README documents the full setup, the exact configuration that got everything working, and the non-obvious bugs encountered along the way — since several of them cost real debugging time.

---

## 1. Hardware / Environment

| Component | Example spec used here |
|---|---|
| GPU | NVIDIA RTX 2060, 6GB VRAM |
| CPU | Intel i7 (10th gen) |
| RAM | 16GB |
| OS | Ubuntu Server 24.04 LTS (bare metal, no VM) |

**Why bare metal instead of a VM:** 16GB of RAM doesn't leave much headroom for hypervisor overhead on top of an LLM + Stable Diffusion + Whisper + Home Assistant running concurrently. Docker containers on bare metal keep resource usage lean.

**Rough RAM footprint per service (idle → active):**

| Service | Idle | Active |
|---|---|---|
| Ollama (7B model loaded) | ~1GB | ~5–6GB |
| Open WebUI | ~0.5GB | ~1GB |
| ComfyUI (Stable Diffusion) | ~1GB | ~3–4GB |
| Whisper (small/distil model) | ~0.5GB | ~1GB |
| TTS (Kokoro/Piper) | ~0.2GB | ~0.3GB |
| Home Assistant | ~0.5GB | ~1GB |

Running everything simultaneously at peak load can approach the full 16GB. In practice this is rarely an issue since image generation and voice transcription don't usually happen at the same instant — but set up a swap file (8–16GB) as a safety net so the system degrades gracefully instead of OOM-killing a container.

---

## 2. Base OS Setup

1. Download [Ubuntu Server 24.04 LTS](https://ubuntu.com/download/server) and flash it to USB (Rufus / balenaEtcher / `dd`).
2. During install: enable OpenSSH for headless management, and set a static IP (or a DHCP reservation on your router) so the server's LAN address never changes.
3. Update after first boot:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

---

## 3. NVIDIA Drivers + CUDA

```bash
sudo apt install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot
```

Verify:

```bash
nvidia-smi
```

You should see your GPU and driver version listed.

Reference: [Ubuntu NVIDIA driver install docs](https://ubuntu.com/server/docs/nvidia-drivers-installation)

---

## 4. Docker + NVIDIA Container Toolkit

**Docker:**

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
# log out and back in for the group change to take effect
```

**NVIDIA Container Toolkit** (lets containers access the GPU):

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Test GPU access from inside a container:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

Reference: [NVIDIA Container Toolkit install guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)

---

## 5. The Software Stack

| Purpose | Tool | Link |
|---|---|---|
| LLM chat/coding | Ollama | https://ollama.com |
| Chat UI / RAG / hub | Open WebUI | https://github.com/open-webui/open-webui |
| Image generation | ComfyUI | https://github.com/comfyanonymous/ComfyUI (this stack uses the `ai-dock/comfyui` Docker image, which bundles Caddy + a Cloudflare quicktunnel + ComfyUI-Manager) |
| Speech-to-text / text-to-speech | Speaches (faster-whisper + Kokoro) | https://github.com/speaches-ai/speaches |
| Smart home / voice assistant hub | Home Assistant | https://www.home-assistant.io |
| Reverse proxy / HTTPS | Caddy | https://caddyhq.com |
| Private remote access | Tailscale | https://tailscale.com |

---

## 6. Docker Compose

Create the project structure:

```bash
mkdir -p ~/ai-stack && cd ~/ai-stack
mkdir -p ollama openwebui comfyui comfyui-storage homeassistant caddy/certs
```

`~/ai-stack/docker-compose.yml`:

```yaml
services:

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ./ollama:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    network_mode: host
    environment:
      - OLLAMA_BASE_URL=http://127.0.0.1:11434
    volumes:
      - ./openwebui:/app/backend/data
    depends_on:
      - ollama

  comfyui:
    image: ghcr.io/ai-dock/comfyui:latest-cuda
    container_name: comfyui
    restart: unless-stopped
    network_mode: host
    environment:
      - WEB_ENABLE_AUTH=false
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
    volumes:
      - ./comfyui:/home/user/comfyui
      # IMPORTANT — see "ComfyUI checkpoint path gotcha" below.
      # This image's real checkpoint storage lives at /opt/storage, not
      # /home/user/comfyui, and won't persist or be visible unless mounted.
      - ./comfyui-storage:/opt/storage
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  speaches:
    image: ghcr.io/speaches-ai/speaches:latest-cuda
    container_name: speaches
    restart: unless-stopped
    ports:
      - "8000:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    restart: unless-stopped
    privileged: true
    network_mode: host
    volumes:
      - ./homeassistant:/config
      - /etc/localtime:/etc/localtime:ro

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
      - ./caddy/certs:/certs
```

> **Networking note:** `open-webui`, `comfyui`, and `homeassistant` use `network_mode: host` here, which means their published ports (8080, 8188, 8123) bind directly to the host — Compose `ports:` mappings on those services are ignored and can be omitted. `ollama` and `speaches` use the normal Docker bridge network with published ports; services on `network_mode: host` reach them at `127.0.0.1:<port>` since the published port is exposed on the host itself.

Bring it up:

```bash
docker compose up -d
docker compose ps
```

---

## 7. Configuring Each Service

### Ollama + Open WebUI

```bash
docker exec -it ollama ollama pull qwen2.5:7b
docker exec -it ollama ollama pull llama3.1:8b
docker exec -it ollama ollama run hf.co/majentik/gemma-4-E4B-RotorQuant-GGUF-IQ4_XS:IQ4_XS
```

> Ollama runs *inside the container* — there's no bare `ollama` binary on the host, so every pull/run command needs `docker exec -it ollama ...` in front of it.

Visit `http://<server-ip>:8080`, create an admin account, and start chatting. Document upload / RAG works out of the box.

### ComfyUI

Visit `http://<server-ip>:8188`. Download a checkpoint and drop it in the checkpoints folder (see the path gotcha below), then load the default workflow and generate.

**To surface ComfyUI generation inside Open WebUI:**

Admin Panel → Settings → Images:
- Image Generation Engine: **ComfyUI**
- Base URL: `http://127.0.0.1:8188`
- Model: your checkpoint filename
- Upload a ComfyUI workflow JSON and map the node IDs for prompt/model/width/height/steps/seed

Then, per model you want image-gen enabled on:

Workspace → Models → *(your model)* → enable the **Image Generation** capability checkbox. This is a separate toggle from the admin-level Images engine config above, and generation silently won't trigger without it.

### Speaches (STT + TTS)

Speaches serves an OpenAI-compatible API for both Whisper (STT) and Kokoro (TTS) at `http://<server-ip>:8000`. Point Open WebUI's Audio settings and Home Assistant's Assist pipeline both at it.

In Open WebUI Admin Panel → Settings → Audio:
- STT/TTS API Base URL: `http://127.0.0.1:8000/v1`
- STT Model: `whisper-1` (Speaches' OpenAI-compatible alias)
- TTS Model: `tts-1` (Speaches' OpenAI-compatible alias for Kokoro)
- TTS Voice: use a real Kokoro voice name (e.g. `af_heart`), or an OpenAI voice name like `alloy`/`onyx` — Speaches will auto-substitute unsupported names with a default Kokoro voice, but see the per-model override gotcha below.

### Home Assistant + Assist

1. Visit `http://<server-ip>:8123` and complete onboarding.
2. Settings → Voice Assistants → Add Assistant.
3. Conversation agent: Settings → Devices & Services → Add Integration → "Ollama" → point at `http://<server-ip>:11434`.
4. STT/TTS: point at the Speaches server (or install HA's own Whisper/Piper add-ons if you'd rather not share the container).
5. Wake word: cheapest hardware path is an [M5Stack Atom Echo](https://www.home-assistant.io/voice_control/) (~$15, flashes with ESPHome), or run [openWakeWord](https://github.com/dscripka/openWakeWord) on the server with a mic plugged in.

### HTTPS + Remote Access (Caddy + Tailscale)

Browsers require a secure context (HTTPS) for microphone access (`getUserMedia()`), so plain `http://<lan-ip>:8080` will fail with "permission denied" for voice input even though everything else works.

1. Install Tailscale on the server and every client device, then verify with `tailscale status`.
2. Enable **MagicDNS** and **HTTPS Certificates** in the Tailscale admin console's DNS settings.
3. Issue a real Let's Encrypt cert for your tailnet hostname:

```bash
sudo tailscale cert <your-machine>.<your-tailnet>.ts.net
```

4. `Caddyfile` example:

```
<your-machine>.<your-tailnet>.ts.net {
    tls /certs/<your-machine>.<your-tailnet>.ts.net.crt /certs/<your-machine>.<your-tailnet>.ts.net.key
    reverse_proxy 127.0.0.1:8080
}
```

5. Restart Caddy after mounting the cert/key into `./caddy/certs`.
6. **iPhone-specific fix:** if MagicDNS names don't resolve over cellular/other networks, enable "Override local DNS" with a public resolver (e.g. Cloudflare) under Tailscale admin → DNS.
7. Set up monthly cert renewal since Tailscale/Let's Encrypt certs expire ~90 days out:

`~/ai-stack/scripts/renew-cert.sh`:
```bash
#!/usr/bin/env bash
sudo tailscale cert <your-machine>.<your-tailnet>.ts.net
sudo chown "$USER":"$USER" ~/ai-stack/caddy/certs/*
docker restart caddy
echo "$(date): cert renewed" >> ~/ai-stack/scripts/renew-cert.log
```

Allow passwordless `sudo tailscale cert` via a sudoers drop-in (`/etc/sudoers.d/tailscale-cert`), then add to crontab:

```
0 3 1 * * /home/<you>/ai-stack/scripts/renew-cert.sh
```

Don't port-forward these services directly to the internet — Tailscale keeps everything private without exposing any ports on your router.

---

## 8. Downloading Models

**Ollama models:**

```bash
docker exec -it ollama ollama pull qwen2.5:7b
docker exec -it ollama ollama pull qwen3:8b
docker exec -it ollama ollama pull phi4-mini
```

**Stable Diffusion checkpoints** (from Hugging Face — use the direct `/resolve/main/...` file URL, *not* the model page URL, or you'll download an HTML page instead of the model):

```bash
wget -O ~/ai-stack/comfyui-storage/stable_diffusion/models/ckpt/dreamshaper_8.safetensors \
  "https://huggingface.co/Lykon/DreamShaper/resolve/main/DreamShaper_8_pruned.safetensors?download=true"

wget -O ~/ai-stack/comfyui-storage/stable_diffusion/models/ckpt/realistic_vision_v6.safetensors \
  "https://huggingface.co/SG161222/Realistic_Vision_V6.0_B1_noVAE/resolve/main/Realistic_Vision_V6.0_NV_B1_fp16.safetensors?download=true"
```

**Speaches STT/TTS models** (pulled at runtime via its API rather than a manual download):

```bash
# English-only, small/fast
curl -X POST "http://127.0.0.1:8000/v1/models/Systran%2Ffaster-distil-whisper-small.en"

# Multilingual, larger, more accurate
curl -X POST "http://127.0.0.1:8000/v1/models/Systran%2Ffaster-whisper-large-v3"

# Kokoro TTS
curl -X POST "http://127.0.0.1:8000/v1/models/speaches-ai%2FKokoro-82M-v1.0-ONNX"
```

---

## 9. Common Commands Cheat Sheet

```bash
# Bring the whole stack up / down
docker compose up -d
docker compose down

# Status
docker compose ps
docker ps

# Follow logs for one service
docker logs -f open-webui
docker logs -f comfyui
docker logs -f speaches

# Grep recent logs for a specific issue
docker logs open-webui --since 2m 2>&1 | grep -i -C 10 -E "tts|speech|422|audio"

# Shell into a container
docker exec -it ollama bash
docker exec -it comfyui bash

# Recreate a single service after editing docker-compose.yml
docker compose up -d --force-recreate comfyui

# Pull latest images and recreate
docker compose pull open-webui
docker compose up -d open-webui

# Confirm GPU is visible inside a container
docker exec -it ollama nvidia-smi

# Check what a container is actually listening on (useful with network_mode: host)
sudo ss -ltnp | grep -E ':8080|:8188|:8000'

# List Docker networks (useful if you're not on network_mode: host everywhere)
docker network ls
docker inspect <container> --format '{{json .NetworkSettings.Networks}}' | jq
```

---

## 10. Lessons Learned / Troubleshooting Notes

These are bugs that cost real time to track down — leaving them here in case they save someone else the same hours.

**TTS returns 422 Unprocessable Entity, but a manual `curl` to the same endpoint works fine.**
Open WebUI has *layered* audio config: per-model override → per-user setting → admin/global default → env var, and a per-model override silently wins even when the admin panel looks correct. Check **Workspace → Models → (your model) → Advanced Params** for a voice name left over from a different TTS backend (e.g. an OpenAI voice name your local TTS engine doesn't support). Point it at a voice your engine actually has, or leave it blank to inherit the global default.

**STT returns 404 even though the model name looks correct in the admin panel.**
Database string fields can pick up invisible leading/trailing whitespace from copy-pasting, which breaks an exact-match lookup against the STT provider's model registry. If a config value "looks right" but fails, clear the field completely and retype it rather than editing around the existing text. (If you have DB access, wrapping the value in brackets in a `SELECT` — e.g. `SELECT '['||value||']' FROM ...` — makes stray whitespace visible.)

**Browser says "permission denied" for the microphone, only over LAN.**
This is virtually always a secure-context issue, not a permissions bug — `getUserMedia()` requires HTTPS (or `localhost`). Put a reverse proxy with a valid cert in front of the UI (see the Caddy + Tailscale section above) rather than debugging browser permissions directly.

**Open WebUI's Image Generation toggle is on, engine is configured, ComfyUI is reachable — but zero requests ever hit ComfyUI, and the model says "no image generation tool was provided."**
There are two separate toggles that both have to be on, and it's easy to configure only one:
1. Admin Panel → Settings → Images (the engine-level config — base URL, model, workflow mapping)
2. Workspace → Models → *(your specific model)* → **Image Generation capability checkbox**

Both are required. If the model itself reports it wasn't given an image-generation tool (rather than hallucinating a fake result), that's a strong signal it's #2, not a networking or engine config problem.

**Tool calls / image generation silently "not seen" by the model despite the tool clearly being enabled.**
If several tools (memory, notes, calendar, image generation, etc.) are enabled at once, their combined schema can push the effective prompt size past the model's context window — Ollama's default `num_ctx` (2048) is easy to exceed once you're running more than one or two tools. The model receives a truncated prompt and the newest/last tool definitions may simply not make it in. Fix: raise `num_ctx` (e.g. to 8192) in the model's Advanced Params in Workspace → Models.

**A downloaded checkpoint doesn't show up in ComfyUI (or disappears after a container restart), even though `wget` reported success.**
Docker images that repackage ComfyUI (e.g. the `ai-dock` image) sometimes run from a different internal path than you'd expect, with the "real" checkpoint directory reached via a symlink to somewhere else entirely (in this case `/opt/storage/...` rather than the more intuitive `/home/user/comfyui/...`). If a volume isn't mounted over that *real* path, anything you download there lives only inside the container's writable layer and won't persist or be visible where you expect. Check `docker exec <container> curl http://127.0.0.1:<port>/object_info/CheckpointLoaderSimple` to see what ComfyUI itself thinks is available before assuming it's a frontend caching issue.

**A same-day-fresh Docker image tag behaves unexpectedly.**
If you're pulling `:main`/`:latest` and something that should work doesn't, check the image's release notes for recently added major features (a new "tool approval" flow, a rewritten settings page, etc.) before deep-diving your own config — pinning to the previous stable tag and retesting is a fast way to rule out a regression.

---

## License

MIT — use, adapt, and share freely.
