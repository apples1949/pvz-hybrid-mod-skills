# 僵尸 Mod 开工前：需要哪些素材（部位 / 格式 / 规格）

> 本文件是 `pvz-hybrid-zombie-authoring` 的 reference。
> 与 `zombie-skin-and-head.md` 的分工：
> **本文件 = 动手前「要什么」**（需求侧问卷 + 规格基准 + 素材来源 + 没有素材的降级路线）；
> 那份 = 拿到之后「怎么用」（换头三节点、`.dat` 二进制规格、头对位反解、位姿同帧）。
> 所有数值均为**实测**，出处随行标注。

---

## 0. ★★★ 铁律：调用本技能的第一件事是「问素材」

**任何**「做个僵尸 Mod / 把僵尸换成 XX」的请求，**先问素材，再谈方案**。

理由：素材形态直接决定可行性、工作量与路线选择——

| 用户手上的东西 | 实际能走的路 | 工作量 |
|---|---|---|
| 什么都没有（只是「做个会发射的僵尸」） | **复用原版**外观，只改数值 + 加组件/插件 | 最小 |
| 1 张参考图 | 抠像 → 逐帧 → 自制 `.dat`（另见 `pvz-hybrid-mod-authoring` 的单图管线） | 中 |
| 要说「身体用 A、头用 B」 | 换头「三节点」 | 中 |
| 经典版 `*.reanim.compiled` | ★ **直转**（首选，帧数据已有） | 小 |
| 逐帧 PNG 序列 | 直接进图集 | 小（但要自己定帧尺寸） |
| 部件特写若干张（头/体节/尾…） | 拼链（见 SKILL.md §4b.5） | 大 |

**同时必须主动告知「可以复用原杂交版已有的素材」** —— 用户通常不知道原版素材可直接拿来用，
也不知道「换头」比「全新素材」省事得多。

### 0.1 可直接照抄的问法（按回答分支，别一口气问完就动手）

```
开工前先确认下素材（没有也没关系，多数情况可以直接复用原杂交版已有的素材）：

1) 外观想要什么效果？
   a. 直接复用一个原版僵尸（说名字或给张图，我从 Chapter1 系列里挑最像的）
   b. 换头：身体用原版 <X>，头用 <Y>
   c. 全新素材：给我【经典版 reanim 文件】或【逐帧 PNG 序列】或【一张参考图】（三选一）
2) 有部件图吗？（头 / 身体 / 尾 / 其他部位分开给的）—— 有请一起发，并说清哪张是哪个部位、哪边是朝向
3) 护具要不要？用原版哪一款（路障 / 铁桶 / 报纸 / 铁门…），还是自己给图？
4) 要发射吗？要的话弹丸外观用原版哪种（豌豆 / 火球 / …），还是自己给图？
5) 有音效吗？（没有就用原版或静音）
```

> ⚠️ **别自己替用户决定**：素材形态一变，后面 80% 的工作都不一样（见 SKILL.md §1「四个决定」）。
> 也**别要求用户先做「图集」** —— 图集排版是脚本的事（官方口径见 §2）。

---

## 1. 素材清单总表（部位 / 格式 / 规格）

| # | 部位 | 用途 | 推荐格式 | 规格硬要求 | 必需性 |
|---|---|---|---|---|---|
| 1 | **身体**（躯干+腿+手臂，全部图层） | 主外观 | ① 经典 reanim `*.reanim.compiled`<br>② 逐帧 PNG 序列 | 帧**等尺寸**、透明底；朝向与基底僵尸一致 | ★ 必需（除非整只用原版） |
| 2 | **头**（换头时） | 头部外观 + 开火动画 | 同上 | 同上；**图层要齐**（原版头有 7 层，见 §5） | 换头时必需 |
| 3 | **护具**（路障/铁桶/铁门/报纸…） | `Armor` 三件套 | `.tres` 数据 + 帧 | 与头同尺度、同帧序 | 可选（可复用原版） |
| 4 | **弹丸**（要发射才需要） | 子弹外观 | 原版图集 / 自己的帧 | —— | 可选（可复用原版） |
| 5 | **参考图** | 定朝向、定比例、定相对位置 | 单张 PNG/JPG | 侧视全身、纯色底（便于抠） | 建议给（否则比例只能靠口径定，见 `zombie-skin-and-head.md` §10d） |
| 6 | **部件特写**（多体节 / 拼链） | 组装成一条链 | 单张 PNG ×N | ≥ 目标显示尺寸的 2×（要缩放）；说明每张的部位与朝向 | 拼链时必需 |
| 7 | **音效**（可省） | 发射/死亡音 | wav/ogg | —— | 可选 |

> 「等尺寸」是**硬要求**：逐帧序列里任何一帧尺寸不同，图集排版与帧索引都会错位。
> 单帧尺寸**不必**等于官方 `105×134`（可以等比缩放），但**必须全角色统一**。

---

## 2. 官方规格基准（实测，别猜）

以官方 `Chapter1/Normal`（普通僵尸）为基准 —— 出处：`…V0.28\Asset\Anime\Character\Zombie\Chapter1\Normal\`

| 项 | 实测值 | 出处 |
|---|---|---|
| **单帧尺寸** `tileSize` | **`105 × 134`** | `Asset\Anime\Character\Zombie\Chapter1\Normal\Generated\ZombieNormalRasterCompositeData.tres` |
| **图集尺寸** | **`3360 × 1206`**（PNG `3 396 415 B`） | 同上 `atlas` 指向的 `ZombieNormalRasterComposite.png` |
| **每行列数** `columns` | **`32`** | 同上 |
| **合成图原点** `origin` | `(-52, -86)` | 同上 |
| `variantFrameStride` | `131` | 同上 |
| **场景显示偏移** `offset` | `(-40, -80)` | `Asset\Anime\Character\Zombie\Chapter1\Normal\Sprite\Normal\ZombieNormal.tscn` |
| **`trueFrameRate`** | `180.0` | 同上（⚠️ 这是 AdobeAnimate 的时间基，**不是**显示帧率） |
| 显示帧率 | `frameRate = 12` | 本项目 `SuperGatlingPea.tres`（僵尸侧惯例） |
| **clip 帧号布局** | `BodyIdle 0..24` / `HeadIdle 25..49` / `HeadFire 50..86`；帧号**全局单调** | `references/zombie-skin-and-head.md` §1 |
| 发射帧 | `HeadFire.start + 12` | 同上 |
| 官方插槽（含各自 `offset`） | `HeadSlot(followSlotId=8, offset=(25,30))`、`ArmSlot(31,(10,20))`、`ConeSlot(22,(30,30))`、`BucketSlot(23,(34,32))`、`ScreendoorSlot(25,(30,60))`、`GroundSlot(1)` | `Asset\Anime\Character\Zombie\Chapter1\Normal\Sprite\Normal\ZombieNormal.tscn` |

⚠️⚠️ **`HeadSlot` 是「每个角色各自一套」的，不能跨角色抄**：
普通僵尸是 `offset = (25, 30)`；而 **Paper（读报僵尸）** 是 `position = (-14.015516, -40.408867)` +
`rotation = -0.27867758` + `scale = 0.79857695`
（出处 `Asset\Anime\Character\Zombie\Chapter1\Paper\Scene\TowerDefenseZombiePaper.tscn` **:67-71**，实测；
⚠️ 别按名字里的 `TowerDefenseZombiePaper` 去 `Prefab\` 下找 —— 那里没有这个文件）。
两者数值完全不同 ⇒ **换头前先读你那个基底的 `Sprite` 场景**，照它填。

⚠️ `origin`（合成图锚点）与场景 `offset`（显示偏移）**不是一回事**，数值也不同（`(-52,-86)` vs `(-40,-80)`）
⇒ 改一个不等于改另一个（偏右下的现象见 SKILL.md §4b.12）。

---

## 3. 素材来源地图（「允许复用原版」具体指什么）

| 来源 | 路径 | 形态 | 能怎么用 |
|---|---|---|---|
| **经典版 reanim** | 本机 `D:\zzz\extract_1789988101\`：`compiled\new\*.reanim.compiled`（**实测 620 个**）+ `reanim\`（文本态）。⚠️ **V0.28 重置版解包树里一个 `.reanim.compiled` 都没有**（实测，别去那儿找） | 逐帧二进制 + 轨道/变换 | ★ 首选：帧数据现成，直转即可（本项目 `build_official_skin.py` 已验路线） |
| **重置版本体角色** | `…V0.28\Asset\Anime\Character\Zombie\<Chapter*>\<Name>\` | `*.tres`（`AdobeAnimateData`，`animeFile` 指向 `.dat`）+ `Generated\*RasterComposite.png` + `Sprite\`、`Scene\`、`Armor\`、`Packet\` | 复用基底：组件集 / 护具 / 卡库 / 场景都可照抄（`.cs` 除外，包内禁） |
| **官方独立头先例** | `Chapter1\Normal\Sprite\Sunflower\SunFlowerHead.{tscn,tres}` | 一个 `Node2D` + 自己的 `.tres`，逐层 `Animation/LayerVisible/<层名>` | ★ 官方「独立头」就是这么做的 —— 与本技能 Step 5 的三节点同思路 |
| **「植物僵尸」整体先例** | `Chapter1\Normal\Sprite\<Plant>\` + `Scene\<Plant>\` | 头 + 攻击配置成套 | 换头 + 换攻击的完整官方样板（本项目主线僵尸即此类） |
| **多部件 BOSS** | `Challenge\FootballGargantuar\Sprite\Black\*.png` | **部件单图**（`40×38` ~ `232×145`，如 `_outerleg_lower` / `_qiumen` / `_shoulder`） | 拼链（SKILL.md §4b.5）的官方骨料 |

⚠️ **解包树里没有 `.dat`** —— 全树只有一个 `icudt_godot.dat`。官方 `*.tres` 的 `animeFile`
指向的 `<Name>.dat` **需要自己生成**（本项目产物：`SuperGatlingPea.dat` 344 434 B + `.tres` 142 786 B
+ `SuperGatlingPeaAtlas.png` 256×296）。
⇒ 所以「复用原版外观」的现实做法是 **① 直接引用/照抄官方 `Sprite\*.tscn` 与 `Scene\*.tscn`
（最省，不碰 `.dat`）**，或 **② 从经典版 reanim 自己转一套三件套**。
（官方 `Generated\*RasterComposite.png` 大图集直接拿来用**未验证**，要试请先自证。）

---

## 4. 三种素材形态的判定与处理

| 用户给的东西 | 先判什么 | 处理路径 |
|---|---|---|
| `*.reanim.compiled` | 用 `reanim_probe*.py` 读轨道/帧数 | 直转 `.dat` + 图集 + `.tres` + `skin_params.json`（见 `zombie-skin-and-head.md` §2） |
| 逐帧 PNG 序列 | **帧数、单帧尺寸是否全等、命名顺序**（`001..NNN` 或 `frame0..N`） | 排版图集 → 生成 `.dat`/`.tres`；自制皮肤记得运行期 `forceLocalRender` + `forceCpuPoseRender` |
| 一张参考图 | 朝向（侧视？正视？）、底色是否纯色 | 走「单图 → 角色贴图」管线（抠像 + 逐帧），见 `pvz-hybrid-mod-authoring` |
| 部件特写若干张 | **每张的视角与朝向**（正视/俯视/侧视） | 先定旋转角再裁切拼链（SKILL.md §4b.5，三条独立判据） |

---

## 5. 「没有素材」时的降级路线（按省事程度排序）

1. **只改数值/机制** ⇒ 完全不动外观，直接复用一个原版僵尸（`Chapter1\Normal` 最干净）。
2. **要一点点不一样** ⇒ 换头（身体用原版 + 头用另一个原版角色/植物头）——
   官方已有先例（`Asset\Anime\Character\Zombie\Chapter1\Normal\Sprite\Sunflower\SunFlowerHead.tscn`），不用自己画。
3. **要明显不一样但没有素材** ⇒ 要一张参考图，走扣像管线。
4. **手上有部件图** ⇒ 拼链（最费，先确认用户真的要走到这一步）。

---

## 6. 素材验收自检（拿到素材先跑这四条，别等实机）

1. **帧尺寸全等**：逐帧读 PNG IHDR（`struct.unpack('>II', b[16:24])`），去重后必须只剩 1 组尺寸。
2. **透明底**：alpha 通道要有 0 也要有非 0（全不透明的图 = 没抠过底）。
3. **帧数下限**：至少覆盖你要用的 clip —— `BodyIdle 25` 帧、`HeadIdle 25` 帧、`HeadFire 37` 帧
   （按 `zombie-skin-and-head.md` §1 的布局；自定布局则按自定的）。
4. **朝向对基底**：拿基底僵尸对应的那张素材**并排看一眼**再定要不要横翻 ——
   **不要在素材上预翻**（僵尸朝向由引擎的横翻处理，SKILL.md Step 6）。

> 这四条都可以用一段十几行的 Python 一次跑完（本项目对照脚本在 `.cache\` 下，见 SKILL.md §5）。
