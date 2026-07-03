# ComfyUI Krea2 Style Transfer

Krea2 的 ComfyUI 本地免训练风格参考节点。

[English README](README.md)

这个项目给开源版 Krea2 增加本地风格参考能力。它不是官方 Krea Style Reference 模块，也不会调用官方 API，而是在 ComfyUI 本地采样过程中注入参考图的风格信号。

核心目标很明确：

> 迁移参考图的视觉风格，同时保持新的提示词内容，做到无明显内容泄露，并尽量不损失 Krea2 原本的画面美感和质感。

在单图参考场景里，它的使用体验很接近一个“免训练临时风格 LoRA”：不用准备数据集，不用训练 LoRA，不修改模型权重，只用一张参考图，就能临时给 Krea2 加上对应的画风倾向。

## 解决了什么问题

开源版 Krea2 目前没有开放官方 Style Reference 模块。常见替代方案都有问题：

- 普通图生图容易改变主体、构图和人物特征。
- 纯 VLM 反推风格词依赖文字描述质量，不够直接。
- 早期参考注入路线虽然能迁移风格，但容易内容泄露、画面变脏、质感下降，参数也很难用。

这个节点聚焦在更实用的路线：

- 单图参考风格迁移，同时保持 Krea2 原生画质。
- 在测试案例里没有明显参考图内容泄露。
- 能迁移线条、色彩、材质、笔触、渲染语言和整体视觉气质。
- 普通用户直接用 `recommended`，高级用户再用 `custom` 微调。
- 保留双图实验路线，用于探索两张参考图的风格融合。

## 核心技术思路

这个项目最核心的发现是 `low_scale_end` 和 `ref_k_strength` 的关系。

在参考注入路线里，`low_scale_end` 对风格、质量和泄露影响非常大：

- `low_scale_end` 高：风格更容易进来，但参考图内容也容易泄露，画面可能变脏、质感下降。
- `low_scale_end` 低：画质更好，内容泄露明显降低，但参考风格也容易被压没。

所以原本会形成一个很难平衡的矛盾：

> 要风格，就容易泄露和劣化；要质量和无泄露，风格又进不来。

本项目引入了独立控制 reference K path 的 `ref_k_strength`。

它把两个原本绑在一起的问题拆开了：

- `low_scale_end` 用来压住内容泄露，并保持图片质量。
- `ref_k_strength` 用来在低 `low_scale_end` 下重新激活参考图风格信号。

这是这个节点最重要的改进点。它让 Krea2 可以在低泄露、高质量的参数区间里，仍然保住参考图风格。

简单说：

> `low_scale_end` 负责压住不想要的参考内容泄露，`ref_k_strength` 负责把风格拉回来。

## 单图效果

下面这些例子都使用一张参考图和新的提示词。结果遵循新的提示词内容，同时继承参考图的画风。

<p>
  <img src="docs/images/single_image1.png" width="49%" alt="单图风格参考效果 1">
  <img src="docs/images/single_image2.png" width="49%" alt="单图风格参考效果 2">
</p>
<p>
  <img src="docs/images/single_image3.png" width="49%" alt="单图风格参考效果 3">
  <img src="docs/images/single_image4.png" width="49%" alt="单图风格参考效果 4">
</p>

## 节点说明

### `Krea2 Style Reference`

准备一张风格参考图。

输入：

- `vae`
- `target_latent`
- `reference_image`

输出：

- `reference_latent`
- `reference_preview`
- `debug`

这里要接入实际生成时使用的目标 latent。参考图会被适配到目标 latent 尺寸，让参考路径和当前生成尺寸匹配。

### `Krea2 Style Transfer`

单图风格迁移主节点。

输入：

- `model`
- `reference_latent`
- `ref_conditioning`
- `mode`

`recommended` 是当前调好的低泄露推荐档。`custom` 会显示高级参数，包括 `ref_k_strength`、`low_scale_end` 和相关 RF/attention 参数。

### `Krea2 Two Style References`

打包两张准备好的参考 latent。

这个节点有意只支持两张图，原因见下面多图说明。

### `Krea2 Two Style Transfer`

双图风格迁移实验节点。

关键输入：

- `primary_reference`
- `ref_k_1`
- `ref_k_2`

`primary_reference` 用来指定哪张参考图作为主风格锚点。

### `Krea2 Size Preset`

常用 Krea2 尺寸和比例的便捷 latent 节点。

## 单图推荐参数

当前 `recommended` 模式锁定的是这组实测参数：

```text
style_strength: 1.00
ref_k_strength: 1.06
ref_value_mix: 1.00
value_adain_strength: 0.65
rf_mode: flowturbo_pc
gamma: 0.50
beta: 2.50
high_scale_start: 1.04
high_scale_end: 0.00
low_scale_start: 1.00
low_scale_end: 1.10
adain_strength: 0.85
blocks: 7-27
```

推荐采样器：

```text
steps: 8
cfg: 1.0
sampler: euler_ancestral
scheduler: simple
denoise: 1.0
```

## 为什么多图最多只做两张

多图功能目前有意限制为两张参考图。

在这条免训练路线里，参考图不是由一个训练好的官方融合模块统一融合。每张参考图都会把自己的风格信号带入 K/V 路径。两张图时，结果还比较可控：一张可以作为主风格锚点，另一张补充色彩、线条、纹理或氛围。

但三张、四张以上时，多组风格信号往往会互相竞争，而不是稳定融合。实测中容易出现风格变弱、画质下降、随机由某张参考图主导，或者结果变得很难解释。

所以对这条路线来说，两张是更实用、更可控的上限。

## 为什么要区分首图

双图参考不是简单加权平均。

我们实测发现，参考图顺序会影响结果。第一张或者主参考图往往会影响整体风格入口，比如基础色调、轮廓方式、背景倾向、白边/黑边这类强视觉特征。

`primary_reference` 的作用就是让用户主动指定哪张图作为主风格锚点。副参考图仍然会参与，但主参考图更容易决定整体视觉方向。

### 双图效果

下面每一组使用相同参考图和提示词，只是调换主参考图顺序。

<p>
  <img src="docs/images/two_image_order_a1.png" width="49%" alt="双图参考顺序 A 结果 1">
  <img src="docs/images/two_image_order_a2.png" width="49%" alt="双图参考顺序 A 结果 2">
</p>
<p>
  <img src="docs/images/two_image_order_b1.png" width="49%" alt="双图参考顺序 B 结果 1">
  <img src="docs/images/two_image_order_b2.png" width="49%" alt="双图参考顺序 B 结果 2">
</p>

## 官方可能也存在参考顺序/路由现象

这个项目不声称复现了官方 Krea Style Reference 模块。

不过在官方 Krea2 的多参考结果里，也能观察到类似现象：同一组四张结果中，有些图明显偏向一种参考方向，有些图偏向另一种参考方向。这说明官方系统也可能不是简单平均所有参考图，而是存在某种顺序、路由、随机参考占优或分阶段影响。

下面这张截图只是现象观察，不是官方实现证明。

<p>
  <img src="docs/images/official_order_hint.png" width="100%" alt="官方 Krea2 结果里可能存在参考顺序或路由现象">
</p>

我们的双图路线可能摸到了部分相似原理：参考图不是被简单平均，而是作为不同风格信号，在采样过程里根据顺序、阶段或随机性影响结果。

## 这个项目引入了什么

- 独立 ComfyUI 实现，不依赖第三方风格迁移节点。
- 单图低泄露推荐档。
- 独立的 reference K path 控制：`ref_k_strength`。
- 低 `low_scale_end` 保质量、压内容泄露，再用 `ref_k_strength` 拉回风格。
- 双图实验路线，带 `primary_reference` 主参考控制。
- 更简单的使用方式：普通用户只用 `recommended`，高级用户再开 `custom`。

## 局限

- 这不是官方 Krea Style Reference 模块。
- 它不是 LoRA 的完整替代品。它在很多单图风格参考场景里像一个免训练临时风格适配器，但不会训练或保存风格权重。
- 单图参考是目前最稳定、最适合展示的路线。
- 双图参考仍然是实验功能，且对顺序敏感。
- 风格很弱或很通用的参考图，效果可能不明显。
- 当前路线要求参考 latent 和目标 latent 尺寸匹配。

## 安装

把这个文件夹复制或 clone 到 ComfyUI 的 `custom_nodes` 目录：

```text
ComfyUI/custom_nodes/ComfyUI-Krea2-StyleTransfer
```

重启 ComfyUI。

这个项目只提供自定义节点。你仍然需要一套可用的 Krea2 ComfyUI 环境，包括 Krea2 UNet、VAE 和 text encoder。

## 推荐工作流

项目配套两个工作流：

- `workflows/Krea2 Style Transfer Workflow.json`
- `workflows/Krea2 Two Style Transfer Workflow.json`

建议先用单图工作流，它是当前主路线。

## 说明

这个项目是在本地 Krea2 工作流中通过大量实测迭代出来的。它参考了社区公开讨论里的 Krea2 reference injection、RoPE、K/V 风格迁移方向，但节点代码、使用体验、推荐参数，以及基于 `ref_k_strength` 的低泄露路线，都是本项目独立实现和调试出来的。
