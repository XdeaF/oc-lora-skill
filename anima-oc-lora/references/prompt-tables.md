# 提示词表

## 角度词

插在 `1girl, solo,` 之后。

| 目录名 | 角度词 |
|---|---|
| front | （空，不写角度词） |
| three_quarter | `three-quarter view, looking at viewer` |
| side_right | `from side, right side view, profile, looking away, full body` |
| side_left | `from side, left side view, profile, looking away, full body` |
| back | `from behind, back view, looking away, no face visible` |
| back_turn | `from behind, back view, looking back over shoulder, profile face` |
| above | `from above, high angle` |
| below | `from below, low angle` |

---

## 姿势池（37 个，逐 request 轮换）

**第一原则：手部姿势必须显式写。** 不写就会自动回去抓衣襟/抓翻领，这是最顽固的缺陷。

### 站姿（最危险，容易触发双人图）

```
hands in pockets
arms at sides, hands down
arms crossed
one hand in pocket
hands behind back
hand on hip
hand raised to hair
hands clasped in front
(contrapposto:1.2), weight on one leg, (relaxed leg slightly forward and crossed:1.15)
(contrapposto:1.2), weight on one leg, (hand in pocket:1.1)
leaning against wall, one foot against wall, hands in pockets
leaning on railing, both hands on railing
walking, arms swinging, looking ahead
running, leaning forward, arms bent
standing on tiptoe, reaching up, arm raised high
backing away, turned away, looking over shoulder
```

### 坐 / 蹲 / 跪（安全，不触发双人图）

```
sitting on chair, legs crossed, hands on knee
sitting on floor, knees up, arms around knees
sitting on steps, leaning back on hands
sitting cross-legged, hands on knees
kneeling, upright, hands on thighs
kneeling, one knee up, forearm on knee
crouching, one knee down, hands on ground
lying on side, propped on elbow
```

### 动作 / 带道具

```
holding smartphone, looking at phone screen
holding takeaway cup, drinking
holding bag on shoulder
eating, holding food
bending down, picking something up
throwing, arm extended
raising both arms overhead
lowering arms, hands at sides
adjusting cuff, one hand at sleeve
hand brushing hair back
doing a stretch, arms up
```

---

## 场景池（6 个室内，已实测无背景行人）

```
empty classroom, desks and chairs, window, indoor
office, desk and monitor, indoor
living room, sofa, indoor
bedroom, bed, indoor
meeting room, long table, indoor
apartment hallway, doors, indoor
```

统一在尾部加 `depth of field`。

**不要用街景**：会混入背景行人剪影，污染「solo」语义（在 CFG 1 下没法用负面压掉）。

---

## 表情写法

出情绪图前，**先把负面里的 `smile, grin, open mouth, teeth, blush` 删掉**。

| 表情 | 写法 |
|---|---|
| 面无表情 | `expressionless, closed mouth` |
| 微笑 | `smile, closed mouth` |
| 大笑 | `big smile, open mouth, teeth` |
| 笑到眯眼 | `laughing, closed eyes, open mouth` |
| 难过 | `sad, downcast eyes, looking down` |
| 哭 | `crying, tears on cheeks` |
| 生气 | `angry, frown, furrowed brow` |
| 惊讶 | `surprised, open mouth, wide eyes` |

---

## 负面词基线

注意：CFG 1 下这些**基本无效**，写上去主要是防止用户切到更高 CFG 时崩。真正要靠正面控制。

```
nsfw, explicit, nude, naked, nipples, sex, 1boy, male,
lowres, bad anatomy, bad hands, extra fingers, deformed hands, missing fingers,
asymmetric eyes, mismatched eye color, elf ears, pointy ears,
worst quality, watermark, text, signature,
multiple views, 2girls, extra person, extra legs,
holding clothes, hands on jacket, gripping jacket, pulling jacket, adjusting clothes
```

服装防漂移（想要指定服装时追加）：

```
necktie, white shirt, collared shirt, button up shirt, dress, qipao, cheongsam,
ankle boots, boots, high heels, pumps, bare ankles, visible socks,
cropped pants, ankle-length trousers
```

---

## 基准正面提示词骨架

```
<lora:anima-turbo-lora-v0.2:1>, <lora:<角色名>:0.7>,
masterpiece, best quality, score_8, very aesthetic, absurdres, highres,
detail eyes, multicolored eyes, @<画师>,
1girl, solo, <角度词>,
<年龄/气质>,
<发型：长度、层次、碎发走向、挑染位置>,
<眼：瞳色、高光、眼线>,
<肤色>,
<体型：少量、不要写太多>,
<服装：整段写全，换装就整段替换>,
<道具/姿势>,
<场景>,
<构图：full body, head to toe, feet visible(降权 0.9), contrapposto>,
depth of field
```

**不要写**：`rim lighting` / `edge light` / `studio lighting`（会让裤子反光）。
**不要写**：表情 tag（交给底模）。
**画幅**：832 × 1216。
