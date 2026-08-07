# 多轨 ASR 接口调用文档

## 服务地址

优先使用 178：

```text
http://192.168.125.178:18190
```

以下三个地址调用同一套多轨任务服务：

```text
http://192.168.125.178:18190  # 主地址
http://192.168.125.167:18190  # 备用入口
http://192.168.125.47:18190   # 备用入口
```

下文使用：

```bash
MULTITRACK_BASE=http://192.168.125.178:18190
```

## 默认模型

多轨任务默认同时调用：

```text
1.7b:df,0.6b:df,avia2:df
```

| 名称 | 模型说明 |
| --- | --- |
| `1.7b` | 我们训练的 1.7B 模型 |
| `0.6b` | 我们训练的 0.6B 模型 |
| `avia2` | 面壁提供的第二代 0.6B 模型 |

### 模型能力与选择建议

| 模型 | 训练数据与舱音适配 | 主要能力 | 在多轨流程中的作用 |
| --- | --- | --- | --- |
| `1.7b` | 使用约 428 小时训练数据，主体为中英文真实、公开、人工复核和伪标签 ATC，并包含长音频、呼号数字、降噪数据、通用中英文语音、TTS 关键字段及负样本。没有使用成规模的连续舱音数据专项微调。 | 三个模型中综合 ATC 识别能力最好，尤其适合复杂中英文通话、呼号、数字、频率和较难无线电音频；推理资源需求最高。 | 主要质量候选。 |
| `0.6b` | 数据包括约 385 小时 ATC 核心数据、15 小时降噪数据、约 89 小时伪标签、30 小时连续长片段、结构化 ATC 数据和约 156 小时通用中英文语音。使用了约 10 小时人工标注的川航、Donica、山航连续舱音片段，另有同来源人工复核片段，但不是直接使用完整数小时舱音做端到端训练。 | 参数量小、推理开销较低，具备直接舱音适配经验，通用语音覆盖较广；复杂 ATC、英文和强噪声下的总体准确率通常不如 `1.7b`。 | 舱音适配和轻量模型互补候选。 |
| `avia2` | 面壁提供的第二代 0.6B 中英双语航空模型。交付资料未披露完整训练数据配比，目前不能确认是否使用过舱音数据专项微调，也没有使用我们的舱音数据继续训练。 | 通用英文识别和无语音拒识相对有优势，模型来源及训练路线与我们的模型不同；在当前复杂 ATC、真实航班和噪声测试上的整体准确率弱于我们的模型。 | 独立第三方候选，用于降低三模型同源错误风险。 |

多轨接口默认同时使用三个模型，通过候选互补和后续融合生成结果。若单独调用模型，优先级建议为：最终质量优先使用 `1.7b`，资源或舱音适配优先使用 `0.6b`，独立对照、通用英文或低误触发补充使用 `avia2`。

`:df` 表示先使用默认 DeepFilter 配置处理音频。一般不需要显式传 `variants`。

## 健康检查

```bash
curl "$MULTITRACK_BASE/health"
```

提交完整任务前可以检查三模型预检：

```bash
curl "$MULTITRACK_BASE/preflight"
```

默认预检同时确认 LLM key 已配置，但不会产生外部 LLM 请求。部署排障时可显式做一次真实 LLM 可用性检查（会产生一次极小请求）：

```bash
curl "$MULTITRACK_BASE/preflight?check_llm=true"
```

## 上传双立体声音频

输入约定：

- `file_a`：左声道为机长，右声道为副驾。
- `file_b`：左声道为 ATC/观察员，右声道为环境声。

```bash
JOB_ID=$(curl -fsS -X POST "$MULTITRACK_BASE/jobs/upload" \
  -F 'file_a=@/path/to/flight-A.mp3' \
  -F 'file_b=@/path/to/flight-B.mp3' \
  -F 'run_name=my_flight' \
  -F 'stages=all' \
  | python -c "import json,sys; print(json.load(sys.stdin)['job_id'])")

echo "$JOB_ID"
```

## 上传四个独立轨道

```bash
JOB_ID=$(curl -fsS -X POST "$MULTITRACK_BASE/jobs/upload_tracks" \
  -F 'captain=@/path/to/captain.wav' \
  -F 'copilot=@/path/to/copilot.wav' \
  -F 'atc=@/path/to/atc.wav' \
  -F 'cam=@/path/to/cam.wav' \
  -F 'scene_id=TV9831' \
  -F 'run_name=tv9831_full' \
  -F 'stages=all' \
  | python -c "import json,sys; print(json.load(sys.stdin)['job_id'])")

echo "$JOB_ID"
```

## 使用四个网络文件地址

四个 URL 必须能够从 178 访问。任务会先完整下载四个文件，再开始处理。

```bash
JOB_ID=$(curl -fsS -X POST "$MULTITRACK_BASE/jobs" \
  -H 'Content-Type: application/json' \
  -d '{
    "scene_id": "TV9831",
    "run_name": "tv9831_urls",
    "track_urls": {
      "captain": "https://example.com/captain.wav",
      "copilot": "https://example.com/copilot.wav",
      "atc": "https://example.com/atc.wav",
      "cam": "https://example.com/cam.wav"
    },
    "stages": "all"
  }' \
  | python -c "import json,sys; print(json.load(sys.stdin)['job_id'])")

echo "$JOB_ID"
```

## 使用服务器已有路径

路径必须在 178 上可访问。双立体声：

```bash
curl -fsS -X POST "$MULTITRACK_BASE/jobs" \
  -H 'Content-Type: application/json' \
  -d '{
    "input_mode": "qacvr_pair",
    "scene_id": "TV9831",
    "run_name": "tv9831_server_paths",
    "tracks": {
      "a": "/path/on/178/flight-A.mp3",
      "b": "/path/on/178/flight-B.mp3"
    },
    "stages": "all"
  }'
```

四个独立轨道：

```bash
curl -fsS -X POST "$MULTITRACK_BASE/jobs" \
  -H 'Content-Type: application/json' \
  -d '{
    "input_mode": "four_tracks",
    "scene_id": "TV9831",
    "run_name": "tv9831_four_paths",
    "tracks": {
      "captain": "/path/on/178/captain.wav",
      "copilot": "/path/on/178/copilot.wav",
      "atc": "/path/on/178/atc.wav",
      "cam": "/path/on/178/cam.wav"
    },
    "stages": "all"
  }'
```

## 查询任务

```bash
curl "$MULTITRACK_BASE/jobs/$JOB_ID"
```

主要状态包括 `queued`、`validating`、`waiting_for_download_slot`、`downloading`、`preparing`、`running`、
`succeeded`、`failed` 和 `cancelled`。

查看最近任务：

```bash
curl "$MULTITRACK_BASE/jobs?limit=20"
```

查看日志：

```bash
curl "$MULTITRACK_BASE/jobs/$JOB_ID/log"
```

## 取消、恢复与重试

这些是新增的可选管理接口；原有提交和查询方式不变。

取消排队、下载或运行中的任务：

```bash
curl -fsS -X POST "$MULTITRACK_BASE/jobs/$JOB_ID/cancel"
```

失败或取消后，从已准备好的输入和现有 unit 标记继续运行；不传请求体时保留原 `units`：

```bash
curl -fsS -X POST "$MULTITRACK_BASE/jobs/$JOB_ID/resume" \
  -H 'Content-Type: application/json' \
  -d '{}'
```

也可以只恢复指定 unit，例如 unit 4：

```bash
curl -fsS -X POST "$MULTITRACK_BASE/jobs/$JOB_ID/resume" \
  -H 'Content-Type: application/json' \
  -d '{"units":"4"}'
```

完整重试会创建新 job，不覆盖原结果：

```bash
curl -fsS -X POST "$MULTITRACK_BASE/jobs/$JOB_ID/retry" \
  -H 'Content-Type: application/json' \
  -d '{}'
```

URL 临时凭证不会写入任务记录。只有下载和规范化已经完成、`input_dir` 仍可用时，`resume`/`retry` 才能免 URL 恢复；否则需按原方式重新提交临时 URL。

## 容量与归档

- URL 下载和 multipart 上传都按单轨、四轨总量及磁盘保留空间做保护；超限分别返回 `413` 或 `507`。
- URL 下载任务按全局槽位排队，避免多个四轨任务同时占满 178 磁盘；排队时已返回 job ID。
- URL 原始下载文件在规范化成功后自动删除，正式输入与结果不受影响。
- 新任务成功后异步镜像到 47 的 `/data/xijunting/atc_asr/multitrack_archive/<job_id>`；178 上仍保留在线展示所需文件。
- `GET /jobs/<job_id>` 的 `archive` 字段记录归档状态。成功任务也可调用 `POST /jobs/<job_id>/archive` 手动重试归档。
- 178 上 `18190` 对 ASR 模型栈使用 `Wants + After` 依赖。模型栈滚动重启期间预检会短暂返回 `503`，但不会再由 systemd 连带终止多轨 API 和已启动任务。

## 查看结果页面

整段任务页面：

```text
http://192.168.125.178:18190/jobs/<job_id>/viewer
```

指定 30 分钟单元页面：

```text
http://192.168.125.178:18190/jobs/<job_id>/units/unit_00_0-1800/viewer
```

## 下载结果

```bash
curl -fSLo index.html \
  "$MULTITRACK_BASE/jobs/$JOB_ID/result/index.html"

curl -fSLo transcript_full.json \
  "$MULTITRACK_BASE/jobs/$JOB_ID/result/transcript_full.json"

curl -fSLo transcript_full.txt \
  "$MULTITRACK_BASE/jobs/$JOB_ID/result/transcript_full.txt"

curl -fSLo unit_transcript.json \
  "$MULTITRACK_BASE/jobs/$JOB_ID/result/unit_00_0-1800/transcript.json"
```

## 常用参数

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `run_name` | 自动生成 | 任务 ID 和输出目录名称；建议每次唯一 |
| `scene_id` | 自动生成 | 场景名称 |
| `stages` | `all` | 完整任务使用 `all` |
| `execution_profile` | `optimized` | 正常调用保持默认值 |
| `units` | 空 | 空表示全部；`0` 表示只处理第一个 30 分钟单元 |
| `variants` | `1.7b:df,0.6b:df,avia2:df` | 三模型候选，一般不修改 |
| `llm_model` | `flash` | LLM 融合模型，可选 `flash` 或 `pro` |
| `llm_api_key` | 不传 | 可选的本次请求临时 key；仅部分提交方式支持 |
| `llm_workers` | `4` | LLM 融合并发数 |
| `llm_max_batches` | `0` | `0` 表示处理全部 LLM batch |

自定义模型参数示例：

```bash
-F 'variants=1.7b:df,0.6b:df,avia2:df'
```

JSON 提交方式：

```json
{
  "variants": "1.7b:df,0.6b:df,avia2:df",
  "execution_profile": "optimized",
  "stages": "all"
}
```
