# 出图 Prompt 模板（OC 设计工作坊 · 阶段 7 使用）

本文件只在阶段 7–8 读取。所有 prompt 必须**从一致性锚点表生成**，禁止临时编。

## 0. 先做锚点块

顺序固定，任何平台都不改这个顺序，方便逐字复用与出错时二分排查：

1. 主体与年龄：`1girl` / `1boy` + 年龄段
2. 身份外观一句话
3. 发型发色（含分缝方向、长度落点）
4. 眼睛（含左右方向定义、瞳型）
5. 服装结构（高领/收腰/开衩/绑带/靴高等）
6. 标志性道具
7. 色彩限定
8. 风格锚点

**左右方向必须定义清楚写法**，例如 `LEFT eye golden yellow, RIGHT eye sky blue (character's own left/right)`。不写定义，正背面必然画反。

## 1. 平台分支

| 平台 | 写法 | 必须带的参数 |
|---|---|---|
| Midjourney | 自然语言 + 逗号短语混合 | `--ar` 画幅、`--seed` 固定、`--style raw` 减 MJ 默认美化、`--no` 负面词 |
| Stable Diffusion / SDXL | danbooru 标签式，逗号分隔 | 完整 Negative、固定 seed、固定 checkpoint、CFG/steps 写出来 |
| NovelAI | 标签式 + 质量前缀 | 质量前缀 `masterpiece, best quality, very aesthetic, absurdres` + 完整 Negative |

**质量前缀因平台而异，不要跨平台照搬。** 把 SD 的 `masterpiece` 搬到 MJ 会污染风格。

## 2. 三视图（turnaround）

写法：**先生成锚点块 + 视角行，再单独生成负面词块。每个视角各给一个完整可粘贴块**，不要只写"把正面那句替换一下"——用户复制时不该做字符串替换。

| 视角 | 视角行 |
|---|---|
| 正面 | `front view, standing straight, facing viewer, arms relaxed` |
| 侧面 | `side view, profile view, facing right, arms relaxed` |
| 背面 | `back view, seen from behind, facing away, prop clearly visible` |

**先决定背面要不要露脸**：露脸（回头视角）就看得到瞳色，不露脸则正面/侧面承担全部面部识别度。这个决定要写进设定文件。

## 3. 跨图一致性手段（按性价比排序）

1. **锚点块逐字复用** — 最有效，且免费
2. **固定 seed** — 挑出一张满意的图，记下 seed，后续沿用
3. **同模型同采样参数** — 换 checkpoint = 风格必然漂移
4. **图生图** — 用满意的那张以低降噪出新角度，而不是纯文生图
5. **参考图机制** — Midjourney 用 `--cref <图URL> --cw 100`（保留脸+服装）；SD 用 IP-Adapter 或 reference-only
6. **ControlNet** — 固定同一站姿，保证姿态不漂
7. **锁死易错项** — 异色瞳、多部件服装放 Negative（`asymmetric eyes, wrong eye color, mismatched clothing`）
8. **角色 LoRA** — 长期反复画同一角色时唯一真正稳的方案；前期不做

**不推荐"一张图里画三视图"**：每个视角都会变小，瞳色和金纹这类细节最容易糊，且常出现"三个不同的人"。

## 4. 自检门径（过不了就不写 prompt）

- [ ] 锚点表已存在且已确认？
- [ ] 每个视角的 prompt 是完整的，不是替换说明？
- [ ] 左右方向定义写出来了？
- [ ] 负面词里锁了异色瞳与服装？
- [ ] seed 位留了？
- [ ] 平台参数（`--ar` / Negative / 质量前缀）对不对口？
