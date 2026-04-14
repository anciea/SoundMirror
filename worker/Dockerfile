# ── Seed-VC RunPod Serverless Worker ──────────────────────────────
FROM runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04

WORKDIR /app

# ── System dependencies ──────────────────────────────────────────
RUN apt-get update && apt-get install -y --no-install-recommends \
    ffmpeg \
    git \
    && rm -rf /var/lib/apt/lists/*

# ── Clone Seed-VC ────────────────────────────────────────────────
RUN git clone --depth 1 https://github.com/Plachtaa/seed-vc.git /app/seed-vc

# ── Install Python dependencies (torch already in base image) ────
# Removed: boto3, resemblyzer, jiwer, pydub
RUN pip install --no-cache-dir \
    runpod \
    requests \
    scipy \
    librosa \
    "huggingface-hub==0.28.1" \
    munch \
    einops \
    descript-audio-codec \
    hydra-core \
    transformers \
    soundfile \
    pyyaml \
    accelerate \
    demucs \
    pedalboard

# ── Pre-generate matplotlib font cache (saves 23s at runtime) ────
RUN pip install --no-cache-dir matplotlib && python -c "import matplotlib; print('Font cache generated')"

# ── Pre-download model weights (free during build, not at runtime) ──
ENV HF_HUB_DOWNLOAD_TIMEOUT=120
RUN python -c "\
import time; \
from huggingface_hub import snapshot_download; \
for i in range(3): \
    try: \
        snapshot_download('Plachta/Seed-VC', local_dir='/app/seed-vc/checkpoints/Seed-VC'); \
        print('Seed-VC weights downloaded'); break; \
    except Exception as e: \
        print(f'Attempt {i+1} failed: {e}'); time.sleep(10); \
else: print('WARNING: Seed-VC download failed, will download at runtime')"

RUN python -c "\
import time; \
from transformers import WhisperModel, WhisperFeatureExtractor; \
for i in range(3): \
    try: \
        WhisperModel.from_pretrained('openai/whisper-small'); \
        WhisperFeatureExtractor.from_pretrained('openai/whisper-small'); \
        print('Whisper cached'); break; \
    except Exception as e: \
        print(f'Attempt {i+1} failed: {e}'); time.sleep(10); \
else: print('WARNING: Whisper download failed, will download at runtime')"

# ── Pre-download Demucs htdemucs model (~80MB) ──
RUN python -c "\
import torch, time, os; \
cache_dir = os.path.expanduser('~/.cache/torch/hub/checkpoints'); \
os.makedirs(cache_dir, exist_ok=True); \
for i in range(3): \
    try: \
        for f in os.listdir(cache_dir): os.remove(os.path.join(cache_dir, f)); \
        from demucs.pretrained import get_model; \
        get_model('htdemucs'); \
        print('htdemucs model downloaded'); \
        break; \
    except Exception as e: \
        print(f'Attempt {i+1} failed: {e}'); \
        time.sleep(5); \
else: \
    print('WARNING: demucs download failed, will download at runtime')"

# ── Copy handler ─────────────────────────────────────────────────
COPY handler.py /app/handler.py

# ── Entry point ──────────────────────────────────────────────────
CMD ["python", "-u", "/app/handler.py"]
