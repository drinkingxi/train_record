# 单模型 ASR 接口调用文档

## 服务地址

优先使用 178：

```text
http://192.168.125.178:28080
```

以下三个地址调用同一套服务，接口和任务 ID 完全兼容：

```text
http://192.168.125.178:28080  # 主地址
http://192.168.125.167:28080  # 备用入口
http://192.168.125.47:18080   # 备用入口
```

下文使用：

```bash
ASR_BASE=http://192.168.125.178:28080
```

## 可用模型

调用时只使用下面三个名称：

| `model` | 模型说明 |
| --- | --- |
| `1.7b` | 我们训练的 1.7B 模型 |
| `0.6b` | 我们训练的 0.6B 模型 |
| `avia2` | 面壁提供的第二代 0.6B 模型 |

### 模型能力与选择建议

| 模型 | 训练数据与舱音适配 | 主要能力 | 推荐用途 |
| --- | --- | --- | --- |
| `1.7b` | 使用约 428 小时训练数据，主体为中英文真实、公开、人工复核和伪标签 ATC，并包含长音频、呼号数字、降噪数据、通用中英文语音、TTS 关键字段及负样本。没有使用成规模的连续舱音数据专项微调。 | 三个模型中综合 ATC 识别能力最好，尤其适合复杂中英文通话、呼号、数字、频率和较难无线电音频；推理资源需求最高。 | 默认首选；重视最终识别质量时使用。 |
| `0.6b` | 数据包括约 385 小时 ATC 核心数据、15 小时降噪数据、约 89 小时伪标签、30 小时连续长片段、结构化 ATC 数据和约 156 小时通用中英文语音。使用了约 10 小时人工标注的川航、Donica、山航连续舱音片段，另有同来源人工复核片段，但不是直接使用完整数小时舱音做端到端训练。 | 参数量小、推理开销较低，具备直接舱音适配经验，通用语音覆盖较广；复杂 ATC、英文和强噪声下的总体准确率通常不如 `1.7b`。 | 资源受限、希望提高吞吐量，或需要具有舱音训练经验的候选时使用。 |
| `avia2` | 面壁提供的第二代 0.6B 中英双语航空模型。交付资料未披露完整训练数据配比，目前不能确认是否使用过舱音数据专项微调，也没有使用我们的舱音数据继续训练。 | 通用英文识别和无语音拒识相对有优势，模型来源及训练路线与我们的模型不同；在当前复杂 ATC、真实航班和噪声测试上的整体准确率弱于我们的模型。 | 适合作为独立第三方基线、低误触发补充或多模型融合候选，不建议单独作为复杂 ATC 场景的默认模型。 |

快速选择：

- 不确定选哪个：使用 `1.7b`。
- 更关注速度、资源占用或舱音适配：使用 `0.6b`。
- 更关注独立模型对照、通用英文或低误触发：使用 `avia2`。

## 健康检查

```bash
curl "$ASR_BASE/health"
```

## 提交转录任务

音频和视频使用同一个接口。视频会自动抽取音频。

```bash
JOB_ID=$(curl -fsS -X POST "$ASR_BASE/v1/asr/transcriptions" \
  -F 'file=@/path/to/audio_or_video' \
  -F 'model=1.7b' \
  | python -c "import json,sys; print(json.load(sys.stdin)['job_id'])")

echo "$JOB_ID"
```

切换模型只需修改 `model`：

```bash
-F 'model=0.6b'
-F 'model=avia2'
```

默认自动识别中英文。已知语言时可以强制指定：

```bash
-F 'language=Chinese'
# 或
-F 'language=English'
```

### 通过 URL 提交

当音视频已经位于可访问的 HTTP/HTTPS 地址时，可直接提交 URL，无需调用方先下载再上传：

```bash
JOB_ID=$(curl -fsS -X POST "$ASR_BASE/v1/asr/transcriptions/url" \
  -H 'Content-Type: application/json' \
  -d '{
    "url": "https://example.com/path/audio_or_video.mp4?temporary_token=...",
    "model": "1.7b",
    "use_vad": true,
    "denoise": false
  }' \
  | python -c "import json,sys; print(json.load(sys.stdin)['job_id'])")

echo "$JOB_ID"
```

接口会立即返回 Router 任务 ID，随后在后台下载、预处理并进入模型队列。查询状态、日志和结果仍使用下文相同的 `$JOB_ID`；临时 URL 的 query 参数不会写入任务记录或日志。

URL 必须使用 `http` 或 `https`，指向常见音频/视频格式，单文件大小上限默认 20 GiB。下载超时、空文件、格式不支持或空间不足时，任务会变为 `failed`，具体原因可从状态或日志接口查看。

## 查询任务

```bash
curl "$ASR_BASE/v1/asr/jobs/$JOB_ID"
```

任务成功时 `status` 为 `succeeded`；失败时为 `failed`。

查看日志：

```bash
curl "$ASR_BASE/v1/asr/jobs/$JOB_ID/log"
```

## 获取结果

```bash
curl "$ASR_BASE/v1/asr/jobs/$JOB_ID/result.txt"
curl "$ASR_BASE/v1/asr/jobs/$JOB_ID/result.json"
```

保存到文件：

```bash
curl -fSLo result.txt "$ASR_BASE/v1/asr/jobs/$JOB_ID/result.txt"
curl -fSLo result.json "$ASR_BASE/v1/asr/jobs/$JOB_ID/result.json"
```

`result.json` 的主要内容：

```json
{
  "model": "1.7b-exp32-e5",
  "segments": [
    {
      "start": 0.0,
      "end": 3.2,
      "text": "转录文本",
      "language": "Chinese",
      "confidence": 0.91,
      "repetition_guard_triggered": false
    }
  ]
}
```

接口默认只压缩连续至少 5 次且长度明显异常的机械循环，正常复诵一两次会保留。触发时 `text` 为保护后的线上结果，同时增加 `raw_text` 保存模型原始输出，便于审计；该保护不用于模型直推评估。

## 常用参数

```bash
JOB_ID=$(curl -fsS -X POST "$ASR_BASE/v1/asr/transcriptions" \
  -F 'file=@/path/to/audio_or_video' \
  -F 'model=1.7b' \
  -F 'denoise=true' \
  -F 'denoise_profile=df6' \
  -F 'use_vad=true' \
  -F 'max_seg_sec=8.0' \
  -F 'hotwords=塔台,地面,跑道' \
  -F 'llm_correct=false' \
  | python -c "import json,sys; print(json.load(sys.stdin)['job_id'])")
```

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `file` | 必填 | 调用方机器上的音频或视频文件；`@` 不能省略 |
| `url` | URL 接口必填 | 可临时访问的 HTTP/HTTPS 音频或视频地址；与 `file` 二选一 |
| `model` | `1.7b` | `1.7b`、`0.6b` 或 `avia2` |
| `language` | 不传 | 自动识别；也可传 `Chinese` 或 `English` |
| `denoise` | `false` | 是否启用降噪 |
| `denoise_profile` | `df6` | 启用降噪时可选 `df6` 或 `df9`，通常使用 `df6` |
| `use_vad` | `true` | 是否使用 VAD 分段 |
| `max_seg_sec` | `8.0` | VAD 后单段最长秒数 |
| `batch_size` | `16` | 分段推理批量大小 |
| `hotwords` | 不传 | 逗号分隔的提示词 |
| `max_new_tokens` | `440` | 最大生成 token 数；AVIA2 后端会应用自身上限 |
| `repetition_penalty` | `1.0` | 重复惩罚 |
| `llm_correct` | `false` | 是否启用 LLM 文本修正 |
| `llm_model` | `flash` | LLM 修正模型，可选 `flash` 或 `pro` |
| `llm_api_key` | 不传 | 可选的本次请求临时 key |

## 并发调用

文件上传和 URL 提交均支持多个请求并发，三个模型也可同时被调用。服务端处理链路包括：

- URL 下载：每个 Router 默认最多 8 个下载任务并行，超出的任务自动排队。
- 共享预处理：当前生产配置每个 Router 有 4 个降噪 worker 和 4 个 VAD worker。
- 模型推理：三个 Router 共同分担请求；各模型后端使用动态 batch，当前上限 64、聚合窗口 10 ms。
- 任务队列：每个 Router 配置 64 个 job worker；提交成功后通过 job ID 异步查询，不需要保持长连接。

调用方可直接并发提交，不应等待前一个任务转录完成后再发下一个。并发上限仍受音频长度、是否降噪、模型选择及 GPU 显存影响；大批量调用建议先从 16 至 32 个并发逐步压测。

## Python 示例

```python
import time
import requests

base = "http://192.168.125.178:28080"

with open("audio.wav", "rb") as audio:
    response = requests.post(
        f"{base}/v1/asr/transcriptions",
        files={"file": ("audio.wav", audio, "audio/wav")},
        data={"model": "0.6b", "use_vad": "true"},
        timeout=120,
    )
response.raise_for_status()
job_id = response.json()["job_id"]

while True:
    status = requests.get(f"{base}/v1/asr/jobs/{job_id}", timeout=30).json()
    if status["status"] in {"succeeded", "failed"}:
        break
    time.sleep(1)

if status["status"] != "succeeded":
    raise RuntimeError(status)

result = requests.get(
    f"{base}/v1/asr/jobs/{job_id}/result.json",
    timeout=30,
).json()
print(result)
```
