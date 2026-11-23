# 用 VideoCaptioner 校对剪映字幕

## 取字幕与原声

1. 确认最新保存的剪映草稿、时间线和对白轨。导出对应的现有底部字幕为 UTF-8 SRT，留副本。后续要写回原生轨时，保留条目与原字幕片段的对应关系。
2. 用同一剪辑时间线的原声听辨。已有对白字幕时，直接在现有分句和时间的基础上校对。需要补转写时，再为剪后原声选择可用的语音识别方式。

## 生成校对候选

按需在 Python 3.10–3.12 环境安装已核对的版本。用本机的 OpenAI 兼容模型处理字幕。把例子中的端口、模型名和文件路径改成当前环境的值。外部服务会接收字幕文本和提示词，使用前先确认服务和发送范围。

```powershell
python -m pip install "videocaptioner==1.4.2" "openai>=1.97.1,<2" "httpx>=0.28,<1"
videocaptioner subtitle --help
```

此版本会直接导入 `httpx`，上述依赖组合保留了经测试的客户端接口。先用一小段字幕确认日志里没有超时或失败，再处理整条字幕轨。

准备一份 UTF-8 的 `terms.txt`，记录从原声、正式资料或用户确认的术语和人名。虚构示例写法如下。

```text
主题为虚构的海岸观测访谈。已经确认的术语是“潮位计”。
只纠正有依据的语音识别错字，保留原有语言、口语、分句、条目顺序和标点。
听辨不确定的地方保持原文。避免新增人名、数字、事实或省略否定词。
```

本机模型服务已启动时，可以运行下列命令。例中的 `local` 是无鉴权回环服务所用的占位值。服务需要鉴权时，使用已配置的密钥并省略命令中的 `--api-key local`。

```powershell
videocaptioner subtitle "original.srt" --no-split --no-translate `
  --api-base "http://127.0.0.1:8080/v1" --api-key local --model "local-caption" `
  --prompt-file "terms.txt" --format srt -o "candidate.srt"
```

## 复核并写回剪映

1. 比对条数、起讫时间、顺序、断行和标点。关闭断句后仍可能出现漏条、合并文字或重新编号。按原起讫时间或原生片段映射对齐，缺失和无法匹配的条目沿用原文。回写时沿用原字幕时间码、分句和应保留的标点。
2. 逐处核对文字差异。优先听辨人名、数字、术语、否定词或替换过多的条目。比如原字幕写「潮味计」，听清并确认专用术语后，可采用候选「潮位计」。对听不清的地方保留原文并标记待复听。部分模型错误会回退到原文，命令成功退出后仍需查看日志和差异。应用数据目录内的模型请求日志可能包含字幕与提示词。
3. 在新的工作草稿中，将实际采用的文字修改写回原生对白字幕片段。沿用原有时间点、样式和轨道关系，在剪映抽查受改位置及文字可编辑性。需要调整时码时，另按剪后原声确认出入点。交付时可另附修订 SRT。

[VideoCaptioner v1.4.2 安装包](https://pypi.org/project/videocaptioner/1.4.2/) · [官方 CLI 文档](https://github.com/WEIFENG2333/VideoCaptioner/blob/v1.4.2/docs/cli.md) · [校对命令源码](https://github.com/WEIFENG2333/VideoCaptioner/blob/v1.4.2/videocaptioner/cli/commands/subtitle.py)
