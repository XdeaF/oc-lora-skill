# 死路清单与环境坑

## 一、这些方向别再试（都实测过，全部无效）

| 想解决的问题 | 别再试 | 正解 |
|---|---|---|
| 裤子镜面反光 | 负面堆 `leather / patent leather / shiny fabric / latex` | **删掉正面的 `rim lighting, edge light`**。光源问题不能用材质负面压 |
| 画面回到「抓衣襟」 | 负面堆 `holding clothes / hands on jacket / gripping jacket` | **正面每张都写手部姿势**（`hands in pockets` / `arms at sides`） |
| 蓝内衬太抢眼 | 负面加蓝 / 权重从 1.1 压到 0.9 再到 0.6 / 把内衬词整个删掉 | 全部无效。最后接受它当识别色，或换衣服 |
| 表情不够强 | 提高 LoRA 权重到 0.85 / 0.95 | 权重无关。换更强的表情词，或换 seed |
| 侧脸不够侧 | 只写 `from side, profile` | 补 `right side view` + `full body` |
| 换画风 | 删掉 `@画师` | 画风已内化，删了也不变。只能换别的画师 tag 或重训 |
| 双人图 | 加 `(1girl:1.2), (solo:1.2)` 权重 | 权重无效。收窄画幅到 832，或换成坐/跪/走姿，或用实景背景 |
| 提高 CFG 让负面生效 | CFG 1.4 | **更糟**：双人率从 40% 升到 60%，画风还变平淡。turbo LoRA 的 CFG 必须锁 1 |

---

## 二、CFG=1 的根本含义

turbo / 加速 LoRA 强制 CFG 1，此时**负面提示词几乎不起作用**。

推论：**所有「压掉某个元素」的需求都必须靠改正面写法解决**，不是加负面。

这条推翻了很多常见教程的写法。遇到「负面加了但没效果」，第一步先确认 CFG 是不是 1。

---

## 三、PowerShell 5.1 编码坑（Windows 上最高频的事故源）

这台机器上 `$PSVersionTable.PSVersion` 是 **5.1**，不是 7。后果：

| 操作 | 坑 | 正确写法 |
|---|---|---|
| 写含中文的 .ps1 | 存成 UTF-8 无 BOM，运行时按 GBK 解码 → 报 `表达式或语句中包含意外的标记` / `哈希文本不完整` | `[System.IO.File]::WriteAllText($f,$t,(New-Object System.Text.UTF8Encoding($true)))` |
| 读 UTF-8 无 BOM 的脚本 | `Get-Content` 按 GBK 解码，中文变量（如路径）变乱码，后续文件操作全错 | `[System.IO.File]::ReadAllText($p,[System.Text.Encoding]::UTF8)` |
| 用 edit 类工具改脚本 | 改完 BOM 可能丢失 | 改完重新用 `WriteAllText(..., UTF8Encoding($true))` 补 BOM |
| 排序 seed 数字 | seed 常超过 Int32 上限（如 2919226960），`[int]` 抛「值对于 Int32 太大」 | `[long]` |
| 管道接 python 输出 | `python ... 2>&1 \| Tee-Object` 触发 NativeCommandError，吞掉真实错误 | `*> file` 重定向 |

**判断 BOM 是否存在**：

```powershell
$bytes = [System.IO.File]::ReadAllBytes($f)
$hasBom = ($bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF)
```

---

## 四、不要碰 Forge 的全局设置

曾经为了「让图片直接落到角度文件夹」去改 `POST /sdapi/v1/options` 的 `outdir_txt2img_samples`。踩到的坑：

- **中文路径会静默失败**。PS 5.1 发中文路径时 JSON 编码错乱，Forge 收到 Latin-1 乱码路径，写入不存在的目录，**一张图都不生成而且不报错**。磁盘上连乱码目录都没建出来，所以连"污染"都看不见，只是空跑。
- **任务被杀会让设置留在污染状态**。后台任务被中止时来不及还原，用户下次自己出图全部落进我的测试目录。

**正确做法**：

1. 用默认设置生成
2. 生成后按 `LastWriteTime` 找出新增的 PNG
3. 校验尺寸后 `Move-Item` 到目标目录
4. 目标目录用 ASCII 路径；需要中文文件夹名时用 **junction** 暴露：
   ```powershell
   New-Item -ItemType Junction -Path "中文目录" -Target "G:\ascii\path"
   ```
   移除 junction 只删链接不动数据：`cmd /c rmdir "<junction>"`
5. 用 junction 前务必验证目标数据仍在（junction 和真目录用 `Test-Path` 无法区分）

**另一种路径方案**：`directories_filename_pattern` 设成 `.` 可以把文件平铺进 outdir 根，不建日期子目录。但空字符串会拿 prompt 前几词当文件夹名，不要用。

---

## 五、8GB 显存管理

| 场景 | 症状 | 处理 |
|---|---|---|
| 批量出图累计 50–60 张后 | Forge 后端死、GPU 有残留、无 python 进程 | 每 25–50 张重启 Forge；或缩短批次，加 7–10 秒间隔 |
| Forge 与训练同时跑 | 两者各占 3.2 / 3.5 GB，合计 7776/8192 MiB，训练从 8.8 s/it 掉到 4.3 s/it | **训练前必须关 Forge** |
| 训练启动即崩 | `fatal : Memory allocation failure` + `CUDA error: unknown error` | `--cache_latents` 必开，绕过在线 VAE 编码；`--vae_batch_size 1`；设 `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` |
| Forge 启动就崩每个请求 | 相对路径解析错 | 必须以 `sd-webui-forge-classic` 为工作目录启动 |

---

## 六、数据集质量坑

| 坑 | 后果 | 防法 |
|---|---|---|
| 训练集混入双人图 | 出图时偶发画面边缘多出第二只鞋/第二个人（约 2–4/12） | 逐张看拼图剔除，不要靠脚本自动识别 |
| 姿势单一 | 动作被焊成角色特征，之后每张都自动摆 | 30+ 姿势池，逐 request 轮换 |
| 表情打了 tag | 表情被焊死 | 故意不打 |
| 体型打了 tag | 比例锁死 | 故意不打 |
| 服装 tag 共现率 100% | 换装换不动 | 数据集里就该有 2–3 套衣服，或接受"必须整段重写服装描述" |
| WD14 误判出 category 3/4 | 别人的角色名进了 caption，整炉污染 | 对照 `selected_tags.csv` 全量剔除 |
| 触发词撞名 | 和底模内置人物/画风混在一起 | 打标前先查重 |
| 平涂纯白/灰底 | 触发转面表 + turbo 出黑底 | 用室内实景背景 |

**删图后编号会断**（`back_001 / back_004 / back_005`）是正常的，不要试图重编号——重编号会让之前的对照记录失效。

---

## 七、备选手段（卡死时再试）

### 已经实测过的一条：关掉 turbo 走 CFG 6

**这是最有效的一条，优先试这个。** 详见 SKILL.md「Turbo 是不是必须的」：

- turbo 强制 CFG=1，负面提示词**数学上无效**（`x = uncond + 1×(cond−uncond) ≡ cond`，条件项直接把无条件项消掉）
- 关掉 turbo、CFG 6 + 28 步，负面立刻生效：实测正面写 `red necktie`、负面写 `necktie`，CFG 1 出红领带、CFG 6 干净无领带
- 代价：耗时 2.5–3 倍；构图会随 CFG 变化（不能做单变量对照）；画风略变平，风格化细节（如彩色内衬）被削弱

所以遇到「怎么调正面都压不掉某个元素」，先关 turbo 试 CFG 6，别再继续堆正面词。

---

以下手段**没有在本流程里实测过**，卡死时再试。用之前先标记为实验，不要当结论。

### 反光的其他杠杆

- **换面料名词代替形容词**：booru 向底模对 `matte / no sheen` 这类形容词响应很弱，但对**面料名词**响应强——`wool, cotton twill, flannel, tweed, corduroy`。同时避开工装感版型：`tight pants / skinny / leggings / spandex` 天然被渲染成反光，换 `straight-leg / loose / wide-leg`。
- **改裤子颜色**：纯黑是镜面高光最敏感的颜色。换炭灰 / 深棕 / 藏青，或加裤线、织纹打断高光。
- **局部 inpaint**：只涂裤子区域，prompt 只写面料名词，LoRA 权重降到 0.5，denoise 0.5。

### 破「抓衣襟」的其他杠杆

- **扣死衣襟**：正面写 `buttoned jacket, closed jacket front, fully buttoned`。衣襟开不了，这个动作就没有发生的余地（affordance removal）。这是一个很便宜的先试项。
- **ControlNet OpenPose 硬锁骨架**：用「双臂下垂」的骨架强制结构，weight 0.8、guidance end 0.6–0.7。
- **img2img 引导**：拿一张手放下的图做底，denoise 0.35–0.45。
- **权重网格**：1.0 / 0.85 / 0.7 / 0.6 各跑 10 张。0.6–0.75 时底模的姿势服从性通常会回来。

### 重训时的额外杠杆

- **caption dropout** 0.05–0.1
- **加正则图**：10–20 张同风格但不是她的图，压过拟合
- **alpha 减半 / 减少 epochs / early stop**：过拟合时降低容量
- **角色 LoRA 与服装 LoRA 分离**：换装需求频繁时，把衣服拆成独立 LoRA，从根上解决服装焊死
