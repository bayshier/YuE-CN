# 原始示例

`song.json` 包含智能体技能使用的原创 **City Lights** 歌词与风格请求。`melody.abc` 是为这些词写的八小节原创短旋律：每行歌词七个音节，每小节七个音符起音。`score.abc` 加上了和声；`score-jazz.abc` 只改动了和弦符号。这些都是可直接运行的输入，不是已记录的基准或质量结果。

在仓库根目录执行 `pip install .` 之后：

```bash
# 由歌词与风格创作歌曲及其乐谱。
python examples/generate.py --output outputs/create

# 用自由伴奏演绎附带的旋律。
python examples/generate.py --abc-file examples/melody.abc \
  --cot melody --output outputs/melody

# 在相同旋律、歌词、风格与种子下对比两种和声方案。
python examples/generate.py --abc-file examples/score.abc \
  --cot full --output outputs/harmony-original
python examples/generate.py --abc-file examples/score-jazz.abc \
  --cot full --output outputs/harmony-edited
```

脚本会保留全部原生工件，拒绝写入已存在的输出目录，并在生成报告截断时以非零状态退出。随机种子固定在 `song.json` 中；种子有助于对比，但不保证跨设备或跨软件版本得到完全一致的输出。可用 `--revision` 与 `--vae-revision` 固定模型版本。

不依赖 GPU 也能检查符号改动：

```bash
python skills/yue2-music/scripts/abc_tools.py inspect examples/score.abc
python skills/yue2-music/scripts/abc_tools.py compare \
  examples/score.abc examples/score-jazz.abc
```

对比会校验音符音高、时值、节拍与速度未被改动；最终音乐效果的评估仍需试听或音频转写。
