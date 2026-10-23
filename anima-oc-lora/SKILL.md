---
name: anima-oc-lora
description: Use when the user wants to train a character LoRA for an anime/OC character (Anima / SDXL / Forge + Next-Trainer 或 lora-scripts), or when they already trained one and need to diagnose why it clones badly — keywords like 训练角色 LoRA、练 LoRA、数据集素材、三视图、打标、caption、换装不生效、动作被焊死、姿势固定、双人图、像不像她、要不要重训、LoRA 验证。Covers the whole pipeline from anchor image to dataset generation, tagging, training, and validation.
---

# Anima 角色 LoRA 全流程

## Overview

把一个二次元 OC 变成能稳定复现的角色 LoRA。核心不是训练参数，而是**数据集里的「哪些是她、哪些是这张图碰巧的样子」必须能被区分开**。

**与直觉相反的三条**：

1. **表情、视角不要打 tag**。打了就焊死；不打，底模自己会，LoRA 只管「她是谁」。
2. **CFG=1 时负面提示词基本无效**（turbo LoRA 强制 CFG 1）。要压掉任何东西只能改正面写法。
3. **光源问题不能用材质负面词压**。裤子反光的元凶是正面写了 `rim lighting, edge light`，堆 `leather / patent leather` 一辈子修不好。

---

## Phase 0 — 上游交接（先确认这一步）

动手训练前先问：**这个角色有没有已经定稿的设定文件？**

**如果有**（通常是 `oc-<角色名>.md`，由 `oc-design` 技能产出），从里面读这五项，缺一项就停下来问：

| 要读的 | 用来干什么 |
|---|---|
| 角色锚点图路径 + seed | Phase 1 的基准 |
| 已定稿的外貌 tag 串 | 挖出身份前缀 |
| 已定稿的服装段 | 变成 caption 的服装 tag |
| 已确认的设计决策 | 不要擅自推翻（比如已定「蓝内衬是识别色」就别再去压它） |
| 已知失败模式 | 直接继承，不要重复踩 |

**如果没有**，说明角色还在概念阶段，**先去用 `oc-design` 把设计定稿**，不要边设计边训 LoRA——训完发现设计改了，五小时白烧。

**交接时最容易出事的一处**：`oc-design` 产出的提示词里可能混着**散文长句**（`white collared shirt worn open at the chest with visible buttons`），而训练 caption 必须用**短 tag**（`white_shirt, black_blazer`）。散文长句当 caption 会直接导致换装失效。**出图用的 prompt 和训练用的 caption 是两套规范，不要混用。**

---

## Phase 1 — 锚点定稿

**目标**：一张用户认可的、能代表「她是谁」的全身图。

**锚点合格标准（四条硬杠，任一不过换图）**：

- 单人（没有第二个人、第二只鞋）
- 裤脚盖住鞋面、不露脚踝
- 服装材质正确（裤子哑光，不反光）
- 有重心（contrapposto，不是立正站）

**关键陷阱**：锚点图自己那份 prompt 可能带缺陷，**绝不能直接拿来当基准配方**。典型例子——原图用了 `rim lighting, edge light` 做出了好看的轮廓光，但同一组词会让裤子永远反光。要把「能复现她」的词和「这张图碰巧好看」的词分开。

**把定稿写进一个持久化文件**（`oc-<角色名>.md` 或 `README.md`），含：锚点路径、种子、正面提示词、负面提示词、参数、可变量表、决策记录、已知失败模式。换会话靠它续命。

---

## Phase 2 — 素材生成

**目标**：8 个角度 × 每个角度 30–60 张可用图。

**角度清单**：`front` / `three_quarter` / `side_right` / `side_left` / `back` / `back_turn` / `above` / `below`

**参数**（已验证，别改）：

| 项 | 值 |
|---|---|
| 画幅 | **832 × 1216**（宽度决定双人图率，见下） |
| Steps | 18 |
| Sampler | Euler + Simple |
| CFG | 1 |
| Shift | 3 |
| turbo LoRA | `<lora:anima-turbo-lora-v0.2:1>` |

**画幅宽度是双人图的开关**（实测，同 prompt 只改宽度）：

| 宽度 | 双人图率 |
|---|---|
| 1024 | ~40% |
| 896 | ~45% |
| **832** | **0%** |

**双人图的真正触发条件**：**站姿 + 平涂纯色底 = 模型以为你要画「角色转面表」**，于是并排放两个。解除方式（任一即可）：

- 非站姿（坐/跪/蹲/走/倚）
- 实景背景（室内教室/办公室/客厅/卧室/会议室/走廊）
- 收窄画幅到 832

**姿势必须轮换**。如果整个数据集 80% 是同一个动作（比如「双手抓外套前襟」），模型会把这个动作当成角色特征，之后每张都自动摆出来。准备 30+ 姿势池，每个 request 换一个。

**背景用室内实景，不用平涂**。平涂白/灰底既触发转面表，又会让 turbo LoRA 出黑底。街景容易混入背景行人，室内更干净。已实测安全的 6 个：空教室 / 办公室 / 客厅 / 卧室 / 会议室 / 公寓走廊，全部带 `depth of field`。

**四条筛选标准（抽卡淘汰线，任一不过直接丢，不要试图修）**：

1. 单人
2. 裤脚盖住鞋面、不见脚踝
3. 裤子哑光不反光
4. 有重心（contrapposto）

**不要用脚本自动识别双人图**。试过基于列投影的检测器，两个方向都会错（贴太近会漏检、单人图会误判），最终仍要人眼看拼图。

---

## Phase 3 — 打标（caption）

**结构**：`固定身份前缀` + `WD14 自动 tag` + `角度词`

**固定身份前缀**（每个 caption 都写死，共现率必须 100%）：

```
<触发词>, 1girl, solo, short_hair, bangs, black_hair, blue_hair,
multicolored_hair, colored_inner_hair, blue_eyes, pale_skin, red_eyeshadow
```

触发词用不撞名的自造词（`<名字>_character`）。**打标前先查重**：对照 `selected_tags.csv` 确认该词不存在，否则会和底模内置人物/画风撞名，整炉污染。

**三层过滤**（缺一层就翻车）：

1. **剔除 category 3 / 4**（版权 / 角色）。WD14 会把某张误判成 `virtual_youtuber`，不剔除就等于往数据集里塞别人的角色名。
2. **手工黑名单**误判元素：`animal_ears / high_heels / heels / pantylines / earrings / jewelry / cameltoe / cleavage`
3. **体型 tag 一律不写**：`breasts / large_breasts / medium_breasts / flat_chest / abs`。让画面自己表达，写了会锁死比例。

**故意不写的三类**（这是本流程的核心判断，不是遗漏）：

| 不写 | 原因 |
|---|---|
| 表情 tag | 写了会把表情焊死。实测不打表情 tag，8 种表情（含张嘴大笑、流泪、生气、惊讶）全部能出 |
| 视角/镜头 | 同理，交给角度词在出图时控制 |
| 体型 | 见上 |

**必须写的**：服装 tag、姿势 tag、场景 tag。这些是「这张图碰巧的样子」，不写清楚就会被当成角色属性。

---

## Phase 4 — 训练

**关键前置**：**训练前必须关掉 Forge**。8GB 显存下两者会互抢，训练速度从 8.8 s/it 掉到 4.3 s/it，且容易 OOM。

**参数**（Anima 官方推荐，已验证）：

| 项 | 值 |
|---|---|
| network_module | `networks.lora_anima` |
| network_dim / alpha | 16 / 16 |
| train | **unet only** |
| optimizer | AdamW8bit |
| unet_lr | 1e-4 |
| precision | bf16 |
| gradient_checkpointing | 开 |
| **cache_latents** | **必开**（不开会崩在 VAE 编码，报 `fatal : Memory allocation failure` + `CUDA error: unknown error`） |
| repeats × epochs | 10 × 1 |

**步数参考**：387 张 × 10 repeat = 3870 步，约 5 小时（4.56 s/it），最终 loss 0.0703。loss 落在 0.07–0.09 都可接受，低于 0.05 要警惕过拟合。

**命名**：Forge 只认不带版本后缀的文件名。`xxx_v1.safetensors` 在 Forge 里会找不到，复制时去掉 `_v1`。

---

## Phase 5 — 验证

**必须六组都跑，缺一组就会漏掉一类缺陷**。详见 `references/validation-protocol.md`。

| 组 | 测什么 |
|---|---|
| A | 换 seed 看脸稳不稳（4 个 seed） |
| B | 换场景看身份丢不丢（3 个场景） |
| C | 换装看服装能不能脱离（2 套完全不同的衣服） |
| D | 视角 back / above / side |
| E | 表情 8 种 |
| F | 删掉画师 tag 看画风塌不塌 |

**C 组是最重要的一组**：如果只能出训练时那套衣服，说明数据集把服装焊成了角色属性，必须重训。

**判读纪律（我在这上面连错三次）**：

- **不能看缩略图下结论**。张嘴露牙、泪痕这类细节在缩略图上完全看不见，必须看全分辨率。
- **不能从 tag 统计推断能力**。0% 的表情 tag 只说明「数据没给表情标签」，推不出「模型不会表情」——真正的结论要靠对照实验。
- **结论不确定就补测**。宁可多跑 12 张，不要给一个会被推翻的判断。

---

## Phase 6 — 定稿与分发

- 权重复制到 `models/Lora/<名字>.safetensors`（去 `_v1`）
- 写一份 `README.md`：展示图、快速使用、触发词、训练参数、验证结果表、**已知缺陷**
- 已知缺陷要写具体（「不写手部姿势会抓衣襟」而不是「偶有瑕疵」），这是给未来的自己看的
- 附一份 `docs/checklist.md`「出图前检查清单」+「别再试的方向」表
- 87.6 MB 的单文件在 GitHub 100 MB 硬上限内，可以直传，不必上 LFS；但训到 v3 之后仓库会膨胀，那时改用 Releases 挂附件

---

## 排障阶梯（一次只改一个变量）

遇到「改了没反应」时，**不要同时改三样**，否则画面变了也不知道是谁的功劳。按顺序单变量验证：

| 阶梯 | 操作 | 结论指向 |
|---|---|---|
| T0 | 同 seed，CFG 1 跑一张，负面写一个荒唐词（如 `(zombie:2.0)`） | 画面没变化 = 证实负面在 CFG 1 下确实无效，后续所有「加负面」都是白费 |
| T1 | 正面删光所有光源词（`rim lighting, edge light, specular, backlight, studio lighting`），其余不动 | 问题消失 = 光照层问题，就是这个原因；仍在 = 换下一阶梯 |
| T2 | 不挂角色 LoRA，只挂 turbo，跑同一段 prompt | 仍反光 = 底模/turbo 默认光照，不是你数据集的问题 |
| T3 | 角色 LoRA 权重 1.0 → 0.7 → 0.5 | 逐步减弱 = 缺陷被烤进权重，只能重训；完全不减 = 纯提示词问题，改 prompt 就够 |
| T4 | 关掉 turbo，CFG 提到 6–7，重新加回负面词 | 负面开始生效 = 确认这是 CFG 问题；此时可以考虑终图阶段不带 turbo |

**T3 是最省时间的一步**：它能在不重训的前提下判断「该改 prompt 还是该重训」，很多人跳过它直接重训，白烧五个小时。

## Turbo 是不是必须的（已实测）

turbo LoRA 的作用是 18 步出图，代价是**锁死 CFG=1，负面全部失效**。

如果目标是**大批量抽卡 / 素材生成**，turbo 是刚需（否则 30 步×几百张太慢）。

如果目标是**出成品终图、且需要负面生效**，可以关掉 turbo，用 CFG 6–7 + 28–35 步。

**实测结论（同一 seed、同一 prompt，只改 CFG 与是否挂 turbo）**：

| 配置 | 单张耗时 | 负面是否生效 |
|---|---|---|
| turbo + CFG 1 + 18 步 | 16–34 秒 | ❌ 完全无效 |
| 关 turbo + CFG 6 + 28 步 | 40–44 秒 | ✅ 生效 |
| 关 turbo + CFG 7 + 35 步 | 50–52 秒 | ✅ 生效 |

**决定性证据**：正面写 `red necktie`、负面写 `necktie, tie, red tie`。CFG 1 出两张全带显眼红领带；CFG 6 出两张**都没有领带**。这是负面生效最干净的证明。

**代价与副作用**：

- 耗时 2.5–3 倍（抽卡量大时不能忍）
- **构图会变**：同一 seed 在 CFG 1 和 CFG 6 下画出的姿势/取景完全不同，不能拿来做「只改一个变量」的对照
- 画风略变平，蓝内衬这类「风格化小细节」会被削弱，yunsang 味道变淡
- 但人体比例、身份、哑光裤子都保持得住，甚至比 CFG 1 更干净

**怎么选**：抽素材用 turbo；出终图、或者遇到「怎么调都调不掉某个元素」时，关 turbo 走 CFG 6，这是比继续堆正面词更有效的手段。

---

## 交互纪律

- **一次只改 1–2 个变量**。同时改画风+场景+画幅，画面崩了无法归因。
- **给完整可粘贴的提示词**，不要让用户自己「追加」到旧 prompt 上——拼接错误是最高频的事故源。
- **不要擅自改用户的 LoRA 与画师串**。要动先问。
- **不要碰 Forge 的全局设置**（输出目录、文件名 pattern）。后台任务被杀时设置会留在污染状态，用户下次自己出图会全部落错地方。正确做法：用默认设置生成，之后按时间戳 diff 找出新文件并移动。
- **用户说「不用你跑、给我提示词」就停手**，只交 prompt。

---

## 本机已有脚本（先查，不要重写）

本机已经有一套跑通的脚本。**动手前先看这里，不要重新造一遍**。

脚本根目录：`<你的脚本目录>\`

| 阶段 | 脚本 | 用途 | 关键参数 |
|---|---|---|---|
| 环境 | `dsh-start-forge.ps1` | 启动 Forge | 必须以 `sd-webui-forge-classic` 为工作目录，约 60 秒就绪 |
| 素材 | `gen-angles-poses.ps1` | 8 角度 × 姿势轮换出图 | `-Only <角度> -Req 10 -Batch 3 -W 832 -H 1216 -AllPoses -PoseStart 25` |
| 筛选 | `make-contact-sheet.ps1` | 生成拼图 | `-SrcDir <目录> -PerSheet 30 -Cols 6 -ThumbW 200` |
| 归集 | `fix-materials-layout.ps1` | 搬运、重命名、去重 | 排序用 `[long]`，否则大 seed 抛 Int32 溢出 |
| 打标 | `run-tagging.ps1` + `dsh-wd14-tag.py` | WD14 批量打标 | 输出 `_tags\<角度>.tsv`，gen_th 0.35 / char_th 0.85 |
| Caption | `make-captions.ps1` | 生成训练 caption | 固定身份前缀 + 三层过滤 |
| 训练 | `<你的工具目录>\run-anima-train.ps1` | 启动训练 | `-DataSet <数据集名> -OutName <输出名> -Repeats 10 -Epochs 1 -ResPair '832,1216'` |
| 验证 | `verify-character.ps1` | 六组验证 | 复用时改里面的身份段和 LoRA 名 |
| 量化 | `face-similarity.py` | ArcFace 量化「像不像她」 | `--anchor <锚点> --dir <目录> [--neg <对照>] [--out x.csv]` |
| 对照 | `cfg-test.ps1` | turbo vs CFG 6/7 对照实验 | `-PerGroup 2`，产出三组对比图 |
| 分发 | `sync-to-github`（示例脚本名） | 一键同步技能 + 权重到 GitHub | `sync-to-github "提交说明"`，四步：同步技能 → 拷权重 → commit → push |

**注意**：

- 历史迭代版（`gen-dataset-v2/v3/v4/v5`、`dsh-validate-ocA-v2/v3/v4` 等）已归档到 `_archive\2026-10-23\`，**不要用**。
- 所有含中文的 `.ps1` 必须是 **UTF-8 with BOM**，否则 PowerShell 5.1 按 GBK 解码直接报语法错。
- 训练脚本已支持非方分辨率（`-ResPair`），不要改回硬编码正方形。

---

## 参考文件

- `references/validation-protocol.md` — 六组验证的具体脚本结构、参数、判读标准
- `references/pitfalls.md` — 死路清单、环境坑、PowerShell 编码规则、显存管理
- `references/prompt-tables.md` — 角度词、姿势池、场景池、表情写法、负面词基线
