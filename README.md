# OC Design → LoRA Skill

两个给 AI 编程助手用的技能（skill），把「讨论一个原创角色」一路做到「训练出能稳定复现的角色 LoRA，并且能验收」。

不是通用教程，是从一个真实跑通的角色项目里提炼的**踩坑记录 + 可执行流程**。所有数字都是实测的，不是估计的。

## 里面是什么

| 技能 | 管什么 |
|---|---|
| `oc-design/` | 用工作坊的方式把模糊的角色想法逼成可落地的设定：锚点观感、可变量、不可变量、决策记录、已知失败模式。产出 `oc-<角色名>.md` |
| `anima-oc-lora/` | 拿上面那份设定，走完素材生成 → 打标 → 训练 → 验收 → 分发六阶段 |

`anima-oc-lora` 的 Phase 0 会先读 `oc-<角色名>.md`，所以两个技能是接力关系，不是并列。

## 安装

复制到你的助手技能目录即可（以 DeepSeek Harness / Claude Code 这类支持 `~/.dsh/skills`、`~/.claude/skills` 的为例）：

```bash
git clone https://github.com/XdeaF/oc-lora-skill.git /tmp/oc-lora-skill

# DeepSeek Harness
cp -r /tmp/oc-lora-skill/oc-design       ~/.dsh/skills/
cp -r /tmp/oc-lora-skill/anima-oc-lora   ~/.dsh/skills/

# Claude Code
cp -r /tmp/oc-lora-skill/oc-design       ~/.claude/skills/
cp -r /tmp/oc-lora-skill/anima-oc-lora   ~/.claude/skills/
```

Windows PowerShell：

```powershell
git clone https://github.com/XdeaF/oc-lora-skill.git $env:TEMP\oc-lora-skill
Copy-Item $env:TEMP\oc-lora-skill\oc-design      $env:USERPROFILE\.dsh\skills\ -Recurse
Copy-Item $env:TEMP\oc-lora-skill\anima-oc-lora  $env:USERPROFILE\.dsh\skills\ -Recurse
```

装完助手会自动发现，用自然语言触发（「帮我设计一个 OC」「练一个角色 LoRA」「为什么换装不生效」），不需要记命令。

## 文件

```
oc-design/
  SKILL.md                 8 阶段工作坊 + 10 条不可协商交互规则
  prompt-templates.md      提示词模板
  critique-rubric.md       六维评分表（18 分制）

anima-oc-lora/
  SKILL.md                 六阶段主流程 + 排障阶梯 T0-T4 + 脚本索引
  references/
    validation-protocol.md 六组验证 + 定量验收标准 + ArcFace 身份量化
    pitfalls.md            踩过的坑 + 已实测的备选手段
    prompt-tables.md       角度词 / 姿势池 / 场景池 / 服装 tag 表
```

## 几条最有价值的结论

**1. 数据集里没写的东西会被焊死。**
训练集 22 张里有 20 张是「双手抓着外套前襟」，LoRA 就把这个动作学成了角色特征——之后不写姿势提示词，它必抓衣襟。数据集里不写姿势 tag ≠ 姿势自由，而是姿势随机且会被平均成一个。

**2. 打光词能让材质负面全部失效。**
裤子怎么都像乳胶，往负面里堆了五轮 `leather / patent leather / shiny fabric` 全无效。真正的原因是正面写了 `rim lighting, edge light`——轮廓光在 SD 里的标准表现就是沿边缘拉亮线，长条圆柱形的腿最容易被拉纵向高光。删掉这个词，高光当场消失。

**3. `CFG=1` 时负面提示词在数学上是失效的。**
`x = uncond + 1×(cond − uncond) ≡ cond`。用「正面要红领带 + 负面禁红领带」验证过：turbo + CFG 1 必出领带，关 turbo + CFG 6 两张都没有。所以「压不掉某个元素」时，别继续堆负面词，要么改正面写法，要么关掉 turbo 走 CFG 6（代价：耗时约 3 倍）。

**4. 画幅宽度决定会不会出双人图。**
平涂纯色底 + 站姿 + 1024 宽 → 约 40% 概率画出「正面 + 侧面」的转面表。832 宽降到 0%。非站姿（坐/跪/蹲）也能解除。

**5. 表情不该打 tag。**
训练集表情 tag 覆盖率为 0，但 8 种表情（中性/微笑/大笑露牙/笑到眯眼/难过/落泪/生气/惊讶）全部能出。让 LoRA 只管「她是谁」，表情交给底模——这是分层，不是缺陷。

**6. 「像不像她」可以量化，但别迷信单张分数。**
ArcFace 嵌入 + 余弦相似度。负样本（不同角色）最高 0.8296，正样本最低 0.8419，间隔只有 0.0124，建议阈值 0.836。分数主要被**头部角度和遮挡**影响，不是分辨率（与脸宽相关系数 −0.029），所以只能同角度比。

## 每个技能都在做的事

不是给你一段提示词，而是**拦住你走错方向**：

- `oc-design` 每轮只推进一个阶段、只问一个问题，并且强制挑刺——不会一次给你一大篇「又冷又温柔、眼神有故事感」的空话
- `anima-oc-lora` 的排障阶梯按「改动成本」排序，T0 用 30 秒自证负面是否生效，避免花两小时重训一个其实是提示词问题的 LoRA

## 请求失败时

`anima-oc-lora` 里所有命令都是示意，路径用 `<你的脚本目录>` 这类占位符。脚本本身没有随仓库发布——需要的话可以按技能里的索引自己实现，每一条都写了用途和关键参数。

## 关于本仓库

从作者自己的角色项目（Anima / SDXL / Forge + Next-Trainer，RTX 3070 Ti 8GB）里提炼，角色相关的路径、命名、权重文件都已剔除，只留方法论与实测数据。没有开源许可证，供阅读和参考使用。

如果这些踩坑记录帮你省下了几个小时，那就达到目的了。
