# 翻唱一段录音

翻唱从一份可读的作曲开始：先转写原始录音，核对旋律，再让 YuE2 用新风格把它演绎出来。通用 YuE2 权重无需针对翻唱做任何微调即可支持这一工作流。

## 1. 单独部署 SheetSage2

SheetSage2 与 YuE2 的依赖版本不同。请保持两个独立环境，通过 ABC 与音频文件交换数据。转写与生成先后运行，即可共用一块 GPU。

SheetSage2 使用 Python 3.10 或 3.11，需要 FFmpeg 6.1 及其共享库。按其[模型卡](https://huggingface.co/m-a-p/SheetSage2)完成对应平台的准备，然后在 YuE 仓库根目录执行：

```bash
python3.11 -m venv .venv-sheetsage2
.venv-sheetsage2/bin/python -m pip install huggingface-hub==0.36.0
.venv-sheetsage2/bin/huggingface-cli download m-a-p/SheetSage2 \
  --local-dir models/SheetSage2
.venv-sheetsage2/bin/python -m pip install \
  torch==2.8.0 torchaudio==2.8.0 \
  --index-url https://download.pytorch.org/whl/cu126
.venv-sheetsage2/bin/python -m pip install -r models/SheetSage2/requirements.txt
```

加载 SheetSage2 时会按其配置自动加载 MERT-v2-FullSong 编码器，无需单独执行 MERT2 特征提取步骤。

## 2. 转写并核对旋律

```bash
.venv-sheetsage2/bin/python models/SheetSage2/infer.py source.wav \
  --output cover-score --melody-only
```

`cover-score/score.abc` 会保留人声与器乐旋律，同时省略和弦符号。检查命令的退出状态与转写警告，然后对照原始录音审听音符、节拍与段落顺序。转写错误可能被带入翻唱。

同样的操作也可以通过 Transformers 完成：

```python
from transformers import AutoModel

model = AutoModel.from_pretrained(
    "models/SheetSage2", trust_remote_code=True,
).eval().to("cuda")
result = model.transcribe(
    "source.wav", output_dir="cover-score", melody_only=True,
)
if not result.get("abc") or result.get("abc_error"):
    raise RuntimeError("Transcription did not produce a usable melody score")
print(result.get("warnings", []))
```

需要可复现的运行时，请选定经过核对的模型 revision 并保留下载的配置。模型的 Python 实现由 `trust_remote_code=True` 执行。

## 3. 提供歌词与目标风格

参照[原始示例](../examples/song.json)，准备包含 `style`、`lyrics`、`cot` 与 `seed` 的 `cover-request.json`，并将 `cot` 设为 `melody`。歌词用你手头的版本，或先转写再校对；段落标签与歌词顺序要和乐谱对齐。翻译歌词时，请让断句与音节数贴合旋律。

切回 YuE2 环境：

```bash
.venv/bin/python examples/generate.py --request cover-request.json \
  --abc-file cover-score/score.abc --cot melody --output outputs/cover
```

要使用带固定和声的完整乐谱，请用 `cot="full"`。更换风格时推荐仅旋律模式，因为这样伴奏的自由度更大。[基准对比](benchmarks.md#零样本翻唱生成)同时报告了身份保留与目标风格/质量之间的权衡。

## 试试附带的原创旋律

这个例子只需要 YuE2：

```bash
.venv/bin/python examples/generate.py --request examples/song.json \
  --abc-file examples/melody.abc --cot melody --output outputs/original-melody
```

它用原创素材测试按乐谱条件生成的接口，并不是转写结果或基准分数。完整的翻唱工作流可参考[公开翻唱 Demo](https://map-yue2.github.io/#cover)。
