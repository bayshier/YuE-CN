# 用 YuE2 生成歌曲

在仓库根目录安装（Python 3.12，`python -m pip install .`）。受支持的环境是支持 BF16、显存 24 GB 的 NVIDIA GPU，一次处理一个请求。默认输出为 48 kHz 立体声，带完整的符号规划与试听解码器 [YuE2-Vae](https://huggingface.co/m-a-p/YuE2-Vae)。

## 风格、歌词与规划

把流派、乐器、人声特点、语言和速度写进 `style`；把要唱的词写进 `lyrics`，用 `[Verse]`、`[Chorus]` 等段落标签分隔。从[原始请求](../examples/song.json)开始即可。

```python
import json
from pathlib import Path
from yue2 import YuE2Pipeline

request = json.loads(Path("examples/song.json").read_text(encoding="utf-8"))
with YuE2Pipeline.from_pretrained("m-a-p/YuE2-3B", device="cuda") as pipe:
    song = pipe(**request)
    song.save_artifacts("outputs/song")
    print(song.truncated)
```

`full` 规划旋律与和弦，`melody` 只规划旋律，`off` 不使用符号规划直接生成。传入 `abc` 则直接使用该作曲，需要搭配 `full` 或 `melody`。原生旋律输入包含 `Vocal` 与 `Ins` 两个声部，不含和弦符号。其他 ABC 方言可能需要先转换；[技能中的 ABC 参考](../skills/yue2-music/references/abc-editing.md)描述了受支持的记谱方式。

一次流水线调用产出一个候选。基准测试中的候选挑选是单独的评测步骤。`cfg_scale` 控制文本引导强度；默认值开箱即用，改动采样或引导参数可能影响质量。

等价的命令行调用：

```bash
yue2 generate --request examples/song.json --output outputs/song-cli
```

## 在合成前保存并检查规划

```python
import json
from pathlib import Path
from yue2 import YuE2Pipeline, SymbolicPlan

request = json.loads(Path("examples/song.json").read_text(encoding="utf-8"))
with YuE2Pipeline.from_pretrained("m-a-p/YuE2-3B", device="cuda") as pipe:
    plan = pipe.plan(**request)
    plan.save("outputs/plan")
    restored = SymbolicPlan.load("outputs/plan")
    semantic = pipe.generate_semantic(restored)
    latents = pipe.synthesize(semantic)
    audio = pipe.decode(latents)

import soundfile as sf
sf.write("outputs/plan/audio.flac", audio, 48000)
```

`SymbolicPlan.load` 会恢复精确的 token ID 并校验保存的文件，适用于未改动的规划。要修改作曲，请复制 ABC、编辑副本，再作为新的 `abc` 输入提交；不要就地修改已保存的规划。端到端的 `save_artifacts()` 是保留完整生成记录的最简方式。

## 输出与可复现性

`save_artifacts()` 会保留 `audio.flac`、适用时的 `score.abc`、`plan.json`、语义 token、`latent.npy`、生效配置、耗时、模型标识与完整性记录。请检查 `result.json` 及其 `truncated` 标志——即使模型触及 token 上限，保存的音频也可能仍可播放。每次修改请求都请使用全新目录，并在对比中保留失败样本。

`from_pretrained` 接受本地模型目录，以及 `revision`、`vae_revision`、`cache_dir` 和 `local_files_only=True` 参数。做对比时请固定模型与 VAE 的 revision。不同的 GPU、运行时版本或采样设置都可能改变同一随机种子的生成结果。

## 试听与评测解码器

| 解码器 | 用途 |
|---|---|
| [YuE2-Vae](https://huggingface.co/m-a-p/YuE2-Vae) | 默认生成与试听 |
| [YuE2-Vae-legacy](https://huggingface.co/m-a-p/YuE2-Vae-legacy) | 复现已记录的基准评测协议 |

用 `YuE2Pipeline.from_pretrained("m-a-p/YuE2-3B", vae="m-a-p/YuE2-Vae-legacy")` 显式选择。试听音频与评测音频请放在不同目录。对比解码器时，请解码同一份缓存的声学潜变量，而不是重新生成一首歌：

```bash
python skills/yue2-music/scripts/run_yue2.py decode \
  --source outputs/song --output outputs/song-benchmark \
  --vae m-a-p/YuE2-Vae-legacy
```

该辅助脚本会校验源工件并保留潜变量标识。基准结论只针对声明的评测模型与解码器；本地重新生成一次并不等于复现了已发表的汇总结果。
