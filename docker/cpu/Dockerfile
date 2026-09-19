# =============================================================================
# MinerU CPU 镜像（纯 CPU，pipeline 后端，无需 GPU）
#
# 构建：在有网络的机器上执行，构建阶段把 pipeline 模型下载并打包进镜像，
#       运行时完全离线（不需要再联网下载模型）。
#   docker build -t mineru:cpu -f docker/cpu/Dockerfile .
#
# 运行 CLI（CPU 必须显式指定 -b pipeline）：
#   docker run --rm -v $PWD:/data mineru:cpu \
#     mineru -p /data/input.pdf -o /data/output -b pipeline
# =============================================================================
FROM python:3.11-slim

# OpenCV 依赖（libgl1/libglib2.0-0）+ 中文字体
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        fonts-noto-core \
        fonts-noto-cjk \
        fontconfig \
        libgl1 \
        libglib2.0-0 && \
    fc-cache -fv && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 先安装 CPU 版 PyTorch，避免后续 pip 解析到 CUDA 版（体积巨大且 CPU 环境不可用）
RUN python3 -m pip install --no-cache-dir \
        torch torchvision \
        --index-url https://download.pytorch.org/whl/cpu

# 安装 mineru core（含 pipeline 依赖 + gradio WebUI，不含 vllm）
RUN python3 -m pip install --no-cache-dir -U 'mineru[core]>=3.4.0,<4.0.0'

# 构建时下载 pipeline 模型（布局/OCR/表格/公式），打包进镜像 → 运行时离线可用
RUN /bin/bash -c "mineru-models-download -s huggingface -m pipeline"

ENV MINERU_MODEL_SOURCE=local \
    MINERU_DEVICE_MODE=cpu

ENTRYPOINT ["/bin/bash", "-c", "export MINERU_MODEL_SOURCE=local MINERU_DEVICE_MODE=cpu && exec \"$@\"", "--"]
