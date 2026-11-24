# 用 VideoCaptioner 校对剪映字幕

## 取字幕与原声

1. 在剪映确认用户指定的版本及最后保存时间。同名草稿或子草稿较多时，打开目标时间线，核对时长和底部对白轨，再导出 UTF-8 SRT。
2. 需要批量写回时，保留一份原始草稿快照，以及每条字幕的片段 ID、素材 ID、原生起讫时间和原文。原生时间可能比 SRT 的毫秒时间更精确。SRT 用于比对和交付，写回沿用草稿里的时间。
3. 以剪后对白为准。已有字幕时可以直接处理，听不清的地方再用本地语音识别辅助定位。用原片复听时，先把剪后时间映射到对应的原始素材区间。遇到剪点、偏移或同期轨切换时，在附近找一处清晰话音核对对齐，再判断字词。

## 生成校对候选

按需在 Python 3.10–3.12 的独立环境安装已核对的版本。外部模型服务会接收字幕文本和提示词，使用前先确认服务和发送范围。

```powershell
python -m pip install "videocaptioner==1.4.2" "openai>=1.97.1,<2" "httpx>=0.28,<1"
videocaptioner subtitle --help
```

已有本机 OpenAI 兼容模型服务时，直接用它。Windows 上也可以用 llama.cpp 的 `llama-server.exe` 和支持中文的 GGUF 指令模型。以下以 Qwen2.5-3B-Instruct 为例，将路径替换成下载后的实际路径。服务只绑定本机回环地址。

```powershell
.\llama-server.exe -m ".\qwen2.5-3b-instruct-q4_k_m.gguf" `
  --host 127.0.0.1 --port 8080 --alias local-caption
```

在另一窗口运行 `Invoke-RestMethod 'http://127.0.0.1:8080/v1/models'`，确认返回的模型 ID 与下面的 `--model` 一致。[llama.cpp 服务文档](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md) · [Qwen 官方 GGUF 模型](https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF)

准备 UTF-8 的 `terms.txt`，记录已确认的术语和人名。下面使用虚构示例。

```text
访谈谈到海岸观测。已经确认的术语是“潮位计”。
只纠正有依据的语音识别错字，保留原有语言、口语、分句、条目顺序和标点。
无法判断的地方保留原文。人名、数字、否定词和专业术语需要特别核对。
```

```powershell
videocaptioner subtitle "original.srt" --no-split --no-translate `
  --api-base "http://127.0.0.1:8080/v1" --api-key local --model "local-caption" `
  --prompt-file "terms.txt" --format srt -o "candidate.srt"
```

命令中的 `local` 是无鉴权回环服务的占位值。服务需要鉴权时，用已配置的密钥并省略命令中的 `--api-key local`。先试一小段，查看日志和输出差异，再跑整条字幕轨。若长稿超时或局部回退到原文，可以减小 `--batch-size` 和 `--thread-num`，再重跑有问题的区段。应用的数据目录和请求日志可能包含字幕及提示词。

## 复核并写回剪映

1. **按时间做一对一比对。** 即使关闭断句，候选仍可能漏条、重编号，或把相邻对白合到同一条。用起点和终点组成匹配键。遇到漏条、重复时码、合并或内容串行，保留原条，必要时单独回听。比如中间一条短回应被漏掉，后面各条仍各自匹配原时码，避免整段文字错位。
2. **逐条确认改字。** 每条记录原文、拟改文字和时间，重点复听人名、数字、术语、否定词及大段替换。比如听清讲的是「潮位计」，才把「潮味计」改掉。对依据不足的改动保留原文。也可把复听时另外发现的明确错字记入同一份修改清单。
3. **在工作副本中写回原生字幕。** 写回前确认原稿仍对应采集时的时间线，复核目标片段 ID、时码和原文。按清单更新显示文字。若批量工具发现字幕与其他片段共用文字素材，或改字会影响多段字样式，就用剪映逐条处理这些位置。保留原有出入点、顺序、断行和样式。
4. **核对交付。** 比较前后的字幕条数、所有起讫点和改字清单，确认非对白轨及素材链接符合原稿。打开交付草稿抽查几个修正位置，点选字幕确认可编辑。需要时，从这份工作副本导出修订 SRT。交付说明记录实际完成的检查与采用的修正数。

[VideoCaptioner v1.4.2 安装包](https://pypi.org/project/videocaptioner/1.4.2/) · [官方 CLI 文档](https://github.com/WEIFENG2333/VideoCaptioner/blob/v1.4.2/docs/cli.md) · [校对命令源码](https://github.com/WEIFENG2333/VideoCaptioner/blob/v1.4.2/videocaptioner/cli/commands/subtitle.py)
