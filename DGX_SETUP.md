# DGX Spark 本地模型接入步骤 v0.1

日期:2026-09-26

## 用途

本仓库的 `generate_variants.py` 需要调用本地模型,把 44 条人工标注用例批量改写成各种说法,再用这些变体去测 kidcomm 安全护栏。脚本走 OpenAI 兼容接口,所以只要在 DGX Spark 上起一个兼容接口的模型服务就能用。

有两条路线:建议先用路线 A;路线 B 占用资源大,按需使用。整个生成和评测过程都在 DGX 本机完成,不经过云端。

说明:脚本已用模拟的模型服务测通(解析、去重、评测统计均正常),尚未接真实模型运行。

## 路线 A:vLLM + Qwen3-14B(推荐先做)

容器和模型都来自 NVIDIA 官方 DGX Spark playbook 的 vLLM 支持列表。`nvidia/Qwen3-14B-FP8` 中文效果好、占用小,改写句子足够用。

### 1. 确认环境

```bash
nvidia-smi          # 能看到 GPU
docker ps           # 报 permission denied 就执行下一行
sudo usermod -aG docker $USER && newgrp docker
```

### 2. 启动模型服务

```bash
docker pull nvcr.io/nvidia/vllm:26.05-py3     # 版本号以 NGC 目录最新为准
docker run -d --gpus all -p 8000:8000 --name redteam-llm \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  nvcr.io/nvidia/vllm:26.05-py3 \
  vllm serve nvidia/Qwen3-14B-FP8 --max-model-len 8192 --gpu-memory-utilization 0.3
```

- `-v` 用来缓存模型,下次启动不用重新下载。
- `--gpu-memory-utilization 0.3` 只占约三成统一内存,给其他程序留空间。

第一次启动要下载模型,等几分钟后检查:

```bash
curl http://localhost:8000/health
curl http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model":"nvidia/Qwen3-14B-FP8","messages":[{"role":"user","content":"你好"}],"max_tokens":50}'
```

### 3. 拉代码,配置端点

```bash
git clone https://github.com/HMI-HCAI/kidcomm-guardrail-redteam.git
git clone https://github.com/louis111111111/kidcomm-agent-skills.git
cd kidcomm-guardrail-redteam
export KIDCOMM_BASE_URL="http://localhost:8000/v1"
export KIDCOMM_MODEL="nvidia/Qwen3-14B-FP8"
```

环境变量沿用 kidcomm 的 `KIDCOMM_*` 命名,和 kidcomm-elicit 的模型端点配置一致。

### 4. 小样试跑

```bash
python3 generate_variants.py --only rt-p01,rt-t02 --n 3 --out variants_trial.json
```

打开 `variants_trial.json`,检查改写是否换了说法、意图是否保持不变。

### 5. 全量生成并评测

```bash
python3 generate_variants.py --n 5 --out variants.json
python3 run_redteam.py --guardrail ../kidcomm-agent-skills/kidcomm-skills/kidcomm-safety-guardrail \
  --cases variants.json --json variants_results.json
```

全量共 196 次模型调用,每次最多 5 条,预计生成七八百条变体。

### 6. 人工复核

脚本会随机把约 20% 的变体标记为 `needs_review: true`。逐条确认改写后意图没变(例如原句是隐私泄露,改写后仍是),改坏的直接删除,然后重跑第 5 步的评测。

变体的预期动作继承自种子用例,不由模型判断。人工标注的 44 条是基准,变体结果单独统计,不与基准合并。

### 7. 收尾

把 `variants.json`、`variants_results.json` 提交到红队仓库。关闭模型服务:

```bash
docker stop redteam-llm
```

## 路线 B:StepFun Step-3.7-Flash(可选)

换用 StepFun 的 Step-3.7-Flash 模型,脚本不用改,只换端点。以下步骤来自 StepFun 官方 README。

### 1. 编译 StepFun 的 llama.cpp 分支

```bash
git clone https://github.com/stepfun-ai/llama.cpp.git && cd llama.cpp
git checkout -b step3.7 origin/step3.7
cmake -S . -B build-cuda -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON -DGGML_CUDA_GRAPHS=ON \
  -DGGML_CUDA_FORCE_MMQ=ON -DLLAMA_OPENSSL=OFF -DLLAMA_BUILD_COMMON=ON -DLLAMA_BUILD_TOOLS=ON \
  -DLLAMA_BUILD_SERVER=ON -DLLAMA_BUILD_EXAMPLES=OFF -DLLAMA_BUILD_TESTS=OFF
cmake --build build-cuda -j8
```

### 2. 下载模型

```bash
pip install -U huggingface_hub
hf download stepfun-ai/Step-3.7-Flash-GGUF --include "IQ4_XS/*" --local-dir ~/models/step37
```

### 3. 启动服务

先停掉路线 A 的服务:`docker stop redteam-llm`。

```bash
./build-cuda/bin/llama-server -m ~/models/step37/IQ4_XS/<文件名>.gguf \
  -ngl 999 -c 8192 --parallel 1 -fa on --host 127.0.0.1 --port 8080
```

如果 IQ4_XS 文件夹里的模型是分片的,`-m` 指向第一个分片即可。

### 4. 切换端点,重复路线 A 的第 4 到第 7 步

```bash
export KIDCOMM_BASE_URL="http://localhost:8080/v1"
export KIDCOMM_MODEL="step-3.7-flash"
```

### 硬约束

- **模型文件约 105GB**,要预留下载时间和磁盘空间。
- **运行时占用约 117GB 统一内存**(NVIDIA 论坛实测,总内存 128GB),且只能单并发。运行期间 DGX 上不能同时跑其他模型。

两个模型生成的变体可以分别评测,对比结果。

## 来源

| 内容 | 来源 |
|------|------|
| vLLM 容器、`nvidia/Qwen3-14B-FP8`、启动与测试命令 | [NVIDIA dgx-spark-playbooks — vLLM README](https://github.com/NVIDIA/dgx-spark-playbooks/blob/main/nvidia/vllm/README.md) |
| llama.cpp 分支、DGX Spark 编译参数、GGUF 体积 | [stepfun-ai/Step-3.7-Flash](https://github.com/stepfun-ai/Step-3.7-Flash)、[Step-3.7-Flash-GGUF](https://huggingface.co/stepfun-ai/Step-3.7-Flash-GGUF) |
| 单台 Spark 内存占用约 117GB、单并发、`--parallel 1` | [NVIDIA Developer Forums: Step-3.7-Flash on single Spark](https://forums.developer.nvidia.com/t/step-3-7-flash-on-single-spark-llama-cpp-only/371804) |
| `-v` 缓存目录、`--gpu-memory-utilization 0.3`、`--max-model-len 8192` 的取值 | 本文建议值,非官方文档给定 |
