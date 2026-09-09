# =============================================================================
# MinerU GPU 镜像（NVIDIA RTX 3090，Ampere / 计算能力 8.6）
#
# 构建阶段把全部模型（pipeline + VLM）下载并打包进镜像，运行时完全离线。
# 国内网络慢可改用 modelscope 源，把下方 -s huggingface 换成 -s modelscope。
#
# 基础镜像按“宿主 NVIDIA 驱动版本”选择（nvidia-smi 右上角可查）：
#   vllm/vllm-openai:v0.21.0        → CUDA 13.0，驱动 >= 580.88
#   vllm/vllm-openai:v0.21.0-cu129  → CUDA 12.9，驱动 >= 575.51  （当前默认）
#   vllm/vllm-openai:v0.11.0        → CUDA 12.8，驱动 >= 570.86  （低 CUDA 备选）
#   vllm/vllm-openai:v0.10.1.1      → CUDA 12.8，驱动 >= 570.86  （MinerU 支持的最低 vLLM）
#
# 注意：vLLM 0.21.0 本身最低只提供 CUDA 12.9；需要更低 CUDA 时必须降级 vLLM。
# 切换示例（老驱动用低 CUDA）：
#   docker build --build-arg VLLM_IMAGE=vllm/vllm-openai:v0.11.0 \
#     -t mineru:gpu -f docker/gpu/Dockerfile .
# =============================================================================
ARG VLLM_IMAGE=vllm/vllm-openai:v0.21.0-cu129
FROM ${VLLM_IMAGE}

# OpenCV 依赖 + 中文字体
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        fonts-noto-core \
        fonts-noto-cjk \
        fontconfig \
        libgl1 && \
    fc-cache -fv && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 安装 mineru core（vllm 已在基础镜像中）
RUN python3 -m pip install -U 'mineru[core]>=3.4.0' --break-system-packages && \
    python3 -m pip cache purge

# 构建时下载全部模型（pipeline + VLM），打包进镜像 → 运行时离线可用
RUN /bin/bash -c "mineru-models-download -s huggingface -m all"

ENV MINERU_MODEL_SOURCE=local

ENTRYPOINT ["/bin/bash", "-c", "export MINERU_MODEL_SOURCE=local && exec \"$@\"", "--"]
