# =============================================================================
# MinerU GPU 镜像 —— pipeline-only（无 VLM），CUDA 12.4
#
# 适配「无法升级驱动」的宿主机：NVIDIA 驱动 550.x（nvidia-smi 显示
# CUDA Version: 12.4）。MinerU 的 VLM 后端强制要求 vLLM>=0.10.1.1
# （CUDA 12.8，驱动 >= 570.86），在 CUDA 12.4 上无法运行，因此本镜像
# 只打包 pipeline 后端（版面分析 + OCR + 表格 + 公式），用 CUDA 12.4 的
# PyTorch 在 GPU 上加速，不安装 vLLM。
#
# 构建：
#   docker build -t mineru:gpu-cu124 -f docker/gpu-cu124/Dockerfile .
#
# 运行 CLI（GPU 上跑 pipeline）：
#   docker run --gpus all --rm -v $PWD:/data mineru:gpu-cu124 \
#     mineru -p /data/input.pdf -o /data/output -b pipeline
#
# 运行 API 服务：
#   docker compose -f docker/gpu-cu124/compose.yaml up -d
# =============================================================================

# 基础镜像自带 torch 2.6.0 + CUDA 12.4（cu124）+ cuDNN 9，驱动 >= 550.54 即可。
# 注意：torch 2.6 是最后一个「PyPI 默认 = CUDA 12.4」的版本，2.7 起默认跳到
# CUDA 12.8（驱动需 >= 570），因此必须锁定 torch==2.6.0。
FROM pytorch/pytorch:2.6.0-cuda12.4-cudnn9-devel

# OpenCV 依赖 + 中文字体
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

# 安装 mineru core（pipeline + gradio，不含 vllm），并锁定 torch 2.6.0，
# 避免 pip 把 torch 升到 2.7+（默认 CUDA 12.8，本镜像会跑不起来）。
RUN python -m pip install --no-cache-dir \
        'mineru[core]>=3.4.0,<4.0.0' \
        'torch==2.6.0' \
        'torchvision==0.21.0' \
        --break-system-packages

# 构建时下载 pipeline 模型（布局/OCR/表格/公式），打包进镜像 → 运行时离线可用
RUN /bin/bash -c "mineru-models-download -s huggingface -m pipeline"

ENV MINERU_MODEL_SOURCE=local \
    MINERU_DEVICE_MODE=cuda

ENTRYPOINT ["/bin/bash", "-c", "export MINERU_MODEL_SOURCE=local MINERU_DEVICE_MODE=cuda && exec \"$@\"", "--"]
