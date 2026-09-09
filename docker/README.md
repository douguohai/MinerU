# MinerU Docker 离线部署说明（CPU + GPU）

> 本文件记录当前 fork 里 Docker 镜像的构建、测试、离线部署方式，以及后续修改要点。
> 核心目标：模型在**构建阶段**下载并打包进镜像，运行时**完全离线**（不联网下载模型）。

## 一、镜像总览

| 镜像 | 后端 | 打包的模型 | 适用场景 |
|---|---|---|---|
| `mineru:cpu` / `:cpu-3.4.5` | `pipeline`（纯 CPU） | pipeline 模型（布局/OCR/表格/公式） | 无 GPU 环境 |
| `mineru:gpu` / `:gpu-3.4.5` | `vllm`（hybrid/vlm） | pipeline + VLM 全部模型 | 4×3090 |

GHCR 地址：`ghcr.io/douguohai/mineru:<tag>`

tag 说明：
- `:cpu` / `:gpu` —— 滚动 latest，每次构建覆盖
- `:cpu-3.4.5` / `:gpu-3.4.5` —— 固化版本，内网部署建议用这个

## 二、文件清单

| 文件 | 作用 |
|---|---|
| `docker/cpu/Dockerfile` | CPU 镜像（`python:3.11-slim` + CPU 版 torch + pipeline 模型） |
| `docker/cpu/compose.yaml` | CPU 版 `mineru-api` / `mineru-gradio` 服务 |
| `docker/gpu/Dockerfile` | GPU 镜像（`vllm/vllm-openai` + 全部模型，`ARG VLLM_IMAGE` 选 CUDA） |
| `docker/gpu/compose.yaml` | GPU 版 `mineru-api` 单服务（默认 1 卡） |
| `docker/test/` | 中文测试 PDF：`cn_digital.pdf`（数字版）、`cn_ocr.pdf`（扫描 OCR 版） |
| `.github/workflows/build-docker-images.yml` | 构建并推送镜像到 GHCR |
| `.github/workflows/test-docker-images.yml` | 从 GHCR 拉镜像做断网冒烟测试 |
| `.dockerignore` | 构建上下文只保留 `docker/` |

## 三、构建

### 方式 A：GitHub Actions（推荐）

1. `Actions` 页 → 选 `build-docker-images` → **Run workflow**。
2. 参数：
   - `vllm_image`：GPU 基础镜像（按宿主驱动选，见下表）
   - `version`：版本号，默认 `3.4.5`
3. 产物：`ghcr.io/douguohai/mineru:cpu`、`:cpu-<version>`、`:gpu`、`:gpu-<version>`。

### 方式 B：本地构建

```bash
# CPU
docker build -t mineru:cpu -f docker/cpu/Dockerfile .

# GPU（默认 CUDA 12.9）
docker build -t mineru:gpu -f docker/gpu/Dockerfile .
# 指定其它 CUDA 版本
docker build --build-arg VLLM_IMAGE=vllm/vllm-openai:v0.11.0 -t mineru:gpu -f docker/gpu/Dockerfile .
```

## 四、GPU 基础镜像 / CUDA / 驱动对照表

| `VLLM_IMAGE`（vllm/vllm-openai） | CUDA | 最低宿主驱动 |
|---|---|---|
| `v0.21.0` | 13.0 | ≥ 580.88 |
| `v0.21.0-cu129`（默认） | 12.9 | ≥ 575.51 |
| `v0.11.0` | 12.8 | ≥ 570.86（低 CUDA 备选） |
| `v0.10.1.1` | 12.8 | ≥ 570.86（MinerU 支持的最低 vLLM） |

> vLLM 0.21.0 本身最低只提供 CUDA 12.9；需要更低 CUDA 必须降级 vLLM。
> 驱动版本用 `nvidia-smi` 右上角查看。

## 五、测试（GitHub Actions）

1. 先跑完 `build-docker-images`。
2. `Actions` 页 → `test-docker-images` → **Run workflow**。
3. 验证点：
   - CPU 镜像：`--network none` 断网解析中文 PDF（数字版 + 扫描 OCR 版）
   - GPU 镜像：检查 GHCR 镜像层（确认模型在构建时打进）
4. 关键输出：`PASS: 中文解析成功，且全程 --network none（无运行时下载）` 即表示模型已打包、运行时不联网。

## 六、离线部署（内网）

在有网机器上（执行一次）：

```bash
docker pull ghcr.io/douguohai/mineru:gpu-3.4.5
docker save ghcr.io/douguohai/mineru:gpu-3.4.5 -o mineru-gpu-3.4.5.tar

docker pull ghcr.io/douguohai/mineru:cpu-3.4.5
docker save ghcr.io/douguohai/mineru:cpu-3.4.5 -o mineru-cpu-3.4.5.tar
```

拷到内网机器后：

```bash
docker load -i mineru-gpu-3.4.5.tar

# GPU 运行（mineru-api 单服务，默认 1 卡；4 卡并行见下方说明）
docker run -d --gpus all --ipc=host --shm-size 32g \
  -p 8000:8000 ghcr.io/douguohai/mineru:gpu-3.4.5 \
  mineru-api --host 0.0.0.0 --port 8000

# CPU 运行（必须带 -b pipeline）
docker run --rm -v $PWD:/data ghcr.io/douguohai/mineru:cpu-3.4.5 \
  mineru -p /data/in.pdf -o /data/out -b pipeline
```

## 七、后续修改要点

| 想改什么 | 改哪里 |
|---|---|
| 模型源 huggingface ↔ modelscope | Dockerfile 里 `mineru-models-download -s xxx` |
| 模型范围（全部 vs 仅 pipeline） | Dockerfile 里 `-m all` / `-m pipeline` |
| GPU 的 CUDA/驱动 | Run workflow 时选 `vllm_image`，或本地 `--build-arg VLLM_IMAGE` |
| 版本号 | Run workflow 时填 `version` |
| GPU 单卡 / 多卡 | `docker/gpu/compose.yaml` 的 `device_ids`；4 卡并行建议 `mineru-router --local-gpus auto`（每卡一个 worker） |
| 显存紧张 | compose 里加 `--gpu-memory-utilization 0.5`（或更低） |
| GitHub runner 磁盘不够 | `build-docker-images.yml` 里 GPU job 的 `runs-on` 换大规格 runner |
| 国内 pip 源 | Dockerfile 里 pip 加 `-i https://mirrors.aliyun.com/pypi/simple` |

## 八、注意事项

1. CPU 版只有 `pipeline` 后端（VLM 必须 GPU），命令行必须带 `-b pipeline`。
2. GPU 实际推理效果需在 3090 机器上验证（GitHub 免费 runner 无 NVIDIA GPU）。
3. GitHub 免费 runner 磁盘 14GB，GPU 镜像构建已加 `free-disk-space` 清理；若仍报 `no space left on device`，换大规格 runner。
