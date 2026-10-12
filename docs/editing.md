# 用乐谱与智能体编辑

YuE2 把预期的作曲以 ABC 记谱法的形式开放出来。智能体可以修改具体的音符、和弦符号、速度或段落顺序，再用同一个生成器渲染修改后的作曲。这就是**白盒音乐生成**所指的可编辑接口。

## 一个完整的和声编辑示例

原始测试素材固定了完全相同的旋律、歌词顺序、节拍、速度、风格与随机种子。`score-jazz.abc` 只把 `score.abc` 中的和弦符号改成了七和弦。

```bash
python skills/yue2-music/scripts/abc_tools.py inspect examples/score.abc
python skills/yue2-music/scripts/abc_tools.py inspect examples/score-jazz.abc
python skills/yue2-music/scripts/abc_tools.py compare \
  examples/score.abc examples/score-jazz.abc
python examples/generate.py --abc-file examples/score.abc \
  --cot full --output outputs/harmony-before
python examples/generate.py --abc-file examples/score-jazz.abc \
  --cot full --output outputs/harmony-after
```

符号对比应报告 `match: true`：音符、时值、节拍与速度均未变化。该检查有意允许和声不同。请试听两个输出来判断修改后的和声是否成立、你是否更喜欢它。

## 从生成结果出发

用 `save_artifacts()` 生成歌曲可以同时保留音频与乐谱，也可以只导出规划：

```bash
python skills/yue2-music/scripts/run_yue2.py plan \
  --request examples/song.json --output outputs/plan
cp outputs/plan/score.abc edited.abc
```

把 ABC、歌词、风格和一份有明确边界的修改简报交给智能体，例如：

> 用受爵士影响的和弦为这首歌重新配和声。保留每一个人声与器乐旋律音符、音符时值、小节边界、段落顺序与速度。本次对比保持歌词与风格不变。保存新的 ABC 文件并解释和弦改动。

然后检查编辑后的乐谱并验证要求的不变量：

```bash
python skills/yue2-music/scripts/abc_tools.py inspect edited.abc
python skills/yue2-music/scripts/abc_tools.py compare outputs/plan/score.abc edited.abc
python examples/generate.py --request examples/song.json \
  --abc-file edited.abc --cot full --output outputs/edited
```

辅助脚本只检查有限的原生 ABC 方言。被拒绝的乐谱可能需要记谱转换或修正；不要把不受支持的写法当成音乐上的错误。参见 [ABC 参考](../skills/yue2-music/references/abc-editing.md)。

## 编辑歌词、风格、速度或曲式

复制请求 JSON，只修改目标字段。修改歌词时请让它贴合旋律的断句与音节数。删除、复制或重排段落时，要同时更新乐谱与对应的歌词段落。要在保留音符的前提下改变速度，可以使用对比辅助脚本的 `--allow-tempo-change` 选项；请通过试听确认实际效果。

编辑会重新渲染一段完整录音。乐谱未变并不意味着编辑点之外的演唱、音色或波形完全一致。请保留原始版本，并为每个版本使用全新的输出目录。注意检查截断标志，比较完整输出，而不是只按期望结果挑选样本。

## 使用打包的技能

在支持 `SKILL.md` 包的智能体中加载 [yue2-music](../skills/yue2-music/SKILL.md)。它的辅助脚本覆盖生成、转写、ABC 校验、缓存解码与听测对比。该技能在既有 API 之间引导智能体，无需单独的智能体生成权重。

[查看 9 步 14 版本的编辑演示](https://map-yue2.github.io/#agentic-music-editing)，其中每个版本都附带音频、乐谱、提示、歌词与对话。
