<div align="center">

# 🎬 zero-to-story-suite

## 多人同框总漂移？先锁舞台坐标，再出宫格分镜
## Stop character drift: lock stage coordinates before the storyboard grid

> ⚠️ **这是一套配合使用的技能组合（2 个）** —— 正文互相引用，建议成套取用。

📮 **商务合作 / Business：wow.vip@foxmail.com**
🎬 **抖音 / Douyin：[v.douyin.com/07EjIDGVErY](https://v.douyin.com/07EjIDGVErY/)**

</div>

---

## 🇨🇳 中文

面向短片前期的双技能套件，专治没有现成剧本又多角色同框的情况。stage-position-map 为含 2 人以上角色的场景产出两件配对资产：机器可读的 Blocking Sheet 场景调度表（二维舞台坐标、朝向目标坐标、英文 spatial/facing 短语、Eyeline Map）和鸟瞰正交舞台定位图 PNG（坐标网格＋角色矩形色块＋道具圆形色块＋朝向箭头）。zero-to-story 接住这套坐标，引导故事设计、生成故事板 MD、角色设定图、黑白分镜图，确认提示词后生成视频。这些技能必须配合使用：调度表与定位图是空间锁定的唯一真值源，下游 9 宫格、黑白 2×4 或任意 N 宫格分镜只能复制坐标，禁止临时换算。

> ⚠️ **这是一套配合使用的技能组合（2 个）** —— 正文互相引用，建议成套取用。

### ✨ 能做什么

- 场景出场 ≥2 个角色时强制生成舞台定位图；独角戏先在对话里问用户是否需要，不需要就跳过
- 输出 Blocking Sheet：角色/道具二维舞台坐标、朝向目标坐标、spatial/facing 英文短语、Eyeline Map
- 输出鸟瞰正交定位图 PNG：坐标网格＋角色矩形色块＋道具圆形色块＋紫白短粗朝向箭头
- 用户没有剧本时用一次对话问齐故事类型等 6 个维度，再建立 Medium Lock 与 Style Brief
- 逐步产出故事板 MD → 角色设定图 → 黑白分镜图，提示词确认后才生成视频
- 坐标全链路沿用：stage_position_xy、facing_target_xy、screen_position、camera_facing 传进每个分镜 prompt

### 📦 包含的技能（配合使用）

- `stage-position-map` —— 见 [`stage-position-map/SKILL.md`](stage-position-map/SKILL.md)
- `zero-to-story` —— 见 [`zero-to-story/SKILL.md`](zero-to-story/SKILL.md)

### 🚀 安装

```bash
git clone https://github.com/yzsp-studio/zero-to-story-suite.git
cp -r zero-to-story-suite/* "$DSH_HOME/skills/"
```

> 技能是**纯文本说明**（`SKILL.md` + `references/`），零依赖、零 API key，用你当前 agent 的会话额度。

---

## 🇬🇧 English

A two-skill suite for short-film pre-production, built for teams with no finished script and multi-character scenes. stage-position-map produces two paired assets for scenes with 2+ characters: a machine-readable Blocking Sheet (2D stage coordinates, facing target coordinates, English spatial/facing phrases, an Eyeline Map) and a top-down orthographic stage position map PNG (coordinate grid plus character rectangles, prop circles and facing arrows). zero-to-story picks up those coordinates and walks through story design, a storyboard MD, character sheets, black-and-white panels, then video generation after prompt approval. The skills must be used together: the blocking sheet and stage map are the single source of truth, and every downstream 9-grid, 2×4 or N-grid panel copies coordinates instead of estimating them.

> ⚠️ **A cooperating suite of 2 skills** — they reference each other; use them as a group.

### ✨ What it does

- Forces a stage position map whenever a scene has 2+ characters; solo scenes ask first and can skip
- Blocking Sheet output: 2D character/prop stage coordinates, facing target coordinates, English spatial/facing phrases, Eyeline Map
- Top-down orthographic stage map PNG: coordinate grid, character rectangles, prop circles, short purple-white facing arrows
- With no script, one dialog collects 6 story dimensions, then sets the Medium Lock and Style Brief
- Stepwise delivery: storyboard MD → character sheets → black-and-white panel grids; video only after prompt approval
- Coordinates carry into every downstream prompt: stage_position_xy, facing_target_xy, screen_position, camera_facing

### 📦 Skills included (designed to work together)

- `stage-position-map` — [`stage-position-map/SKILL.md`](stage-position-map/SKILL.md)
- `zero-to-story` — [`zero-to-story/SKILL.md`](zero-to-story/SKILL.md)

### 🚀 Install

```bash
git clone https://github.com/yzsp-studio/zero-to-story-suite.git
cp -r zero-to-story-suite/* "$DSH_HOME/skills/"
```

> Skills are **plain-text instructions** (`SKILL.md` + `references/`). Zero dependencies, no API keys — they run on your agent's existing session quota.

---

## 📮 联系 / Contact

| | |
|---|---|
| 🎬 **抖音 / Douyin** | [https://v.douyin.com/07EjIDGVErY/](https://v.douyin.com/07EjIDGVErY/) |
| 📮 **商务合作 / Business** | **wow.vip@foxmail.com** |
| 🐙 **GitHub** | [https://github.com/yzsp-ai](https://github.com/yzsp-ai) |
| 💖 **打赏 / Support** | [afdian.com/a/yzsp-ai](https://afdian.com/a/yzsp-ai) |

## 📄 License

[MIT](LICENSE) · © 2026 [yzsp-ai](https://github.com/yzsp-ai)
