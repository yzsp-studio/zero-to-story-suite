---
name: stage-position-map
description: |
  drama 短剧 / 漫剧场景的空间锁定工具。当用户需要为含 ≥2 个角色的场景出一张"标准 90 度垂直俯视（鸟瞰正交平面）舞台定位图"时触发,
  自动产出两件配对资产：(1) Blocking Sheet 场景调度表（机器可读文字表,落画布）—— 角色 / 道具的二维舞台坐标、朝向目标坐标、英文 spatial / facing 短语、Eyeline Map；
  (2) 舞台定位图本体（PNG,落画布）—— 在场景俯视底图上叠加坐标网格 + 角色矩形色块 + 道具圆形色块 + 紫白色短粗朝向箭头 + 坐标指向文字。
  适用：剧本已经拆出场次和角色,镜头组里至少 1 个场景含 ≥2 人调度,即将进入分镜阶段（不论 9 宫格 / 黑白 2×4 / 任何 N 宫格）。
  不适用：单场景 1 人独角戏（agent 会先在 chat 框询问是否需要;用户说不需要就跳过）、纯对话音频内容、Seedance 视频派单（舞台定位图只服务分镜图,不传给 Seedance）。
  下游消费：所有宫格分镜 prompt 必须把舞台定位图作为 image_paths 之一,并同时引用 stage_position_xy (x,y) 坐标 + screen_position 画面方位。
  触发词包括：舞台定位图、场景调度表、blocking sheet、鸟瞰平面图、角色站位图、stage position map、多人场景定位、站位调度、坐标定位图、俯视调度图、blocking、多人镜头位置漂移。
summary-cn: 为多人场景生成调度表 + 鸟瞰俯视舞台定位图,锁分镜空间
summary-en: Blocking sheet + top-down stage position map for multi-character scenes
version: 0.1.0
tags: [Image, Drama, Storyboard, Reference, Blocking, Top-Down]
tags-cn: [图片, 短剧, 分镜, 参考, 调度, 俯视]
trigger-words:
  - 舞台定位图
  - 场景调度表
  - blocking sheet
  - blocking
  - 鸟瞰平面图
  - 角色站位图
  - stage position map
  - 多人场景定位
  - 站位调度
  - 坐标定位图
  - 俯视调度图
  - 多人镜头位置漂移
  - 角色站位
  - 道具坐标
  - eyeline map
author: yzsp-ai
repository: https://github.com/yzsp-ai
---


# 舞台定位图 — Blocking Sheet + 鸟瞰俯视平面图

> 为 drama 短剧 / 漫剧中**含 ≥2 个角色的场景**生成两件配对资产：(1) 场景调度表（文字,落画布,机器可读）+ (2) 舞台定位图（PNG,落画布,鸟瞰正交平面）。
> 这两件资产是空间锁定的**唯一真值源**——下游所有宫格分镜（9 宫格 / 黑白 2×4 / 1 宫格 / 任何 N 宫格）只能从这里复制坐标和朝向,禁止从"画面左 / 画面右 / facing north"临时换算。
> 本 skill 不是终点，而是 zero-to-story 的前置半链路；如果用户接下来要做故事板、黑白分镜或 Panel 提示词，必须继续接 `zero-to-story`，把 `stage_position_xy`、`facing_target_xy`、`screen_position`、`camera_facing` 沿用到后续流程。

---

## 触发条件

| 条件 | 生成？ |
|---|---|
| 本场景出场角色 ≥ 2 人 | **必须生成**（不可跳） |
| 本场景仅 1 人 | **可选** —— 在 chat 框先问用户「本场景只有 1 个角色,是否需要生成舞台定位图？」用户说不需要就跳过 |

**前置依赖**：
- 场景四宫格参考图已生成（含俯视视角,作为舞台定位图的底图）
- 角色参考图（角色卡 / 三视图 / 六视图）已生成
- 道具参考图（道具卡）已生成（若本场景有可见 / 被操作道具）

未达前置依赖 → 先回到 film-assets / character-card-gen 等 skill 把缺的资产补齐,再回来跑本 skill。

---

## 工作流

本 skill 分两步,**必须按顺序跑**:**先 Blocking Sheet（文字）→ 再舞台定位图（图）**。
舞台定位图直接抄 Blocking Sheet 的坐标和朝向,不允许颠倒顺序、不允许同时生成。

### 与 zero-to-story 的强制链路

本 skill 完成后不得把舞台定位图当作孤立终点；若用户目标是故事板、黑白分镜、Panel 视频提示词或后续视频生成，必须继续进入 `zero-to-story`。

#### 双 Skill 互锁触发（HARD）

只要当前任务命中 `stage-position-map` 或 `zero-to-story` 任意一个 skill，并且任务涉及多人场景、空间锁定、黑白 2×4 宫格分镜、故事板、Panel 提示词或视频生成，即视为触发**双 skill 强制链路**：

```text
stage-position-map → zero-to-story
```

- 只加载了 `stage-position-map`：若后续要做故事板 / 黑白 2×4 / 宫格分镜 / 视频提示词，必须继续加载并执行 `zero-to-story`；不得停在舞台定位图。
- 只加载了 `zero-to-story`：若场景含 ≥2 个角色，或需要继承角色 / 道具方位，必须先回到并加载 `stage-position-map`；没有 Blocking Sheet + 舞台定位图，禁止生成黑白 2×4 宫格分镜。
- 两个 skill 都已加载：进入黑白 2×4 宫格分镜前，必须触发下方“坐标继承硬门槛”和“生成后视觉 QC”。

- 完成舞台定位图后，必须把 `stage_position_xy`、`facing_target_xy`、`screen_position`、`camera_facing`、Eyeline Map、角色 / 道具相对距离关系交给 `zero-to-story` 沿用。
- `zero-to-story` 不得重算这些坐标和朝向，只能复制、继承、转写为分镜画面层与镜头朝向层。
- 如果用户只单独调用本 skill，但后续任务明显是分镜 / 故事板 / 黑白宫格 / 视频提示词，必须用弹窗式 `question` 提醒：链路未完成，需要继续调用 `zero-to-story`。
- 如果用户只单独调用 `zero-to-story`，但缺少本 skill 的舞台定位图 / Blocking Sheet，也必须弹窗提醒先回到 `stage-position-map`。

#### 黑白 2×4 宫格坐标继承硬门槛（HARD）

任何黑白 2×4 宫格分镜 prompt，在派 image agent 之前，必须逐格写入一个不可省略的 `Spatial Lock Block`。每个 panel 至少包含：

```text
panel_N_spatial_lock:
  stage_geometry: <从 Blocking Sheet 的场景空间结构逐字继承>
  visible_subjects:
    - <角色名>: stage_position_xy=(x,y), facing_target_xy=(x,y), screen_position=<前景/中景/背景/环境锚点>, camera_facing=<面对/背对/侧对镜头等>
  visible_props:
    - <道具名>: stage_position_xy=(x,y), state=<被操作/可见/背景>
  stage_note: camera moved, subject NOT moved. Only explicit moving subject(s): <角色名 from (x1,y1) to (x2,y2)>.
  screen_direction_lock: <上一格到本格的动作方向 / 视线方向 / 受力方向，不得越轴>
```

硬规则：
- 每格都必须出现 `stage_position_xy` 和 `facing_target_xy`，不能只在总说明里写一次。
- 若角色不在某格出现，才可省略该角色；但出现的每个角色必须写坐标与朝向。
- 只有剧本明确要求移动的角色才能写 from→to；其他角色必须写 `camera moved, subject NOT moved`。
- 舞台定位图必须作为 image_paths 最后一位传入，并在 prompt 中明确：只用于核验坐标、朝向、相互距离和移动轨迹，禁止渲染网格 / 坐标 / 箭头 / 地图边框进面板。
- 缺任一 panel 的 `Spatial Lock Block` = 禁止派单。

#### 黑白 2×4 生成后视觉 QC（HARD）

黑白 2×4 宫格图生成后，agent 必须用视觉检查核对：

- 角色 / 道具相对位置是否与舞台定位图一致。
- 角色朝向、视线、受力方向是否与 `facing_target_xy` 一致。
- 连续面板是否越轴；同一动作方向不得在相邻格反向跳变。
- 明确移动角色是否沿 Blocking Sheet 的 from→to 轨迹移动。
- 非移动角色是否未因镜头切换发生“瞬移”。
- 若有主观转场 / 意识下沉 / 记忆闪回，视觉中心必须锁在 Blocking Sheet 指定的 `stage_position_xy`，不能漂到场景中轴或随机中心。

任一项失败 → 必须重写 prompt 并重新生成黑白 2×4 宫格图；禁止把失败图标记为通过。

### 步骤 1 —— 生成 Blocking Sheet 场景调度表

**输出位置**：用 `hub_canvas_write_text_node` 写一个文本节点,节点名固定为 `场景调度表 — <场景编号 + 场景名>`（例：`场景调度表 — 01-01 病房戏`）。

**坐标系约定**（agent 必须遵守）：

- 场景中心 = `(0, 0)`
- x 轴左负右正,y 轴下负上正
- 每 5 单位 1 刻度,范围按场景比例选 -35~35
- 默认参考点：左 1/3 ≈ `(-15, 0)`,右 1/3 ≈ `(15, 0)`,北侧 ≈ `(x, 5~10)`,南侧 ≈ `(x, -5~-10)`,东侧 ≈ `(15~25, y)`,西侧 ≈ `(-15~-25, y)`
- 重点是**相对位置关系正确 + 朝向箭头正确**,不追求物理精确比例

### 坐标层 / 画面层 / 镜头朝向层分离规则（FAIL 铁律）

本 skill 同时服务宫格分镜图和 BEAT / Seedance Prompt。为避免“上下左右、正面、背面、侧面”等词造成模型误判，所有空间表达必须拆成三层，禁止混用。

#### 1. 坐标层：唯一空间真值，不得被替代

`stage_position_xy`、`facing_target_xy`、任何 `(x, y)` 坐标，都是 Blocking Sheet 的机器可读真值源。

- 禁止用“前景 / 中景 / 背景 / 画面黄金分割点 / 画面边缘”替代 `(x, y)` 坐标。
- 禁止用“面对镜头 / 背对镜头 / 侧对镜头”替代 `facing_target_xy`。
- 禁止删除、弱化、改名 `stage_position_xy` 与 `facing_target_xy` 字段。
- 若自然语言方位与坐标冲突，以坐标层为准；自然语言层必须重写。

错误示例：
- `stage_position_xy = 前景`
- `facing_target_xy = 面对镜头`
- `角色 A 在画面边缘，所以不写坐标`

正确示例：
- `stage_position_xy = (-15, 5)`，画面层描述为“位于前景靠近画面黄金分割点”
- `facing_target_xy = (15, 5)`，镜头朝向层描述为“人物身体微侧，脸部四分之三侧对镜头”

#### 2. 画面层：只描述观众看到的构图位置

宫格分镜与画面描述中，优先使用以下词汇表达画面位置：

- 前景
- 中景
- 背景
- 画面黄金分割点
- 画面边缘
- 靠近 / 远离某个环境元素
- 被某个前景物局部遮挡
- 位于人物关系的视觉重心

避免把“画面左 / 画面右 / 上方 / 下方 / 正面 / 反面 / 左面 / 右面”作为唯一定位依据。如确需说明画面边缘方向，也必须同时绑定构图层或环境锚点，例如“靠近画面边缘的门框旁”，不能只写“在左边”。

#### 3. 镜头朝向层：以镜头 / 观众为参照系

人物身体朝向、脸部可见度、背影关系，必须以摄影机或观众为参照，而不是用“正面 / 背面 / 侧面”裸词。

| 禁用裸词 | 推荐写法 |
|---|---|
| 正面 | 人物面对镜头；直视观众；脸部正对摄影机 |
| 背面 / 反面 | 人物背对镜头；我们只能看到他的后背和后脑勺 |
| 侧面 | 人物以完全侧身轮廓出现，只能看到半边脸 |
| 四分之三侧面 | 人物身体微侧，半张脸被阴影遮挡，呈经典四分之三侧面肖像 |
| 上下位 | 俯拍，人物抬头仰望镜头；仰拍，人物低头俯视镜头 |

#### 4. 三层同时出现时的写法

同一角色在 Blocking Sheet / 分镜 Locked Block 中推荐写成：

```text
stage_position_xy: (-15, 5)
facing_target_xy: (15, 5)
screen_position: 前景靠近画面黄金分割点，靠近病床边缘
camera_facing: 人物身体微侧，脸部四分之三侧对镜头，视线落向对面角色
```

其中：
- `stage_position_xy` / `facing_target_xy` 给宫格分镜和舞台定位图使用。
- `screen_position` 给分镜画面构图使用。
- `camera_facing` 给 BEAT Prompt / Seedance Prompt 使用。
- 任一层缺失或互相替代 = FAIL。

**调度表模板**（agent 按场景实际填充）：

```markdown
# 场景调度表 — <场景编号 + 场景名>

## 场景空间结构
- 坐标系：场景中心 = (0,0),x 轴左负右正,y 轴下负上正；每 5 单位 1 刻度,范围按场景比例选 -35~35
- 物理空间：<尺寸 / 朝向 / 主体物>
- 朝向锚点：<方向参照,如 北窗 / 南门 / 主席台朝东>
- 摄影机相对方位：<摄影机位置基线,如 "camera at scene south, facing north" / "handheld over-shoulder right side of 简化">

## 角色站位（机器可读：每行必须同时带坐标 + 方位；坐标给宫格分镜,方位短语给 BEAT Prompt）

| 角色 | 舞台坐标 `stage_position_xy` | 基础位置（中文）| 基础位置（英文 spatial phrase）| 朝向目标坐标 `facing_target_xy` | 坐标+方位朝向短语 | 朝向（英文,含 facing direction + body angle）| 与他人距离 | 状态 / 动作 |
|---|---|---|---|---|---|---|---|---|
| <角色 A> | `(-15, 5)` | <位置描述> | `west side of bed; frame-left only from south camera` | `(15, 5)` | `面朝<角色B>(15,5)东看` | `facing east toward Subject 2, body turned 30° to camera` | <距离描述> | <状态描述> |
| <角色 B> | `(15, 5)` | <位置描述> | `east side of bed; frame-right only from south camera` | `(-15, 5)` | `面朝<角色A>(-15,5)西看` | `facing west toward Subject 1, eyes downcast at 简化's hand` | <距离描述> | <状态描述> |

## 道具坐标（机器可读：宫格分镜可见 / 被操作道具必填）

| 道具 | 舞台坐标 `stage_position_xy` | 基础位置（中文）| 基础位置（英文 spatial phrase）| 朝向目标坐标 `facing_target_xy` | 坐标+方位朝向短语 | 状态 |
|---|---|---|---|---|---|---|
| <道具 A> | `(0, 15)` | <位置描述> | `center of north wall altar` | `N/A` | `无明确朝向` | <状态描述> |

## Eyeline Map（互动方位 → 每对角色 1 行机器可读 eyeline 短语）

| 视线方向 | 中文描述 | 英文 Eyeline phrase（直接抄入 BEAT Prompt `[Eyeline]` 段）|
|---|---|---|
| <A> → <B> | <方向描述> | `Subject 1 sightline directed rightward, fixed on Subject 2's face` |
| <B> → <A> | <方向描述> | `Subject 2 sightline directed leftward, locked on Subject 1's hand grip` |
| <A> → 道具/方位 | <例如 看向手机> | `Subject 1 sightline drops to phone screen at lower frame center` |

## 镜头组使用规则（铁律）
- 同场景**所有**宫格分镜 prompt **每格**必须从本表抽取 `stage_position_xy`；BEAT Prompt **每镜（不只是首镜）**只抽取人类可读的 `[Spatial Anchor]` + `[Eyeline]` 方位短语,避免把裸 `(x,y)` 坐标写给 Seedance。
- ≥2 人镜头**任何一镜**漏写 `[Spatial Anchor]` 或 `[Eyeline]` 段 → 禁止派单。
- 角色 / 道具坐标在镜头组内**禁止漂移**（不允许 Subject 1 第 1 镜 `(-15,5)`、第 2 镜变成 `(15,5)`）；若剧本要求位置改变 → 必须在分镜表明确写"移动镜"（如 `Subject 1 walks from (-15,5) to (15,5)`）+ Blocking Sheet 加该角色"移动后位置"行。
```

**Blocking Sheet 写完自检**（agent 必跑,全 ✓ 才能进步骤 2）：

| 检查项 | 通过标准 |
|---|---|
| 角色清单 = 场内所有 ≥1 句台词角色 | 不缺人 |
| 每个角色行都有 `stage_position_xy` + `facing_target_xy` + 坐标+方位朝向短语 | 三件齐 |
| 每个角色行都有"基础位置（英文）" + "朝向（英文）" | 英文 spatial / facing phrase 齐,与坐标一致 |
| 宫格分镜可见 / 被操作道具都有 `stage_position_xy` | 道具坐标齐；无明确朝向可写 `N/A` |
| Eyeline Map 已列：场内任两角色间至少 1 行 + 每条都有英文 phrase | 互动方位齐 |
| 摄影机相对方位已锚定 | 英文 camera baseline 短语齐；默认补 `camera handheld, eye-level, facing scene from south` |
| 调度表位置 / 朝向 / 距离跟剧本一致 | 文字跟剧本一致 |

任一项 FAIL → 补完再进步骤 2。

### 步骤 2 —— 生成舞台定位图（鸟瞰正交平面图）

**输出位置**：图生成完毕后,用 `hub_canvas_write_media_node` 落画布,节点名固定为 `舞台定位图 — <场景名>`,文件命名 `<场景名>-stage-position-map.png`。

**底图**：以该场景四宫格中的**俯视角视图**作为底图（一般是场景四宫格右上角的俯视视角）。
若场景四宫格中无标准俯视视角 → 用正视视角 + prompt 指令要求 image 模型转为**标准 90 度垂直俯视（鸟瞰正交平面图）**渲染。

**角色 / 道具双重过滤规则**（每次生成舞台定位图前必过）：

舞台定位图上标注的角色和道具必须**同时**满足两个条件,缺一不可：

| 条件 | 含义 | 不满足时 |
|---|---|---|
| **条件 A：已生成资产卡** | 该角色 / 道具已经生成了参考图（角色卡 / 道具卡已落画布） | 不放入舞台定位图 |
| **条件 B：后续宫格分镜实际出场** | 该角色 / 道具在即将制作的宫格分镜（当前镜头组）中确实会出现 | 不放入舞台定位图 |

**双重过滤执行流程**：Blocking Sheet 全量角色 + 道具 → 逐个过条件 A + B → 都通过才标注；被剔除的记录原因。每次制作宫格分镜前重新过滤（不同组出场不同 → 定位图可能不同；出场完全相同 → 可复用）。

#### 模型路由（CRITICAL,硬约束）

舞台定位图**必须使用 `gpt-image-2`** 生成,派单时 `model` 字段**硬写 `"gpt-image-2"`**,**禁止走默认模型路由**。

原因：舞台定位图的核心价值是坐标网格 + 角色 / 道具名称文字标注 + 刻度数字 + 坐标指向文字,文字准确性是第一优先级。`gpt-image-2` 是当前中英文文字渲染最稳定的模型。默认路由会落到 banana pro,文字渲染不稳定。

#### 叠加元素规范

1. **二维坐标网格**：白色半透明网格线,x / y 轴交叉于场景中心,每 5 单位 1 刻度（-35 ... 0 ... 35）,不改变底图内容。
2. **角色定位标记**：**矩形色块**内标角色名,按出场顺序分配颜色（A = 红 / B = 橙 / C = 黄 / D = 绿 / E = 蓝）,位置逐字复制 Blocking Sheet 角色站位表的 `stage_position_xy`,旁附**紫白色短粗朝向箭头 + 坐标指向文字**。
   - 箭头线宽约为网格线 **3-4 倍**,箭头头部清楚,长度约为角色标记宽度 **1.0-1.5 倍**
   - 只表达朝向,不表达运动
   - 坐标指向文字格式：`面朝{{目标名}}({{target_x}}, {{target_y}}){{方位}}看`,例如 `面朝灵位牌(0,15)北看`
3. **道具定位标记**：**圆形色块**内标道具名,颜色同上循环分配,位置逐字复制 Blocking Sheet 道具坐标表的 `stage_position_xy`。
   - 有明确朝向的附紫白色短粗箭头 + 坐标指向文字（规格同角色箭头）
   - 无明确朝向的道具不标箭头

#### image agent 派单 prompt 模板

```
任务：基于场景俯视参考图,生成 1 张标准 90 度垂直俯视（鸟瞰正交平面）舞台定位图。

底图：[Match Ref Image 1]（场景俯视参考图）

叠加内容（在底图上添加,不改变底图场景内容）：
1. 二维坐标网格：白色半透明网格线,x/y 轴交叉于场景中心,每 5 单位 1 刻度,轴线标注 x/y 字母 + 刻度数字
2. 角色定位标记（仅标注通过双重过滤的角色）：
   - 红色矩形「{{角色A名}}」位于坐标 ({{xA}}, {{yA}}),紫白色短粗箭头面朝{{朝向目标名}}({{target_x}}, {{target_y}}){{方位}}看；箭头线宽约为网格线 3-4 倍,长度约为角色标记宽度 1.0-1.5 倍,箭头头部清楚
   - 橙色矩形「{{角色B名}}」位于坐标 ({{xB}}, {{yB}}),紫白色短粗箭头面朝{{朝向目标名}}({{target_x}}, {{target_y}}){{方位}}看；箭头线宽约为网格线 3-4 倍,长度约为角色标记宽度 1.0-1.5 倍,箭头头部清楚
   - （按通过双重过滤的实际角色数列举；朝向必须用坐标指向写法,禁止用裸方向词如"朝北 / 朝南 / facing north"）
3. 道具定位标记（仅标注通过双重过滤的道具）：
   - 红色圆形「{{道具A名}}」位于坐标 ({{x}}, {{y}}),紫白色短粗箭头面朝{{朝向目标名}}({{target_x}}, {{target_y}}){{方位}}看；箭头线宽约为网格线 3-4 倍,长度约为道具标记宽度 1.0-1.5 倍,箭头头部清楚
   - （按通过双重过滤的实际道具数列举；无明确朝向的道具不标箭头）

画面要求：标准 90 度垂直俯视鸟瞰视角,坐标系清晰可读,标记不遮挡关键场景元素,各标记颜色区分度高。所有中文角色名 / 道具名 / 坐标数字必须清晰无误。朝向箭头必须短粗高对比,线宽明显粗于坐标网格线,长度只够表达方向。
```

派单参数：

| 字段 | 值 |
|---|---|
| `model` | `gpt-image-2`（硬写,禁默认路由） |
| `image_paths[1]` | 场景俯视参考图（或场景四宫格作为底图来源） |
| `image_paths[2..N]` | 通过双重过滤的角色卡 + 道具卡(用于 image 模型确认外观,不必逐个 `[Match Ref Image N]` 引用,但要传) |

#### 生成后自检

agent 收到图后必跑,全 ✓ 才落画布；任一项 FAIL → 重新生成（仍用 `gpt-image-2`）：

- [ ] 角色间相对位置与 Blocking Sheet 一致（如"A 在 B 左前方"在图上确实如此）
- [ ] 每个角色的朝向箭头 + 坐标指向文字与 Blocking Sheet 一致（如"沈栖月面朝灵位牌(0,15)北看"在图上箭头确实从沈栖月位置指向灵位牌方向）
- [ ] 每个朝向箭头都是短粗高对比样式：线宽约为网格线 3-4 倍,箭头头部清楚,长度约为对应角色 / 道具标记宽度 1.0-1.5 倍
- [ ] 道具位置与剧本描述一致（如"灵位牌在北墙正中"确实在图上方中央）
- [ ] 角色 / 道具坐标值与 Blocking Sheet 的 `stage_position_xy` 逐字一致
- [ ] 坐标刻度数字清晰可读、无乱码

---

## 退出 Gate（进入分镜阶段前必查）

舞台定位图是 drama workflow STEP 4 资产层的**退出门槛**之一,该 Gate 必须在进入 STEP 5 分镜之前过：

| Gate 项 | 要求 | 不过的后果 |
|---|---|---|
| 所有 ≥2 人场景已落画布「场景调度表 — <场景名>」| 缺任一场景 = 该场景调度表 FAIL | 禁止进入分镜、禁止生成舞台定位图、禁止生成任何宫格分镜图 |
| 所有 ≥2 人场景已落画布「舞台定位图 — <场景名>」| 缺任一场景 = 该场景定位图 FAIL | 分镜阶段 §5.2 阶段 A 会**双重拦截**：顶部铁律 Step A-1 + §5.2 阶段 A 起手检查双拦截,逼 agent 回到本 skill 把图补齐 |

**Gate 触发时机**：
- agent 在 STEP 4 资产层完成所有角色卡 / 场景卡 / 道具卡之后,准备进入 STEP 5 分镜之前,必须扫一遍所有 ≥2 人场景,逐一确认两件资产都已落画布
- agent 在 STEP 5 §5.2 任何宫格分镜阶段 A（Locked Block 写作前）也会再做一次拦截,这是双重保险

**Gate 失败时的恢复动作**：
- agent 必须回到本 skill 把缺的 Blocking Sheet 或舞台定位图补出来,然后再继续 STEP 5
- 不允许"跳过本场景"或"先生成分镜再补图"

---

## 输出与下游消费

| 资产 | 形式 | 落画布命名 | 文件命名 |
|---|---|---|---|
| Blocking Sheet 场景调度表 | 文本节点（markdown） | `场景调度表 — <场景名>` | —— |
| 舞台定位图 | 媒体节点（PNG） | `舞台定位图 — <场景名>` | `<场景名>-stage-position-map.png` |

每个含 ≥2 人的场景 1 套,跟场景四宫格平级存储。

### 下游消费规则

**分镜图（任何 N 宫格）**：
- 舞台定位图作为 `image_paths` 数组的一员传入 image agent,在 prompt 中用 `[Match Ref Image N]` 引用,标注为"空间定位参考"
- 分镜图 prompt 必须**同时**使用 `screen_position`（画面方位）+ `(x, y)` 坐标（舞台坐标）引用角色 / 道具位置
- `screen_position` 必须使用画面层词汇：前景 / 中景 / 背景 / 画面黄金分割点 / 画面边缘 / 环境锚点；不得只写“画面左 / 画面右 / 上方 / 下方”作为唯一定位
- 人物朝向必须使用镜头朝向层：面对镜头 / 背对镜头 / 侧对镜头 / 四分之三侧对镜头 / 俯拍抬头 / 仰拍俯视；不得只写“正面 / 背面 / 侧面 / 反面”
- 以上自然语言层只服务画面构图和镜头理解,绝不得替代 `stage_position_xy`、`facing_target_xy` 或舞台定位图中的 `(x, y)` 坐标
- 只有画面方位、没有坐标 = FAIL
- **制作任何分镜图（1 / 4 / 9 / 15 / 25 宫格,以及黑白 2×4）之前必须先有舞台定位图**

**BEAT Prompt（每镜）**：
- 从 Blocking Sheet 的"角色站位"英文 spatial phrase 列、"朝向（英文）"列、Eyeline Map 英文 phrase 列**逐字抄入** BEAT Prompt 的 `[Spatial Anchor]` + `[Eyeline]` 段
- ≥2 人镜头任一镜漏写 `[Spatial Anchor]` 或 `[Eyeline]` 段 → 禁止派 Seedance

**Seedance 视频**：
- **不传舞台定位图**。舞台定位图只服务分镜图制作,Seedance 不需要此参考。
- Seedance 拿 BEAT Prompt 里的 `[Spatial Anchor]` + `[Eyeline]` 自然语言方位短语就够,不写裸 `(x,y)` 坐标。

### 与文字 stage_geometry / body_facing 的关系

互补不冲突：
- **舞台定位图** = 空间锁定的**视觉层**（image 模型直接看图）
- **`stage_geometry` / `stage_position_xy`** = 宫格分镜 image prompt / Locked Block 的**语义层**兜底

两层叠加才是完整空间锁定。缺任一层 → 镜头组内角色 / 道具位置会漂移。

---

## 关键约束清单（一句话版）

- **顺序铁律**：先 Blocking Sheet（文字）→ 再舞台定位图（图）,禁颠倒、禁同时
- **场景门槛**：≥2 人必生 / 1 人可选（先问用户）
- **坐标真值源**：Blocking Sheet 是唯一真值,舞台定位图只复制不重算,下游分镜只复制不重算
- **模型硬写**：舞台定位图必须 `gpt-image-2`,禁默认路由
- **双重过滤**：标在图上的角色 / 道具必须同时满足"已生成资产卡" + "本组分镜实际出场"
- **箭头样式**：紫白色短粗,线宽约网格线 3-4 倍,长度约标记宽度 1.0-1.5 倍,只表达朝向
- **朝向写法**：必须用坐标指向格式 `面朝{{目标名}}({{x}}, {{y}}){{方位}}看`,禁裸方向词
- **画布命名**：`场景调度表 — <场景名>` + `舞台定位图 — <场景名>`,严禁改名
- **下游绑定**：所有 N 宫格分镜（含黑白 2×4）必须把舞台定位图传入 `image_paths`；Seedance 不需要此图
- **位置漂移禁**：角色 / 道具坐标在同一镜头组内禁漂移；剧本要求移动 → 在分镜表明确写"移动镜" + Blocking Sheet 加"移动后位置"行

---

## 4 层空间锁完整链路（仓库铁律,本 skill 是第 4 层）

镜头切换时角色翻转 / 瞬移 / screen direction 漂移,是 image 模型理解"相对镜头方位"的固有缺陷。仓库 drama workflow 用 **4 层空间锁**联防,本 skill（舞台定位图）是第 4 层。缺任一层 → 镜头组内角色 / 道具位置会漂移。

| 层 | 名字 | 来源 | 形式 | 作用 |
|---|---|---|---|---|
| 1 | `stage_geometry` 场景方向锚点 | Blocking Sheet 的"场景空间结构"段 | 文字 | 锁场景的物理朝向参照系（北墙 / 南门 / 主席台朝东） |
| 2 | `stage_position_xy` + `body_facing` 绝对位置 / 朝向 | Blocking Sheet 的"角色站位"表 + "道具坐标"表 | 文字坐标 | 锁绝对舞台位置(x,y) + 朝向目标坐标,不随镜头变 |
| 3 | `stage_note` 每帧 camera moved / subject NOT moved | 分镜 Locked Block 的逐帧 stage_note 字段 | 文字 | 每帧解释"是镜头动 / 不是人动",防 image 模型把镜头切换误判为人物移动 |
| 4 | 舞台定位图 | 本 skill 产出的 `<场景名>-stage-position-map.png` | 视觉图 | image 模型直接"看到"全场空间,视觉层兜底,与文字层互补 |

**联防铁律**：
- 任何 ≥2 人镜头组,4 层必须全到位才能派分镜图
- 分镜 prompt 必须同时引用 4 层信息：`stage_geometry`（场景锚点）+ `stage_position_xy`（坐标）+ `stage_note`（镜头移动说明）+ 舞台定位图（`[Match Ref Image {{LAST}}]`）
- 缺任一层 = 派分镜 FAIL,回本 skill 补图

---

## 模型路由全景（drama workflow 内的硬约束）

drama 短剧 / 漫剧的资产生成有两条互不冲突的模型路由,本 skill 涉及第一条：

| 资产类型 | 模型 | 原因 | 派单字段 |
|---|---|---|---|
| **舞台定位图（本 skill）** | `gpt-image-2`（硬写） | 坐标网格 + 名称文字 + 刻度数字 + 坐标指向文字,文字渲染准确性第一 | `model: "gpt-image-2"` |
| **所有宫格分镜图**（含 9 宫格、1 宫格、黑白 2×4 及任何 N 宫格） | `gpt-image-2`（硬写） | 精确格线布局 + 格内编号 + 角色站位坐标指向文字,文字渲染同理 | `model: "gpt-image-2"` |
| **角色卡 / 场景卡 / 道具卡 / 定妆照 / 选角参考图等其它前期美术图** | `nano_banana_2`（硬写,即 banana pro） | 美术质感、人物面部、风格统一,banana pro 是 drama workflow 默认 | `model: "nano_banana_2"` |

**反向铁律**：
- 不得因为"文字标注"、"默认路由"或"fallback 习惯"把**前期美术图**切到 `gpt-image-2`
- 不得因为"已经在用 banana pro"或"省事"把**舞台定位图 / 宫格分镜图**切到 banana pro（banana pro 文字渲染不稳定,会拼错角色名 / 漏坐标 / 数字错位）
- 不得走默认模型路由,两类资产派单时 `model` 字段都必须**硬写**

---

## 宫格分镜引用规范（本 skill 产出图的下游消费契约）

宫格分镜图（任何 N 宫格,含黑白 2×4）的 image agent 派单 prompt 必须按下面规范引用舞台定位图。本 skill 的输出格式与这个规范严格绑定,改字段名 / 改命名 / 改位置都会让下游分镜阶段直接 FAIL。

### `image_paths` 数组位置（CRITICAL）

舞台定位图必须放 `image_paths` 数组的**最后一位**（约定下标 = `{{LAST}}`）：

```text
image_paths = [
  [1] <角色 A 六视图>,
  [2] <角色 B 六视图>,
  [3] <角色 C 六视图>,  # 如有
  [4] <场景四宫格>,
  [5] <道具 A 视图>,    # 如有
  [6] <道具 B 视图>,    # 如有
  ...
  [{{LAST}}] <场景名>-stage-position-map.png   # 舞台定位图,本 skill 产出,放最后
]
```

**为何放最后**：分镜 prompt 在 Phase 1 用 `[Match Ref Image {{LAST}}]` 引用,数组长度变化时只需要更新一个下标。

### 分镜 prompt Phase 1 的 4 处必填引用位

宫格分镜 prompt 的 Phase 1 角色 / 空间锁定段必须按以下顺序写入 4 处引用,缺任一 = FAIL：

1. **Reference Legend 段**：舞台定位图必须明确标注 "x-y coordinate facing map（场景名 正俯视鸟瞰布局图）：只用于核验每个角色 / 道具的 `stage_position_xy`、`facing_target_xy`、短粗高对比坐标指向箭头、坐标指向文字和相互距离关系；**不要将俯视图、网格线、坐标文字、箭头或地图边框渲染进任何宫格面板**"
2. **Section 6 — Spatial Anchor**（≥2 人组必填）：坐标 + 方位词双重标准,与舞台定位图对齐
3. **Section 7c — Absolute Body Facing Lock**（HARD）：坐标指向写法,overrides any camera-induced directional ambiguity
4. **Section 7d — 舞台定位图段**（≥2 人组必填）：`use [Match Ref Image {{LAST}}] as overhead layout reference`

### 反向渲染约束

舞台定位图作为 image_paths 参考传入时,prompt 必须**反向声明**：

- 不要把舞台定位图的俯视视角渲染进任何宫格面板
- 不要把网格线、坐标文字、坐标数字、朝向箭头、地图边框渲染进任何宫格面板
- 宫格分镜的格内画面应该是"角色实际镜头视角"的画面,舞台定位图只是给 image 模型看的"幕后地图",绝不能渲染出来

漏写反向约束 → image 模型容易把舞台定位图风格直接画到第 1 格,整张宫格分镜 FAIL。

### 对账表必查 3 行（分镜阶段 §5.2.2.2）

宫格分镜派单前的对账表里,与舞台定位图相关的 3 行必查：

| 行 | 检查项 | 通过标准 |
|---|---|---|
| 7 | **数量自洽**：出场资产 ref 数 + 舞台定位图 ref 数 = `image_paths` 数组长度 = Phase 1 不同 N 数量 | 三者相等 |
| 8.5 | **无孤儿 ref**：`image_paths` 里每个下标 N 都被 Phase 1 至少 1 条 `[Match Ref Image N]` 引用（舞台定位图在 Section 7d 引用也算有效） | 无孤儿 |
| 21 | **舞台定位图已传入 image_paths？**（≥2 人组必传,放最后一位）+ Phase 1 Section 7d 已引用 + Reference Legend 标注"空间定位参考" | `[Ref {{LAST}}] = <场景名>-stage-position-map.png ✓ / 7d ✓ / Legend ✓`；单人组未生成时填 `N/A` |

---

## BEAT Prompt + Seedance 链路约束（避免下游误传）

舞台定位图**只服务分镜图制作**,在 BEAT Prompt（STEP 6）和 Seedance 视频派单（STEP 7）阶段**绝不能再出现**。

### BEAT Prompt 阶段

- BEAT Prompt 的【参考资产引用】段**禁传舞台定位图**
- BEAT Prompt 只保留人类可读的 `[Spatial Anchor]` + `[Eyeline]` 自然语言方位短语（从 Blocking Sheet 的"基础位置（英文）" + "朝向（英文）" + Eyeline Map 英文 phrase 列逐字抄入）
- BEAT Prompt 必须继承 Blocking Sheet 的自然语言三层映射：`screen_position` 使用前景 / 中景 / 背景 / 黄金分割点 / 画面边缘 / 环境锚点；`camera_facing` 使用面对镜头 / 背对镜头 / 侧对镜头 / 四分之三侧对镜头等镜头参照表达
- ≥2 人镜头任一镜漏写 `[Spatial Anchor]` 或 `[Eyeline]` → 禁派 Seedance

### Seedance Prompt 阶段

- Seedance 派单**不传**舞台定位图（舞台定位图不占 Seedance ref 名额）
- Seedance 派单**不复制裸 `(x, y)` 坐标** —— 坐标只是分镜图的空间真值,迁移到 Seedance 时必须转成自然语言 `[Spatial Anchor]` / `[Eyeline]` / 位置关系
- Seedance 的自然语言方位必须使用镜头 / 观众参照系：面对镜头、背对镜头、侧对镜头、四分之三侧对镜头、俯拍抬头、仰拍俯视；不得只写“正面 / 背面 / 侧面 / 反面”
- 分镜组脚本里的 `stage_position_xy` / `facing_target_xy` 属于"脚本自检 + 舞台定位图 + 宫格分镜图的空间真值",不进 Seedance prompt
- 反例：Seedance prompt 出现 `Subject 1 at (-15, 5)` = FAIL,必须改写为 `Subject 1 stands at west side of bed, frame-left from south camera`

### 链路总览

```text
Blocking Sheet (文字坐标真值)
  ├── 舞台定位图 (视觉层,本 skill 产出,只服务分镜图)
  │     └── 宫格分镜图 (image_paths 含舞台定位图 + 引用 Section 7d)
  │           └── BEAT Prompt (只抽自然语言 Spatial Anchor / Eyeline,禁传舞台定位图)
  │                 └── Seedance 视频 (只用 BEAT,禁传舞台定位图,禁裸坐标)
```

链路上每一节点都不能跳层、不能反向引用、不能把上游真值原样塞到下游。
