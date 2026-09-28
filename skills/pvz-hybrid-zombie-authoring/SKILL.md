---
name: pvz-hybrid-zombie-authoring
description: 为《植物大战僵尸杂交版》(Godot 4 + C#) 从零制作或改造一个「僵尸」Mod 的端到端流程——选基底僵尸、建包（13 个文件）、本体配置（血量/濒死/啃食伤害）、**补数值与属性（生命值/啃食伤害/伤害类型 attackType/移动速度/手册五行属性块）**、**让僵尸真的能往前走（动画 `_ground` 根运动层）与调速**、护具 Armor 三件套、换头「三节点」结构、换外观（经典 reanim 官方素材直转）、给僵尸加发射（自建 Fire 组件 + 托管插件驱动节拍）、植物僵尸共用的攻击判定（没植物不开火、进场才开火）、子弹生成点对齐炮口（FireMarker）、**用 buff 做控场（让全场植物停止发射的攻速归零 buff）**、**多体节 BOSS（N 个独立受击判定共用一条血条 / `physique = 6` 启用内置 Boss 血条 / 多部件素材拼链）**、图鉴去重，以及闸门/负向测试/幂等全套离线验证。当用户要求「做个僵尸 Mod」「把僵尸换成 XX」「给僵尸换头/换贴图/换动画」「让僵尸会开枪/会发射」「僵尸加护具」「植物僵尸的攻击判定」「改僵尸血量/伤害/移动速度/伤害类型」「**僵尸不动/不往前走/不移动/原地滑步**」「让植物停止发射/暂停行动/控场」「**做个多体节 BOSS / 多个部位共享血条 / Boss 血条**」「**给几张部件图自行裁剪组装 / 体节加长 / 消接缝 / 让身体自然弯曲 / 蛇形游走**」「**做这个僵尸需要什么素材 / 我该给你什么图 / 有素材吗 / 直接复用原版素材行不行 / 素材要什么格式和尺寸**」或提到 .pmod / Zombie / 僵尸 / 护具 / Armor / FireMarker / 子弹生成点 / attackType / walkSpeedScale / **_ground 根运动** / AttackSpeedDown / physique / BOSS / BossHealthBar / DamagePoint / 最小转弯半径 / R_min / seg_stretch / OVL / 接缝 / **素材 / 逐帧 PNG / reanim / tileSize / RasterComposite / 部件图** / ModAssembly.dll / **csproj `<AssemblyName>` / 程序集名唯一 / 安卓加载失败 / runtimeAssembly 能不能改** / 图鉴僵尸页 时使用。
agent_created: true
---

# 制作 PvZ 杂交版「僵尸」Mod —— 端到端流程

> 姊妹技能：`pvz-hybrid-mod-authoring`（全量/通用，含包格式与编辑器）、`pvz-hybrid-plant-authoring`（植物端到端）。
> 本技能只讲**僵尸**这条路，以及它与植物的**每一个差异**。
>
> ★★★ **开工第一件事是「问素材」，不是「写代码」**：先确认用户手上有哪些素材（**部位 / 格式 / 尺寸**），
> 并**主动告知可以复用原杂交版已有的素材**（`Chapter1\*` 里的基底僵尸、护具、独立头、卡库都能直接照抄）。
> 问卷模板 + 官方规格基准（单帧 / 图集 / 帧号布局）+ 素材来源地图 + 没有素材时的降级路线
> ⇒ **`references/zombie-asset-requirements.md`**（Step 0 强制先读）。

## 0. 权威来源与边界（务必先看）

| 用途 | 路径 |
|---|---|
| **源码真相（唯一有 `.cs` 的那棵树）** | `D:\zzz\pvzHE\解包\植物大战僵尸杂交版V0.28\` |
| 基底僵尸（读报/普通/路障…） | `Asset\Anime\Character\Zombie\Chapter1\<Name>\` |
| 内置僵尸基场景 | `Prefab\TowerDefense\Character\TowerDefenseZombie.tscn` |
| 内置僵尸组件集 | `Prefab\TowerDefense\Character\ComponentSets\TowerDefenseZombieComponentSet.tres` |
| 护具体系 | `Registry\Armor\`、`Script\Component\TowerDefense\Character\Armor\` |
| 发射体系 | `Script\Component\TowerDefense\Character\FireComponent\FireComponent.cs` |
| 图鉴 | `Prefab\GUI\DialogBox\Almanac\Almanac.cs` |
| 用户 Mod 目录 | `%APPDATA%\Godot\app_userdata\植物大战僵尸杂交版\Mods\` |
| 实况日志 | 同上目录 `PVZHE_Logs\` |

**边界**：游戏内「PVZ Mod 编辑器」（F3）**AI 无法驱动**；但 `.pmod` 只是「zip + 根 `mod.json`」，
纯脚本可写。**动手前先确认用户已有可复制的生成器**（本仓 `build_zombie_super_gatling_paper.py` 为样板，
160KB，含全部自检与金标）。

---

## 1. 先做四个决定（决定后面 80% 的工作量）

### 决定 A：复用什么基底僵尸？

| 基底 | 特点 | 什么时候选 |
|---|---|---|
| `Chapter1\Paper\`（读报僵尸） | **自带护具 + 状态机 + 暴走**（`ArmorHitpointsEmpty("Paper")` → `ToGasp` → 3 倍速） | 要护具 / 要「血量掉光后暴走」的机制 |
| `Chapter1\Normal\`（普通僵尸） | 最干净：只有身体 + 头 | 只换外观、不加机制 |
| 其它（路障/铁桶/舞王…） | 各自带护具或特殊动画 | 想白拿它的护具/动画 |

⚠️ **基底僵尸的「脚本」不能复制进包**（包内禁 `.cs`）—— 见 Step 4 的剥离手法。

### 决定 B：要僵尸「发射」吗？

**僵尸默认没有 `FireComponent`**（`TowerDefenseZombieComponentSet.tres` 里没有 Fire，只有 Armor/Attack 等）。
「啃食」是 `AttackComponent` 默认的 `attackType = "Eat"`，**不是发射**。
⇒ 只要想做「会开枪的僵尸」，就必须：

1. 数据侧：新建 `…ComponentSet.tres`，**基于内置僵尸组件集再加 1 个 FireComponent**；
2. 代码侧：写托管插件（僵尸没有植物那种「内置发射动画链」，节拍要自己驱动）。

### 决定 C：外观走哪条路？

* ★ **先定素材**：这条路选哪条，取决于**用户手上有什么**（逐帧 / reanim / 参考图 / 部件图 / 什么都没有）
  ⇒ 先读 `references/zombie-asset-requirements.md`（含「能复用原版就别要新素材」的省事顺序）。
* **官方素材直转**（首选）：经典未重置版 reanim → 重置版 `.dat`/`.tres`/图集（见 `references/zombie-skin-and-head.md`）。
* **只换头**：走「三节点换头」（`HeadShadow` + `HeadHolder` + `Head`）—— 见 Step 5 与 references。
* **自制逐帧**：能走，但要处理「自制 `.dat` 皮肤静止」⇒ 运行期 `forceLocalRender` + `forceCpuPoseRender`。

### 决定 D：要不要图鉴 / 卡库？

僵尸的卡库是 `GeneralZombie`，**补一处 = 选卡界面 / 关卡编辑器 / 图鉴僵尸页同时生效**
（`Almanac.cs:220` 取的是**同一个实例**，不像植物页那样深拷贝）。
但副作用是**图鉴僵尸页会多出第二条** ⇒ 见 Step 9。

---

## 2. 关键路径（按需查证）

```
<包>/
  mod.json
  Resources/Cards/<Key>.tres                                  ① 卡片（类型 ZOMBIE）
  Resources/Characters/Zombies/<Key>/Config/TowerDefense<Key>.tres       ② 本体配置
  Resources/Characters/Zombies/<Key>/Packet/<Key>.tres                 ③ 卡包条目
  Resources/Characters/Zombies/<Key>/Scene/<Key>.tscn                  ④ 场景
  Resources/Characters/Zombies/<Key>/Scene/<Key>ComponentSet.tres      ⑤ 组件集
  Resources/Characters/Zombies/<Key>/Scene/<Key>FireComponentDefinition.tres  ⑥ 发射定义（要发射才需要）
  Resources/Characters/Zombies/<Key>/Scene/<Key>AttackComponentDefinition.tres  ⑥b 攻击定义（要改「伤害类型」才需要）
  Resources/Characters/Zombies/<Key>/Sprite/<Key>.tscn                 ⑦ 精灵场景
  Resources/Characters/Zombies/<Key>/Armor/Zombie<Key>ArmorData.tres   ⑧ 护具数据
  Resources/Characters/Zombies/<Key>/Armor/Config/Zombie<Key>Armor<X>.tres   ⑨ 护具槽配置
  Resources/Animations/<Skin>.{dat,tres,Atlas.png}                     ⑩ 外观三件套（换贴图时）
  Runtime/ModAssembly.dll                                             ⑪ 托管插件（**这个路径不许改**）
  <中文名>.pvzmodeproject                                              ⑫ 工程标记（**不进包**）
```

**路径硬约束**：`Resources/Characters/Zombies/<Key>/{Scene,Sprite}/<Key>.tscn` —— 恰 **6 段**；
类别目录必须是 `Zombies`；文件名必须等于 `<Key>`。违反 = `ModLoader.InferRuntimeEntry` 推不出类别。

**★★★ 程序集名的硬约束（安卓专属，`Runtime/ModAssembly.dll` 那条不算）**：
包内 `Runtime/ModAssembly.dll` 的**路径**一个字都不能改，但 `.csproj` 里的 `<AssemblyName>`
**必须改成自己的 `<Key>`**（`<AssemblyName>SuperGatlingPaper</AssemblyName>`）。
安卓全 Mod 共用一个非可回收加载上下文，程序集名撞车会直接
`Android Mod assemblies share one non-collectible context, so main assembly names must be unique`
⇒ 第二个 Mod 加载失败。**详见 `references/zombie-package-and-gates.md` §5.1**。

---

## 3. 标准流程（12 步）

### Step 0 — ★★★ 先问素材，再把需求问全

**第一件事永远是「问素材」**（素材形态决定路线与工作量；而且用户往往不知道原版素材可以直接复用）。
问什么、怎么问、每类素材的规格基准与降级方案 ⇒ **`references/zombie-asset-requirements.md`**。

> **准则：能复用原版就别要新素材。** 顺序是
> ① 复用原版外观（`Chapter1\Normal` 最干净）→ ② 换头（身体用原版 + 头换一个，官方已有
> `Asset\Anime\Character\Zombie\Chapter1\Normal\Sprite\Sunflower\SunFlowerHead.tscn` 这种先例）→ ③ 要一张参考图走抠像 → ④ 部件图拼链（最费）。
> **别一上来就要求用户交「图集」**：图集排版是脚本的事，用户只需给**逐帧 PNG 序列 / reanim / 参考图**之一。

素材确认后，再问全需求：① 基底僵尸？② 要不要护具、护具多少血？③ 要不要发射、射速/弹数/散射？
④ 头部贴图换不换、用哪个角色的？⑤ 要不要「没植物不开火、进场才开火」？
⑥ 血量/啃食伤害/卡片价格冷却？⑦ 大招之类的特殊机制？
**缺一项后面就得返工**（尤其④⑤，它们决定要不要写插件）。

### Step 1 — 复制生成器，别从零写

```bash
cp build_zombie_super_gatling_paper.py build_zombie_<新Key>.py
```

生成器要包含：`self_check()`（含 `*_GOLDEN` 金标）、场景/配置渲染函数、`Build` 落盘、
`dotnet build` 调用（或独立 `build_runtime.py`）、打包 `.pmod`、安装到 `Mods/`、镜像到 `%APPDATA%`。
**改常量永远只改一个文件**。

### Step 2 — 定 Key 与中文名

* `<Key>`：ASCII，如 `ZombieSuperGatlingPaper`（**进图鉴的键、也是插件识别角色的锚**）。
* 中文名：`translate` 里**直接写中文**（不需要 `translations.csv`；写了反而要多维护一份）。
* 每个 `.tres` 的 `script_class` / `metadata/_custom_type_script` 要与内置基底对齐。

### Step 3 — 包内 13 个文件

逐个从基底改。最容易漏的是 **③ Packet**（没它卡在包里但选不到）与 **⑨ 护具槽**。

### Step 4 — ★★ 场景必须**显式声明** `ComponentSet`（在 `script` **之前**）

基场景 `TowerDefenseZombie.tscn:10` 自带 `ComponentSet = ExtResource(...)`
⇒ **子场景不写就继承它**。而它指向的 `TowerDefenseZombieComponentSet.tres` 里**没有 FireComponent**
⇒ 「僵尸不会开枪」而且是**零日志**（组件根本没被创建）。

```gdscript
[node name="Zombie<Key>" instance=ExtResource("1")]
ComponentSet = ExtResource("<你的组件集 id>")     # ← 必须在 script 之前
script = ExtResource("<内置僵尸脚本 id>")
```

**继承内置脚本时**：包内**禁止** `.cs` / `.scn` / `.res` ⇒ 只能引用**内置**的 `res://` 路径，
且要**剥掉**基底场景里指向包内的 `ExtResource` 行（否则 ModLoader 会拒整包）。

### Step 5 — 精灵场景：换头走「三节点」

**为什么不能直接换子精灵的贴图**（我踩过）：`CollectOwnedChildBindings`（`:5385`）只看节点类型，
`_Draw`（`:9534`）首行就 `return` ⇒ **子精灵的切片一定被父批次代画、采样父那一张图集**
⇒ 跨 `.tres` / 自制皮肤的子精灵 = **别的角色碎片拼贴**。

**修法（唯一可行）**：身体下留三个节点：

| 节点 | 类型 | 作用 |
|---|---|---|
| `HeadShadow` | 与身体同类精灵 | `visible = false` + **全层 false** ⇒ 零切片；只为吃 `UpdateChild()` 的每帧定位 |
| `HeadHolder` | 普通 `Node2D`（identity） | 打断「父代画」的容器 |
| `Head` | 独立渲染的精灵 | 给人看的那个；写 `z_index = 1` 压住身体 |

> **⚠️ 反过来「要让子精灵排在父后面」怎么办？**（贴图光环、脚底特效等）
> 这时 `z_index = -1` **完全无效** —— 被父代画的子精灵，它的 `ZIndex` 排序位取的是**父精灵**的
> `EffectiveZIndex`。**唯一**有效手段是显式写 `insertLayerId = <id>`（按**父精灵**的
> `layerDictionary` 解释；一般 `id = 0` 是内建隐藏背景板，可见层从 `1` 起 ⇒ 取 `0` 就画在父**后面**）。
> 详见 `references/zombie-skin-and-head.md` **§9**。

两头（`HeadShadow` 与 `Head`）的 `scale` / `offset` / `offsetRotate` **逐字相同**，
位姿由插件每帧从影子抄给可见头（见 Step 8）。
**原版头部的 7 个图层必须在身体上显式关掉**（`Animation/LayerVisible/<原头层> = false`），
否则原头会从头盔底下透出来。

> **⚠️⚠️ 写 `Animation/LayerVisible/…` 时，引号必须包住「整条」属性名**（2026-09-25 用一天换来的教训）：
> 层名含**空格或非 ASCII**（`图层_1` / `skin2_2 ` / `cloak1 复制` …）时：
> ```
> 正确： "Animation/LayerVisible/图层_1" = false     ← 官方写法（全库 6941 个文件零例外）
> 错误： Animation/LayerVisible/"图层_1" = false     ← 只包末段 ⇒ 这一行**完全无效**
> ```
> 坏写法**不报错、不留日志**：`_Set()` 里 `layerDictionary.ContainsKey()` 拿到带引号的名字 ⇒ 为假 ⇒
> 直接 `return true` ⇒ 该层保持**初值 `true`**（`:9049-9069` 没写进 `_layerVisible` 的下标一律按可见算）
> ⇒ **永远可见**。后果实测：`HeadShadow` 的「全层 false」白写 6 层（含**另一张脸 `skin2_2`**、皇冠、
> 花瓣、火圈、光环、披风）⇒ 身体上**多长出一个完整的头**，且光环/披风怎么关都关不掉。
> ⇒ 生成器统一走 `_prop(key)`（判据作用在**整条 key** 上）+ 常驻自检 `_bad_prop_keys()`；
> 全库对照扫描 `.cache/_sq_quotefix_scan.py`。**ASCII/纯数字层名裸写是对的**（`Animation/LayerVisible/1 = true`）。

### Step 6 — 外观三件套

见 `references/zombie-skin-and-head.md`（含经典 reanim 直转、`.dat` 二进制规格、头对位反解）。
**关键：僵尸朝左**（`scale = (-1, 1)` 横翻）⇒ 所有屏幕坐标都要过横翻矩阵，别直接加。

### Step 7 — 发射：数据侧只做两件事

1. **组件集**：基于内置僵尸组件集 + 加 1 个 FireComponent（`InstanceId` 用固定的接线键）。
2. **发射定义** `.tres`：
   * `firePosMarkerPaths = [NodePath("…/HeadSlot/FireMarker")]` —— 指向 Marker2D（**子弹生成点**，见 references）；
   * `fireProjectileList` 只留 **1 条** config（`Fire()` 一次遍历全部弹 ⇒ 逐颗发射必须在插件里循环）；
   * `speed` 为**负**（僵尸朝左，负值才是向前）；
   * 僵尸的场景里要有真实 `Marker2D` 节点（`firePosMarkerPaths` 必须是真 Marker2D）。

⚠️ **数据侧做不到的事**（只能插件）：概率 / 延时改模式 / 真随机 / 改投掷单位 / 逐颗发射 / 特殊大招。

### Step 8 — 托管插件（僵尸的真正工作量在这里）

入口类放在**无命名空间**下（`runtimeEntryType` 是类名），实现 `IXWModRuntimeEntry`，
三个回调（`Initialize` / `OnAllModsLoaded` / `Shutdown`）**一律不许抛**（抛了 = 无条件整包回滚）。

插件的几件典型职责（照抄本仓 `SuperGatlingPaperRuntimeEntry.cs`）：

| 职责 | 要点 |
|---|---|
| 自建节拍 | 挂 `SceneTree.process_frame` 自己推进毫秒时间轴（**不要**去骗内置状态机） |
| 射击判定 | 必须用 `FireComponent.CanFireCheckOnceByData()`（**不是** `CanFire` —— 后者多一道 `timer > 0` 会把节奏拖一轮）；「没植物不开火、进场才开火」都在这里 |
| 开火动画 | 头独立渲染后，头 clip 完全归插件管：`MarkHeadFire()` 切 `HeadFire` 并**续**窗口；`SetHeadClip()` 只在 clip 真变了才 `SetClip()`（否则每颗豌豆都把动画打回第 0 帧） |
| 换头位姿 | 每帧把影子 `Position`/`Rotation` 抄给可见头（10 帧一档的扫描会抖） |
| **子弹生成点** | 每帧 `marker.GlobalPosition = head.GlobalTransform * HeadMuzzleLocal`（见 references/zombie-fire-and-marker.md） |
| 卡库入库 | 把角色补进 `GeneralZombie` 的派生库（按 `Include` 闭包**运行期**算，别写死） |
| 图鉴去重 | 反射去 `_zombieLogicalConfigs` 的重复项 + 公开的 `QueueZombieVirtualRefresh()` |

**编译**：`python runtime_src_zombie_super_gatling/build_runtime.py --check`（两次编译比 sha256）。
⚠️ **生成器不会自动重编 DLL** —— 改了 `.cs` 必须单独跑一次构建，否则打进包的还是旧 DLL（踩过：DLL 停在两天前）。

**★★★ 两条「名字」的硬约束（照抄旧工程最容易踩）**：

| | 写在哪 | 值 | 能改吗 |
|---|---|---|---|
| 包内**物理路径** | `mod.json` → `runtimeAssembly` | **恰好** `"Runtime/ModAssembly.dll"` | ❌ 硬校验，改 = 整包被拒 |
| **程序集身份** | `.csproj` → `<AssemblyName>` | **本 Mod 的 `<Key>`** | ✅ **必须改**（安卓要求主程序集名唯一） |

```xml
<PropertyGroup>
  <AssemblyName>SuperGatlingPaper</AssemblyName>   <!-- ❌ 不要写 ModAssembly -->
</PropertyGroup>
```
```python
# build_runtime.py 取产物时也要跟着改（装机名仍叫 ModAssembly.dll）
ASSEMBLY_NAME = "SuperGatlingPaper"
src = os.path.join(out_dir, ASSEMBLY_NAME + ".dll")
```
**根因**：安卓走 `LoadAndroidAssembly`（`XWModAssemblyLoader.cs:191-193`）→ 全 Mod 共享一个非可回收上下文，
按简单程序集名（`OrdinalIgnoreCase`）记账，撞上即
`throw BuildAndroidAssemblyConflict`（`:314`，文案 `:338-341`）；PC 走 `ModLoadContext`（`:195-197`，
**每 Mod 一个独立 ALC**）所以从不暴露。`runtimeEntryType` 按 `Type.FullName` 匹配
（`XWModCharacterCompanionRuntime.cs:105`）**与程序集名无关** ⇒ 改名只影响构建脚本取产物名。
完整依据、三条自检 → `references/zombie-package-and-gates.md` §5.1。

### Step 9 — 让它「能被选到」+ 图鉴

* 补进**僵尸根卡库** `GeneralZombie` 的 `Zombie` 分类，以及 `Include` 闭包算出的派生库
  （实测 `['GeneralZombie','TotalZombie','Total']`）——只补根库的话，`debugPacketOpenAll` 切到 `Total` 又看不到。
* **图鉴僵尸页会多一条**（`Almanac.cs:411-435` 两条来源都不去重）⇒ 运行期反射去重（Step 8 表末行）。

### Step 10 — 写/跑闸门（**不能省**）

见 §5 与 `references/zombie-package-and-gates.md`。**离线能抓的错绝不留给实机**。

### Step 11 — 装机

生成器负责：写 `dist/<中文名>.pmod` → 复制到 `Mods/` → 解到 `Mods/<中文名>/`（**72 个标准目录**）→
登记 `enabled_mods.json`（**保留别人的条目**）→ 登记 `mod_editor_recent_projects.cfg`。
**改完包重启游戏即可**，别手删 `ModsCache`。

### Step 12 — 实机验收

离线验不了的（并且**必须如实告诉用户**）：
「头真的画对了 / 动画在动 / 子弹真的从炮口出来 / 没被遮挡 / 图鉴只有一条」。
先看 `%APPDATA%\Godot\app_userdata\植物大战僵尸杂交版\PVZHE_Logs\` 的加载日志。

---

## 4. 坑 Top 10（僵尸专属，每条都能白忙半天）

1. **★★ 场景没显式写 `ComponentSet` ⇒ 发射组件根本不创建，且零日志**（Step 4）。
2. **★★ 子精灵必被父代画** ⇒ 换头必须「三节点」（Step 5），直接换贴图只会得到别的角色碎片。
   **同一条规则的第二个面**：既然被父代画，子精灵自己的 **`z_index` 也失效** ⇒ 想让它排在父**后面**
   只能写 `insertLayerId = <id>`（`-1` = 默认 ⇒ 回落顶层 = 画在最前）。
   详见 references/zombie-skin-and-head.md §9。
3. **★★ 子弹生成点 = `firePosMarkerPaths` 指向的 `Marker2D.GlobalPosition`**，
   不是角色原点、也不是炮口；而 `HeadSlot` **是原版护具用的静态插槽、不跟头部美术**
   （「挂在 HeadSlot 下就跟着头」是错的）⇒ 见 references/zombie-fire-and-marker.md。
4. **★ 头在摆 ⇒ 静态值只能对上参考帧**：只要头跟 `anim_head1` 逐帧摆，炮口每帧都在动
   （Idle 段就跨 20px）⇒ 必须插件每帧覆写。
5. **★ `CanFire` vs `CanFireCheckOnceByData`**：自建节拍必须用后者。
6. **★ 僵尸朝左**（`scale=(-1,1)`）⇒ 「屏幕向右上微移」**不能直接加到 `offset` 上**（横向会反向）；
   必须加在**锚点**上再反解（本仓 `HEAD_PLACE_SHIFT`）。
7. **★ 改完 `.cs` 生成器不会重编 DLL** ⇒ 包里的还是旧 DLL（对比 mtime 立刻能看出来）。
8. **★ 图鉴僵尸页两条** ⇒ 运行期去重。
9. **★★ Godot 对 `.tscn` 的坏写法「静默容忍」**——两类都已实证，都不报错、零日志：
   * **节点头收尾必须是 `]`**：写成 `>` ⇒ **静默吞行**（节点不建）；断言别写成不带闭合括号的前缀匹配
     （**断言与实现同错 = 假绿**）。
   * **属性名引号要包「整条」**：`Animation/LayerVisible/"图层_1" = false`（只包末段）⇒ 该行**完全无效**，
     层保持默认 `true`（详见 Step 5 的警示框）⇒ 会凭空多出一个头。
   ⇒ 规程：**只认官方同源文件的写法**（拿同一份 `.tres` 的官方 `.tscn` 逐字对照），
     再配「复刻负向」（把实现改回坏写法，断言必须打红）。
10. **★ 「断言与实现同错 = 假绿」**：① 恒真比较（拿常量比由同一常量渲染的文本）⇒ 加 `*_GOLDEN` 金标；
    ② 门控共用 ⇒ 硬需求写成无开关不变量；③ 前缀匹配漏字节；④ 目标行在当前口径下不生成 ⇒ **打桩注入**。
    **写完/改完断言必须逐条篡改常量确认报警。**
11. **★★ 尺寸 / 位置类需求（「头调大调小一点」「头往上挪一点」）都别目测猜** —— 目测必被下一张图推翻。
    **尺寸**（用户给目标截图）走三步：**① 泛洪分割**量参考图轮廓（从四边泛洪，黑框/草坪判据）→
    **② 离线合成渲染**扫候选倍率（共享图集 + `Manifest` + `PIL.AFFINE` 逆矩阵，量真实像素 bbox）→
    **③ 同高度归一并排**目视复核。至少两个独立判据，取落在参考图两侧最近的一档。
    ⚠️ 归一化要**剔掉尺寸恒定的装饰**（固定像素的光环会稀释倍率差异）。
    ⚠️ 改完 `head_scale` **必须重解 `head_offset`**（`A = rot_scale(θ, −s, +s)` 含节点 scale）。
    **位置**（「跟那张图一样高」）拿**两张真实图直接比**，**别经过离线渲染**（帧/姿势未必一致），四个坑：
    ① 画幅不同 ⇒ 绝对 y 不可比，先找「与头无关」的身体标尺；
    ② 标尺**不能选会被遮挡的量**（裤子常被脚底光环截断 ⇒ 改用**头宽**，且只能比**宽**）；
    ③ 头块取「**最靠上的橙色大块**」（脚底光环常比头大好几倍）；
    ④ 头的锚点**必须"刚性"** —— 头 bbox 含**火焰花瓣**且火焰**逐帧变**，
       顶/底/中心都会被帧差污染（实测「头块中心」少算 **20%**）；只用**脸 / 王冠**这种刚性子块。
    ⇒ 位移**加在锚点上**反解，别直接加到 `offset` 上（`A` 含横翻 + 交叉项）。
    ★★ **「改为现在的 N 倍」= 相对放大**（在上次定标结果上再乘 N，不是「N 倍于原素材」），三条口径：
    ⑴ `head_scale = 旧值 × N`；⑵ 解 `offset` 时**沿用当前 `shift`**（否则上移量会被一起解掉、头掉回原位）；
    ⑶ **旧 `offset` 不许复用**，必须重解。✅「原地放大」的判据 = **头块落点中心逐字不变**。
    ⚠️ 别用启发式取块量放大前后的尺寸（火焰花瓣逐帧变 ⇒ 两倍率下不是同一块，实测给出 `1.19` 假比值）；
    尺寸比报**解析值**。⭐ 对照图最干净画法：同 `box`、`k`、`shift` 渲两张，画**解析锚点十字**；
    且离线渲染落盘是 **RGBA** ⇒ 「旧轮廓叠新图」**直接取 alpha 通道**（`MinFilter(3)` 腐蚀相减），零启发式。
    ★★ **没给参考图**时（「再往右上方移一点」）：铁律 30 仍**禁止目测猜** ⇒ **先问量级**，
    问不到就**按口径定**（查本项目历史同义先例、取保守值；实测先例 18px / 20.16px ⇒ 取 **12px**），
    并写明「**这是默认值、一句话可改**」；**多角色同一句话 ⇒ 同一增量**；位移用**累计 `shift`** 重解。
    ⚠️ **隐藏基准坑**：离线渲染器 `--mult` 实为 `S = 0.45 × mult`，`0.45` 是**女王包历史口径** ⇒
    给**别的角色**渲图必须用 **`--head-scale`**（直接给绝对 `S`），否则头会被渲小 2.2 倍。
    详见 references/zombie-skin-and-head.md §10 / §10b / §10c / §10d。
12. **★★ 「移动速度 / 伤害类型」不在 config 里**，别在 `TowerDefenseZombieConfig` 里找：
    **伤害类型** = `attackType`，只在 `Scene/<Key>AttackComponentDefinition.tres` 上
    （取值 `Default/Eat/Smash/Chomp`，「啃食」= `Eat`）；**移动速度** = 角色 `.tscn` 节点属性
    `walkSpeedScale`（内置口径：普通僵尸 `1.0` = 手册「慢」，橄榄球僵尸 `2.0` = 手册「快」）。
    攻击定义要**覆盖**内置，必须逐字对齐 `InstanceId = "character.attack.0"` /
    `ComponentTypeId = "AttackComponent"` / `WireIndex = 0`，否则 `CanReplaceInheritedDefinition`
    只报错**不替换**（还有 `useParentHitBox` / `checkLine` / `StateMachineDefinition` 也不能漏）。
    另外 `hitpointsNearDeath` **是濒死掉血速率、不是阈值**，别按血量比例缩放。
    手册五行属性块（`血量/伤害/佩戴/类型/移速`）格式见 references/zombie-stats-and-cc.md §A。
13. **★★ `.tres` 字符串不能有裸换行** ⇒ 多行手册文案必须走 `\n` 转义。
    写成真换行时**包能打出来、闸门全绿、装机也成功，只有进游戏才解析不开**
    ⇒ 加机械闸门（`key = "…"` 行必须以 `"` 收尾且引号数为偶数）。
14. **★★ 想让植物「停止发射」，`timeScaleInit = 0` 不够** ——
    `FireComponent:1687`（IZM 分支）与 `CannonComponent:1344`（`flag ? 1.0 : timeScale`）
    **绕开了 timeScale**，植物照打。唯一全路径有效的杠杆是
    `buff.GetAttackSpeedMultiplier()` ⇒ 挂
    `TowerDefenseCharacterBuffAttackSpeedDown { timeScaleValue = 0, time = N }`。
    三条纪律：**只摘自己挂的**（`BuffGet` 已存在且非自己挂 ⇒ 跳过，埃德加二世的火球是真实碰撞场景）、
    **每次给新实例**（`EnterBuff` 不克隆）、**`Shutdown` 兜底摘**
    （否则关掉 Mod 植物永远开不了火）。详见 references/zombie-stats-and-cc.md §B。
15. **★★★ 自建皮肤的僵尸一步都走不了 —— 皮肤里缺 `_ground` 层。**
    全仓**只有** `GroundMoveComponent` 会平移角色（`grep TranslateForPhysicsFrame` 只命中它），
    而它按**名字**从 `layerDictionary` 解析 `"_ground"`，读该层逐帧位姿差当位移：
    `vector2 = 上帧 − 本帧` → `TranslateParent(vector2 * _moveScale)`。
    没有这一层 ⇒ `_usingGroundLayerSource = false` ⇒ `_moveDelta == 0` ⇒ **原地不动**。
    配方：`.dat` 多写一层（原生就是「每层 × 每帧 × 若干元素」，不用改格式），
    元素 **`alpha = 0`**（只做根运动、不可见）、变换**非 Identity**（第 k 帧 origin 取 `(k+1)·step`）；
    **一个剪辑只放一条完整锯齿，回跳必须落在剪辑边界上**
    （边界会置 `sprite.blend`，`BatchUpdateValidated` 开头 `if (pause || blend) { ResetGroundTracking(); return; }`
    才吃得掉）；`Idle*` / `Eat` / `Death*` **绝对不要加**，否则站/啃/倒都会自己滑。
    ⚠️ 同时纠正一个常见误解：**`walkSpeedScale` 不影响移速**（基类从不读它），
    真正的移速 = 该层位移速率（内置普通僵尸 50px/47 帧 = 12.77 px/s）。
    ⚠️ 每帧强制切剪辑必须用 `SetClip` 而非 `SetAnimation` —— 后者置 `blend`，位移会被整段丢掉。
    详见 references/zombie-stats-and-cc.md §B0。

---

## 4b. ★★ 多体节 BOSS 僵尸（N 个独立受击判定共用一条血条）

需求形态：「一个 BOSS 由 1 头 + 9 体节 + 1 尾共 11 个**独立受击判定**组成，全部**共享同一条血条**」。
2026-09-27 勘察《星空机械蜿蜒》时把三条链路全部落到源码，照这套做即可，**不用自制 UI**。

### 4b.1 三条关键事实（先看这个，能省一整天）

| 想知道 | 结论 | 源码 |
|---|---|---|
| 怎么让内置 **Boss 血条**接管 | **`config.physique = 6`**（`ZOMBIE_PHYSIQUE.BOSS`）即自动生效，条数/比例/淡出/最多同时 3 条全白送 | `Prefab/GUI/BossHealthBar/TowerDefenseBossHealthBarManager.cs:597 IsBossCharacter()`；比例 `:615-660` `hitpoints / hitpointsSave`（护具另算 `shieldRatio`）。**图标按 `config.name` 匹配**（`ResolveStyle :664`，未知名字 ⇒ `icon = null`，只剩色条） |
| 一个角色能有几个 hitbox | **只有一个**（`CharacterHitBoxDefinition { Size, LocalTransform }`）⇒ 「N 个独立受击判定」**只能靠 N 个角色实例**，不是给一个角色挂 N 个盒子 | `Resource/TowerDefense/Collision/CharacterHitBoxDefinition.cs`；命中判定 `Core/TowerDefenseManager/SubSystem/TargetSystem.cs:643 / :806`（`checkCharacter.IsHitBoxEnabled` + `WorldHitRect`） |
| 怎么共享血池 | 每帧轮询归一化：`pool = TOTAL − Σ(baseᵢ − hpᵢ)`，回写全部 N 段的 `hitpoints` + `hitpointsSave` | `Resource/TowerDefense/Character/Instance/TowerDefenseCharacterInstance.cs:45 public double hitpoints`（**可写**）、`:42 hitpointsSave`、`:252 event Action hitpointsEmpty`。⚠️ **没有「受伤」事件** ⇒ 只能 `_Process` 轮询（不要去找 hitpointsChanged，不存在） |

### 4b.2 照抄的配方

1. **11 个同源实例**：同一份 `Config/Scene/Sprite`，插件在运行期额外生成 10 个，用节点名/自定义字段打段索引。
2. **只有 0 号（头）`physique = 6`**，其余取 **`HUGE`（同伽刚特尔），绝不能取 `SMALL/NORMAL`** ——
   `TargetSystem.cs:462` 会按 `zombiePhysique >= HUGE` 过滤，取小了某些植物直接不索敌。
   ★★ **枚举真值（`Resource/TowerDefense/TowerDefenseEnum.cs:114`）**：
   `NOONE=0, SMALL=1, NORMAL=2, MID=3, **HUGE=4**, CAR=5, BOSS=6` ⇒
   **体节要写 `4`，不要写 `5`（`5` 是 `CAR`＝车）**。这条极易写错，
   本项目计划书原稿就写成了 5，靠负向用例才逼出来。
3. **11 段全部摘掉 `GroundMove`**（组件集 `RemovedInstanceIds = ["character.ground_move"]`），
   位移交给插件 `TowerDefenseCharacter.SetLogicalGlobalPosition(Vector2)` 驱动 ——
   绕开铁律 15「自建皮肤缺 `_ground` 层 ⇒ 原地不动」那整条坑。
   ⚠️ **Scene 上必须显式写 `ComponentSet = ExtResource("compset")`** —— 基场景
   `TowerDefenseZombie.tscn` 自带一份（那份里没有你要摘的东西），不覆盖 ⇒ 摘除**完全不生效且零日志**。
   另：非头段再摘 `character.show_health`（否则场上晃 N 条小血条，先例 `GraveStone` / `MushroomMinis`）
   + `character.attack.0`（否则 N 倍啃食伤害）。
4. **共享血池每帧归一化**。本项目已跑通的闭式（`hitpointsSave` 每帧一起写，天然免疫难度缩放）：
   ```
   frameDamage = Σ_i max(0, expected[i] − hp[i])     # expected = 上一次写回值
   totalDamage += frameDamage                        # clamp 到 TOTAL
   pool    = TOTAL − totalDamage
   每段写：hitpoints = pool / N ；hitpointsSave = TOTAL / N
   ```
   ⇒ 头段 Boss 血条比例 = `(pool/N) / (TOTAL/N)` = **`pool / TOTAL`** ✓；
   `pool = 0` ⇒ N 段 **同帧** `hitpoints` 归零 ⇒ `hitpointsEmpty` 同帧触发。
   （写 `hitpoints = pool` / `hitpointsSave = TOTAL` 也等价 —— 只要**比例**是 `pool/TOTAL`。
     但 ⚠️ 别只写 `hitpointsSave = TOTAL` 而忘了同时改 `hitpoints`，那样比例恒为 1。）
   数据侧各段 `hitpoints` 写 `TOTAL/N`，且**不写** `hitpointsNearDeath`（默认 0）⇒
   `hitpointsSave == hitpoints == TOTAL/N`（`TowerDefenseCharacterInstance.cs:318-320`）。
   ⚠️ `hitpointScale`（难度缩放）的 setter 会**同时**缩放 `hitpointsSave` 与 `hitpoints`（`:206-230`）
   ⇒ 每帧写回即覆盖，不用手动反除。
   ⚠️ **没有「受伤」事件** ⇒ 只能每帧轮询（`hitpointsChanged` 不存在）。
5. **隐藏非头段的小血条**（`ShowHealthComponent`），否则 11 段各一条小血条。
6. **`unUseBuffFlags` 别直接抄 BOSS 的 `1020`** —— 那是「免疫一大堆 buff」的口径，
   会连带影响你自己的判定；逐项核过再写。
7. **段间跟随 = 链式约束（每段朝前一段、保持 `link[i]` 间距）+ 关节**绝对角度限幅**
   ⇒ 天然蛇形，且**帧率无关**（限幅要写成「绝对朝向 clamp」，不是「每帧加多少」）。
   ★ 把算法抽成**单一真源模块**（本项目 `_src/_ss_spine.py`：`spine_pose` / `render_pose` / `poly_path` / `r_min`），
   mockup 脚本与游戏内插件**共用同一份**，否则离线渲得好看、进游戏对不上。
   ⚠️ 用**拖尾**（本体沿头部走过的轨迹铺开）而不是硬贴曲线，才是蛇的真实运动学。
   **受击部位跟随是结构自带的**（每段是独立角色，hitbox 随节点走），不用额外写。

### 4b.3 官方 BOSS 样板（拿来当骨料，别从零写）

| 路径 | 可抄什么 |
|---|---|
| `Asset/Anime/Character/Zombie/Boss/Boss/`（僵王） | `physique = 6` 的 config；`DamagePoint/` 三阶段外观；`Constraint` 副角色（`Driver`）+ 载具（`RVNode`）；`BallSpawnMarker` / `SpawnMarker` 生成点；组件集里一串 `attackType = "Smash"` 的 `character.attack.N` |
| `Asset/Anime/Character/Zombie/Boss/EdgarII/`（埃德加二世） | 同型 BOSS 的 `Config/Packet/Scene` 三件套最干净写法（`hitpoints = 500000`、`weight = 6000`、`wavePointCost = 2000`、`cost = 1000`、`collisionFlags/maskFlags = 9`） |
| `Asset/Anime/Character/Zombie/Boss/BossDave/` | 带护具的 BOSS（`Asset/Anime/Character/Zombie/Boss/BossDave/Armor/Config/ZombieBossDaveArmorBossDaveDoomShield.tres` + `Registry/Armor/Config/BossDaveDoomShield.tres`） |

⚠️ **`DamagePoint` 不是受击盒**：`CharacterDamagePointData` / `CharacterDamagePointConfig`
（`Resource/General/Character/DamagePoint/`）是**外观阶段**系统 —— `damagePersontage` 当**阈值**排序，
到点切 `animeFliterOpen/Close` + `replaceMediaName` 换贴图/播特效。**与「多处独立受击」无关**，别搞混。

### 4b.4 直接 `res://` 复用僵王火球（不用复制美术）

```
弹体：res://Asset/Config/Projectile/EdgarII/ZombieBossEdgarIIFireball.tres
      （baseDamage=2000, scale=(2.25,2.25), fireMethodFlags=4, penetrateNum=-1,
        damageFlags=7, collisionFlags=63, useDurabilityBlockingSweep=true, splatScene=FireSplats）
粒子：res://Asset/Anime/Character/Zombie/Boss/Boss/Effect/FireBall/FireBall.tscn
      （GPUParticles2D + FuelBall.tres 的 ParticleProcessMaterial，纹理 …/Particles/FireBall/FireballParticles.png）
```
包内 `res://` 指向**游戏自带**资源是**合法**的（`SanitizeCharacterTextResource` 放行），
只有指向**包内自己**才必须写相对路径。

### 4b.5 多部件素材拼链（「给了头/体节/尾三张图，自行裁剪组装」）

用户给几张**部件特写**图 + 一张全身参考图时，**先定每张图的朝向再谈裁切**（2026-09-27 实测）：

1. **先判视角**：三张部件图若都是**正视/俯视特写**（左右对称、有「正面正对镜头」的观感），
   而参考全身图是**侧视横排**，则部件必须**旋转 90°**才能拼成侧视链。
2. **旋转方向靠「三个独立判据」定，别猜**：
   ① 前进方向（僵尸朝左）⇒ 口器/头部尖端要落在左边；
   ② 冠状角刺在侧视里应朝**上下**（若在部件图里朝左右 ⇒ 说明它是俯视，转 90° 后正好）；
   ③ 尾端必须「锥形环在左、大眼在右、尾刺朝右」—— **尾图往往要转 CCW 而头/身转 CW**，
      这一步最易搞反（转了 CW 会整条尾左右颠倒）。
3. **比例自洽的硬判据**（用来证明不是「看着差不多」）：
   单环宽高比必须等于参考图里单环的宽高比。本项目实测 `694:1795 = 0.3866` vs 官方 `116:300 = 0.3867`。
   头宽 / 环宽也应与参考图一致（本项目 5.1 vs 官方 5.04）。
4. **★★ 逐链间距不要用「统一 overlap 百分比」**：`spacing = 宽 × 0.90` 在
   **头（宽 295）→ 首环（宽 63）**这种悬殊跳变处会留下可见空档（头是菱形体，右端本就收细）。
   正确口径是**逐链像素重叠**：`link[i] = w[i]/2 + w[i+1]/2 − OVL[i]`，
   `OVL` 按链接逐个给（本项目 `{0:118, 1..8:14, 9:78}`）。
   ⚠️ `OVL[0]` 常常不是「设」出来的而是**结果的副产物**（头半宽 147.5 + 首节半宽 63 − 深插 92.5 = 118）。
5. **抠底一律 flood fill（限「颜色≈背景 **且** 连通到画布外缘」）**，禁止「全图同色即透明」
   （内部近黑描边会被误伤）。
   ★★ **「只保留最大连通域」一举去掉右下角 AI 水印** —— 水印字符是独立小连通域，
   比写专门的去水印逻辑简单可靠得多。
6. ⚠️ 脚本里 `Image.new((w, h))` 的 `w` 若来自 `sum()`/`max()` 会是 `float` ⇒
   `TypeError: 'float' object cannot be interpreted as an integer`，记得 `int()`。
7. **★★★ 「某档位 + 某手法」必须反解，不许想当然**（2026-09-27 实测）：
   用户要「体节加到 C 档尺寸（in-game `126×163`, w/h 0.773）+ 用**横向拉伸**实现」——
   单环 trim 后 ratio 是 `0.4110`（不是外框的 `0.3866`），所以拉伸倍率 = `0.7730/0.4110 = **1.875**`，
   **不是**直觉上的 2.0（2.0 会得到 `134×163 / 0.822`，档位就错了）。
   ⇒ 任何「倍率」「比例」都写一个探针脚本实测反解（本项目 `_src/step8_ring_probe.py`），
   半分钟的事，能挡住一整轮返工。
8. **★★ 锚点取「bbox 中心」时，必须检查「实体」而不是「外框」能不能接上**：
   本项目尾部实体球在 bbox 内**居中偏左**（逐列 alpha 覆盖率 `x≈10..118 / 171`），右半是一根细长刺。
   按 bbox 中心当锚点时，26px 重叠会让**球体够不到前一段** ⇒ 上方张开楔形缺口
   （实测 223px 封闭空洞，肉眼明显）。可行带 `[62, 86]`，取 78。
   ⇒ 诊断命令：`逐列覆盖率 = (alpha>0).mean(axis=0)`，找 «覆盖率首次 ≥0.55 的列»。
9. **★★ 最小转弯半径是硬约束：`R_min = link / (2·sin(θmax/2))`**
   （2026-09-27 实测：`106.5 / (2·sin5.5°) = **566 px**`）。草坪本体才 720px 宽 ⇒
   **每关节 11° + 节距 106px 时，本体最紧只能弯 566px 半径，S 型只能呈现为缓和长弧**，
   做不到紧绷蛇形；渲染图里会看到多个关节**顶在 ±θmax 上限**、本体「抄近路」穿过 S 路径。
   ⇒ 「关节角」与「节距」是耦合的，**选角度时必须同时算 R_min**，别只说「最多能弯 N×θmax 度」。
   ⇒ 想更蛇形就同步加大 θmax **和** OVL（但它们不是单调关系，见第 10 条）。
10. **★★★ 接缝闸门的方法论（踩过，代价大）**
    - **差分指标抓不到「两件根本没接上」**：以「同角刚性姿态」为基线（这是抵消美术自带镂空的唯一干净办法）时，
      若 link 太大导致两件不相接，**刚性与弯曲都不接** ⇒ 差分 = 0 ⇒ **假绿**。
      ⇒ 必须再配一个**绝对**指标（缝处腰宽 / 厚度比 / 连通性）。
    - **绝对指标的阈值跨关节不通用**：环→环在视觉已干净的 OVL=14 处 ratio 仅 **0.73**；
      尾→环在视觉仍有缺口的 OVL=26 处 0.76，要到 78 才 **0.98**。⇒ 单阈值不可行。
    - **可行性是「有界区间」不是单调的**：重叠太小露缝、**太大从另一侧穿出** ⇒
      **不能二分**，要逐点扫剖面（本项目 `_src/step12_ovl_profile.py`）。
    - **最终值以「2× 放大 + 纯色底」的视觉扫描为准**（本项目 `step9b_tail_zoom.py`：青底 → 漏底即为缝），
      闸门只负责「弯曲不新增缝」的**回归防护**。**把这条局限如实写进脚本 docstring。**
    - 底用**青色**（`(0,255,255)`）最灵：任何漏底一眼可见；白底会被高光骗过去。
11. **★ 弯曲/游走演示图的作图铁律**（不遵守就出「看不出 S」的废图）
    - **路径总长必须 ≈ 链长**：本体只覆盖「头当前位置往后**一个链长**」这一段
      （拖尾运动学）。路径 3600px 而链 966px 时，无论怎么渲都只是「一小段弧」。
      ⇒ 给路径函数加 `target_len` 归一化。
    - **缩放要绕「外框中心」不要绕「形心」**：形心不在外框中心 ⇒ 本体被偏置出画。
      最稳的最终做法 = **渲到透明画布 → `getbbox()` 取内容外框 → 居中贴到底图**（永不出画）。
    - **把头走过的路径画成青色虚线**：本体「抄近路」的差值一眼可见，是解释 R_min 最好的图。
    - 关节角**标在图上**（`±11°`），一眼就能看出是否顶到上限。

### 4b.6 包骨架：「N 段零污染」+ 自制图集零换算（2026-09-27 已全套闸门验证）

| 想知道 | 结论 |
|---|---|
| N 段会不会污染**卡库 / 图鉴 / 波次** | **不会，只要只给头段写 `Resources/Cards/<Key>.tres`**。`Resources/Cards/` 才是**注册**位置；其余段的 Packet 只写进包内镜像 `Packet/<Key>.tres`（不注册）。插件生成它们 **直接从运行时注册表取 PackedScene**（见 §4b.8 —— ⚠️ **不要**用 `ModLoader.TryInstantiateEffectiveCharacter`，它对纯资源包会失败且可能整包回滚），只依赖 manifest 的 **`provides.Character`** ⇒ **不需要 Packet** |
| `provides` 怎么写 | `Character`: N 个键全列；`CharacterSprite`: N 个键全列；**`Packet`: 只头段 1 个键** |
| 自制 `.dat` 的锚点口径 | 把 `sliceTransforms` 设为 **identity**、`origin = (0,0)` ⇒ 每张部件图的**图像中心 = 角色原点** ⇒ 上游算好的 `link[i]`（圆心间距口径）**可直接当角色原点间距**，插件零换算。Scene 里再用 `TransformPoint(12,36)` + `<Key>(−12,−36)` 让净位移 = 0 来闭合 |
| 能不能只做一份图集 | 能：N 段**共用同一份** `.tres`，靠 `Animation/Clip` 选自己那一帧（`.dat` 一个文件承载一张图集 ⇒ 把 N 片拼一张） |
| 两份构建要各跑一遍吗 | 要（`remake` / `console` 的 `PlantsVsZombies.dll` 字节不同）。★ 但实测**插件编译产物两份完全相同**（程序集引用只记名/版本，不记 MVID）⇒ 差异只在运行期。⚠️ console 那份的真实目录是 `D:\zzz\植物大战僵尸**杂交**重制版\data_PlantsVsZombies_windows_x86_64` |

### 4b.7 ★★★ 手写 `.tscn` 的「三方一致」：`[node parent]` / `NodePath` / 实际路径

**症状**（`《星空机械蜿蜒》` 实机首测，2026-09-27）：
图鉴里卡牌**图片与描述全空**；卡牌能选中但**拖放后场上什么都不画**；**零崩溃、零显眼报错**。

日志只给两条（但足够定案）：
```text
ERROR: [InformationPanel] Character preview 'X' has incompatible serialized properties.
ERROR: [TowerDefenseCharacter] Disabled invalid runtime character 'X': config or sprite is null/invalid.
```

**根因**：场景里写了

```text
sprite = NodePath("SpriteGroup/TransformPoint/<Key>")            # ① 期望路径
[node name="<Key>" parent="SpriteGroup/TransformPoint/<Key>" …]  # ② 实际会挂成 …/<Key>/<Key>
```

Godot 里 `[node name="X" parent="P"]` 的**完整路径 = `P/X`** ⇒ ②比①**多一层**
⇒ `NodePath` 解析为 **null**。而 Godot 对「`parent` 路径中不存在的中间节点」会
**静默造一个匿名 `Node2D` 占位**（日志里的 `'@Node2D@1370'` 就是它）⇒ 现场一片安静。

**为什么「图鉴空」和「场上不画」是同一个 bug** —— 两处 C# 判据都是 `sprite == null`：
- `Prefab/GUI/InformationPanel/InformationPanel.cs:151-157`：命中即 `PushError` 后**直接 `return`**
  ⇒ 后面的 `nameLabel` / `expressionLabel` / 数值**一个都没赋**；
- `Prefab/TowerDefense/Character/TowerDefenseCharacter.cs` `DisableInvalidRuntimeCharacter()`：命中即判 invalid。

**正确写法（三处交叉验证，别自己发明）**：
`addons/ModEditor/FileSystem/XWResourceCreateRoute.cs:650`（ModEditor 官方「新建角色」模板，
逐字 `[node name="{0}Sprite" parent="SpriteGroup/TransformPoint" …]`）
＋ `SuperGatlingPaper`（实机可用的换头僵尸，`parent="SpriteGroup/TransformPoint"`）
＋ 基场景 `Prefab/TowerDefense/Character/TowerDefenseZombie.tscn:39`（`TransformPoint` 的 parent 是 `SpriteGroup`）。

**纪律**：
- ★ 自检**必须连「整个节点声明行」一起断言**（`[node name="{key}" parent="…" index="0" instance=…]`），
  只查 `sprite = NodePath(…)` 那一行是**自检盲区** ⇒ 全绿而实机不画（假绿第 ④ 类）。
- ★ 再加一条**负向**断言：源码里不得出现 `parent="…/<Key>"`（复刻这个 bug 的形态）。
- ★ **调试第一动作 = 读 `logs/godot.log` 的 ERROR/WARN，再读源码，别推理。**
  日志在 `%APPDATA%\Godot\app_userdata\植物大战僵尸杂交版\logs\godot.log`（最新会话）
  ＋ `godot<时间戳>.log` 归档 ＋ `PVZHE_Logs/godot_startup_*.log`。
  「零崩溃、零报错、就是不显示」这类症状，日志几乎一定有 `ERROR`。

### 4b.8 ★★★ `CompanionOnly` 是双刃：找不到类会**整包回滚**——纯资源包别用

**症状**：`WARNING: 生成第 N 段失败…诊断：角色场景缺少 CompanionOnly 伴随脚本元数据。`

**机制**：`ModLoader.TryInstantiateEffectiveCharacter`（`addons/ModEditor/ModSystem/ModLoader.cs:843`）
**只走** `XWModCharacterCompanionRuntime.TryCreateInstance`（`XWModCharacterCompanionRuntime.cs:160`），硬要求：

1. 场景 root 带 `metadata/mod_character_script_binding = "CompanionOnly"`（`:180`）；
2. `metadata/mod_character_script_path` 非空，且**文件名去扩展**得到的类型名
   **必须存在于 mod 的托管程序集里**、非抽象、且 `authoredRoot.GetType().IsAssignableFrom(type)`（`:185-192`）
   —— 然后 `Activator.CreateInstance(type)` 造新节点**替换**场景 root。

★★ **致命点**（`ModLoader.cs:630-638`）：场景带了 `CompanionOnly` 却找不到该类 ⇒
`TryCreateInstance` false ⇒ **`return false` 把整包回滚**
（`[ModLoader] package rolled back after companion binding failure: <id>`）。
⇒ 对「纯资源包」（场景就是普通 `TowerDefenseZombie`、没有 per-segment 脚本）**根本不是可选项**。

**正路**：直接用运行时注册表自取场景并实例化 ——

```csharp
if (XWModRuntimeRegistry.TryGetEffectiveRegistration("Character", key, out XWModRuntimeRegistry.Registration reg))
{
    PackedScene scene = reg.ModValue.As<PackedScene>();   // = ModLoader.cs:644-653 注册的 Variant
    Node root = scene.Instantiate();
    TowerDefenseCharacter seg = root as TowerDefenseCharacter;
    if (seg == null) { root.Free(); }
    // AddChild 之后再 SetGlobalPosition（入树才 _Ready ⇒ _Init ⇒ hitpointsSave 才有值）
}
```

`Registration.ModValue` 就是 `ModLoader` 在 `ApplyMod` 时注册进 `Category == "Character"`
的那份 PackedScene ⇒ 「资源能否加载」与卡牌放置时**完全同源**。
⇒ 生成器 `BANNED_TOKENS` 里加 `"CompanionOnly"`（产物侧扫描）+ 自检加断言防它回来 + 一条负向用例。

### 4b.9 ★★★ 缩放：**拆成两层**（单卡 / 整体布局），间距还要带「不重叠下界」

**背景**：自制 `.dat` 的 `sliceTransforms` 是 identity + `origin = (0,0)` ⇒ 部件图像
**1 px : 1 游戏单位**落地（草坪 = 9 格 × 80 = 720）。而 Phase 1 的 `skin_params.json`
是**像素域**真源（`Σlink = 1022 px`）⇒ 直接落包会比草坪还长 74%。
⚠️ Phase 1 的 `display_scale` **只用于预览图**（把 1400×600 地图按真实比例渲到交接图），
**从来没有进入包** —— 别以为它在生效。

**修法**：拆成**两层**常量，职责分离 —— 只有这样「用户说放大 N 倍」才说得清**作用在哪**：

| 层 | 常量 | **作用对象** | 落点 |
|---|---|---|---|
| **A 单卡缩放** | `GAME_SCALE = S` | **每一个部件自身**（= "单张卡片"）。**不决定段间距** | Sprite 节点 `scale = Vector2(S, S)`；hitbox `Size = Vector2(w×S, h×S)`；图鉴/卡面预览 `packetAnimeScale`/`Offset` |
| **B 整体布局缩放** | `LINK_SPREAD = k` | **段间原点间距**（= "整体"的水平跨度）。**完全不动部件尺寸** | 插件里那 N−1 个数 |
| C 不重叠预留缝 | `LINK_GAP_GAME` | 相邻两段「半宽和」之外再留的缝 | 同上 |

⇒ `link[i] = max( 像素值 × S × k , (w[i] + w[i+1]) / 2 × S + GAP )`

★★ **那个 `max` 不是装饰**：上游 `links[]` 里通常含**刻意重叠**
（为让接缝零漏缝，例：`OVL[0]=118` 头→首节深插、`OVL[9]=78` 末节→尾）。
它们是「间距 < 半宽和」的**负**重叠 ⇒ **线性缩放的倍率改不了符号**
（实测：间距 ×4 之后这两处**仍**重叠 12.75 / 5.75 游戏单位）。
⇒ 用户只要说"**不重叠**"，就必须用下界顶高，否则乘几倍都还压着。

**⚠️ 漏乘的后果分三种，别混**：
- **A 漏**（Sprite 不缩）⇒ 1 px 源素材 = 1 游戏单位，整体过大；
- **B 漏**（间距不缩）⇒ 部件缩了而间距没缩（或反之）⇒ **蛇身处处脱节**；
- **hitbox 漏** ⇒ 判定悬在图像外（打不到 + 打空），**零日志**。

**纪律**：
- ⚠️ **不要**改 `skin_params.json`（含 `links[]`）：Phase 1 的两道闸门读它 ⇒ 一改就红。
  缩放只加在**生成器给游戏侧的**那份上。
- ⚠️ **不要**改 `.dat` 的 `sliceTransforms`（`.tres` 会与 `.dat` 对不上，`verify_pmod` 与自检都会红）。
- ★ 断言里的期望值要用**独立基线常量**（`GOLD_GAME_SCALE` 等）算，**不要**复用产物侧那个已经乘过的 helper
  （`_scaled(w * GOLD_GAME_SCALE)` = 双重缩放 ⇒ 自己造出 N 条红；实测被自检当场抓住）。
- ★ 「像素原值不得出现」这类负向判据要**条件化**：当等效倍率 `S × k == 1` 时它与"正确值"
  本来就重合 ⇒ 那种口径下只会**假红**。改判「下界是否真的被顶起来了」。
- ★ **必做三条断言**（都是用户口径，不能只靠"我算过"）：
  ① **不重叠** `link[i] >= (w[i]+w[i+1])/2 × S`（逐关节）；
  ② **不溢出** `首半宽 + Σlink + 末半宽 <= 地图宽`
     ⚠️ "溢出"= 超出**地图**；**不是**超出草坪（一行 9×80=720）—— Boss 比草坪长属设计；
  ③ **口径史**：用户会反复改（实测 `0.75 → 0.25 → 0.50` ＋ 布局 `×2`）⇒ 历史写进 docstring，
     保证"改回只需动固定几处 + 有断言兜底"。
- ★★ **「要不要重叠」是观感问题，不是对错 ⇒ 做成显式开关**（用户实际两次改主意：先要不重叠，
  看到效果又回退）。`LINK_ENFORCE_NO_OVERLAP = False`（忠实沿用上游 `links[]` ⇒ **连成一条**，
  含刻意重叠）/ `True`（下界顶高 ⇒ **段段分开**）。✔ 断言必须**跟着模式条件化**
  （§16 分走「重叠量 == `OVL × S`」或「逐关节不重叠」两条路），并配一条负向专门守
  「断言真的按模式走」（把金标切到 True 而实现留 False ⇒ 立刻红）。
  ⚠️ **诊断脚本的结论文案也要跟着口径走** —— 本例布局表原来**写死**"必须不重叠"，
  模式一关就把**预期行为**报成 `FAIL`（**假红与假绿同样带偏判断**）。
- ★ 交付时给一张**布局验算表**（脚本 import 生成器取真源，改口径自动跟着变）：
  逐段的像素/游戏尺寸、链上原点 x、与上段的间隙（负数 = 重叠）＋ 整体跨度 ＋ 不重叠/不溢出结论。
  （本仓样板：`.cache/_layout_table_ss.py`。）

### 4b.11 ★★ 排查「自制 `.dat` 角色不显示」的正确顺序（2026-09-28 实证，含两次误判）

**别急着改渲染设置 —— 按这个顺序查，每一步都要有日志或源码依据：**

1. **先读 `logs/godot.log`**（第一动作，不要先推理）。`sprite is null` /
   `incompatible serialized properties` / `Disabled invalid runtime character`
   这三条只要命中一条，**根因就在场景 NodePath 或 config**（见 §4b.7），与美术/渲染设置无关。
2. **看节点是否被反复重建**：日志里 `已生成 N 段（累计 M）`，**M 持续增长 = 有段在被外部销毁**
   ⇒ 屏幕上当然看不到（或只闪一下）。此时**加诊断日志**（打印**具体键** + `IsInstanceValid` 判定 + 帧号）
   比继续翻渲染代码有效得多。
3. **最后才查渲染设置**，且必须**对齐一个同类型、实机可画的金标包**
   （僵尸/植物：`SuperGatlingPea` / `UltimateCherryGod`）。

**⚠️⚠️ 关于 `forceLocalRender` / `forceCpuPoseRender`（两个高频误区）：**
- 它们治的是「**动画静止 / ESC 暂停跳帧**」（= **能显示但不动**），**不是"完全不显示"** ——
  症状别认错，认错就是白忙一整天（实测踩过）；
- 它们是 **`public` 非 `[Export]`** ⇒ **写在 `.tscn` 里完全无效**（Godot 静默忽略，连 warn 都没有），
  只能在**插件运行时**用 C# 赋值。**产物场景里不该出现这两行。**
- 实机可画的金标包**都没有**它们，而**都有**：`useMultiMesh = true` / `useTween = false` /
  `Animation/LayerVisible/<每个层名> = true` / `Animation/MediaReplace/<每个媒体名> = null`。
  （层名 / media 名从图集 `*_params.json` 的 `layer_name` / `media_names` 读，别手写。）

**⚠️ 同一语义在不同代码路径上可能结论相反（差点被骗）：**
`AdobeAnimateSprite._Get()` 里判 `Animation/LayerVisible` 是
`num < _layerVisible.Count && _layerVisible[num]` ⇒ **空数组返回 false**；
但**渲染真正消费**的是 `_cachedAllLayersVisible`（`:9045-9062`）⇒ 空数组时 `i >= num` 恒真 ⇒
**保持 true（全可见）**。⇒ **必须找"真正被渲染消费的那个变量"**，不能看到第一处 `return false` 就下结论。

**⚠️ `alpha.getbbox()` 对抠图产物没有判别力**：低 alpha 噪点会把框撑满整幅图
（实测 11 张全部 `(0,0,w,h)`）⇒ 必须**按阈值**（如 `>= 128`）过滤后再取 bbox，
或直接统计"四角 / 四边中点 alpha"。

**⚠️⚠️ 最重要的一条：「日志里没有本 mod 的报错」≠「本 mod 没问题」。**
真凶可能发生在**别的子系统**里，backtrace 里**根本不出现 mod 名字**。
本仓实证：插件生成的段被 `CharacterRegister` 自动注册进 `TowerDefenseManager`，
而关卡编辑器保存时（`LevelEditorMapEditor.Save()`）会**遍历所有已注册角色**并调
`CreatePreSpawnConfig`，其中 `:702` 有一行**裸访问** `character.packet.saveKey`
⇒ 段的 `packet` 为 null（它不是"卡牌放置"来的）⇒ **NRE** ⇒ 编辑器保存/测试失败
⇒ 用户"拖放后看不到任何结果"。
⇒ **查"mod 不生效"时，除了 grep mod 名，一定要把同期全部 ERROR/WARN 看一遍**
（用 `sed -n 'A,Bp'` 看上下文，别只 grep）。修复：`packet` 是 public 字段，插件给它补上
（引擎先例 `BugOverviewDolphinPlacementJumpBlockRuntimeTest`）。

**⚠️ 同理：自检/诊断日志本身也会误报。** 本仓"段失效"日志最初把
**首次生成前的 `null` 空位**报成"被回收"（`第 1 段…C# 引用为 null；帧=518`）⇒
必须加"**曾经生成过**"门控再报警。

**⚠️⚠️ 给"运行时生成的伴随角色"补 `packet` 时，必须成对设 `characterFilter = true`。**
本仓实证的完整因果（走了一轮冤枉路）：
- 关卡编辑器 `LevelEditorMapEditor.Save()` 会**遍历 `TowerDefenseManager.GetCharacter()`
  的每个已注册角色**，为每条写 `preSpawnList`（`packetName = character.packet.saveKey`，
  `:702` 是**裸访问**）⇒ 伴随角色 `packet == null` 就 **NRE**（编辑器保存/测试失败）；
- ⇒ 补 `seg.packet = 头段的 packet` 修好了 NRE，**却把 10 段"伪装"成头段** ⇒
  10 条 `packetName` 全等于头段 ⇒ **进关卡时引擎 spawn 出 11 个头段**；
- ⇒ **正解**：`seg.packet = …`（挡 NRE）**＋** `seg.characterFilter = true`
  （`TowerDefenseCharacter.cs:365` public 字段；`TowerDefenseManager.cs:520-529`
  `GetCharacter()` 里 `if (!item.characterFilter) array.Add(item)` ⇒ 设 true 就不进这份清单；
  UI 标签「只响应角色」`XWBattleFeaturePresenter:508`；引擎自己就这么排除推车
  `TowerDefenseBattleFeatureMower:107`）。**它不动 `characterRegistry`** ⇒
  索敌/碰撞/受击全不受影响，只是"不被当作可保存的关卡内容" —— 正是伴随角色应有的语义。

**⇒ 通用纪律：给"运行时生成、不属于关卡存档"的对象补上"关卡内容才有的字段"时，
必须同时把它从"关卡内容枚举"里排除出去。两条缺一不可。**

### 4b.12 ★★ 贴图相对受击盒「偏右下」⇒ 缺居中 `offset`（自制 `.dat`，2026-09-28 实证）

**症状**：11 段都能显示、也排成一条，但**贴图整体偏到受击判定的右下方**（偏移量约半个图幅）。

**根因**：`AdobeAnimateDrawItemBuilder.cs:683-689`（走 `sprite.Texture` 的那条路径）
```csharp
rect      = ResolveSpriteSourceRect(sprite, texture);     // 源矩形（**像素**尺寸）
vector    = ResolveSpriteDrawOrigin(sprite, rect.Size);   // = sprite.Offset − size/2（仅当 Centered）
transform = new Transform2D(axisX * size.X, axisY * size.Y, vector);
```
纹理 `[0,1]²` 映射到「**以 `vector` 为左上角**、边长 `size`」的矩形 ⇒ **`vector` 就是图像左上角**。
⚠️ `AdobeAnimateSprite` 在这条路径下 `Centered` **不是** true（它自己管绘制原点）
⇒ `ResolveSpriteDrawOrigin` 里那个 `− size/2` **不会**发生
⇒ 只给 `offset = 0` 就等于「**图像左上角落在角色原点**」⇒ 整幅偏**右下**。

（`externalAtlasCentered`（`AdobeAnimatePart.cs:33`，默认 **true**）只作用于**外部图集部件**
 `AdobeAnimatePart` 那条路径；自制皮肤走的是 `sprite.Texture` 路径，用不到它 ——
 别看到它默认 true 就以为已经居中了。）

**修法**：Sprite 节点显式给 **`offset = Vector2(−w/2, −h/2)`**，**像素值**
（`offset` 在节点本地空间生效、会随 `scale` 一起缩放 ⇒ **不要**乘 GAME_SCALE）。
每个部件按自己的 `w/h` 给，不要统一一个值。

**★ 交叉证据（以后照这个查）**：拿一个**实机可画的同类包**看它的 `offset` ——
它一定与"图像尺寸的一半"有明确关系。本仓：`UltimateCherryGod` 帧 138×129 给
`offset = (-69, -93)`，其中 **x 分量 `-69` 恰好 = `−138/2`**（自己补居中量），
多出来的 y 是它把锚点放在**脚底**（比中心再低 28.5px）。

**★ 概念区分（易混）**：`.dat` 的 `origin = (0,0)` 是**数据侧口径**，只表达
"部件图像中心 = 角色原点"这个**设计意图**；要**让引擎真的按这个画**，
还必须在 Sprite 节点上给足 `offset`。**两者不是一回事，缺后者就偏。**

## 5. 闸门总览（离线全套，缺一层就可能白跑实机）

| 门 | 命令 | 验什么 |
|---|---|---|
| 生成器自检 | `python build_zombie_<Key>.py --self-check` | 常量/结构/金标/跨语言一致性（`fails = 0`） |
| 负向测试 | `python .cache/_neg_test_head.py` | 逐条篡改常量，断言**必须响**（含注入式用例） |
| 几何门 | `python .cache/check_head_fit.py` | 头对位反解 + 换头落点 + **炮口/生成点**（两个独立来源） |
| 字节核对 | `python .cache/_byte_check_head.py` | 落盘产物**按字节**核对（含工作区 == 安装镜像） |
| 插件入口 | `python .cache/run_entry_sgp.py` | 入口类型 public/非嵌套/有无参构造/实现接口 |
| ModLoader 闸门 | `python .cache/run_gates_sgp.py` | **两份构建**各跑一遍 `check_gates_*.cs`（反射读 DLL + 真产物文本） |
| 幂等 | `python .cache/check_sgp_idempotent.py` | 3 连跑同 sha256 + 只读产物断言 |

**两份构建都要跑**：remake `…\0.28.1\植物大战僵尸杂交重制版\data_PlantsVsZombies_windows_x86_64`；
console `D:\zzz\植物大战僵尸**杂交**重制版\data_PlantsVsZombies_windows_x86_64`
（⚠️ 旧笔记里的 `D:\zzz\植物大战僵尸重制版\…` **不存在**）。

**交付文档与产物必须事实一致**：改完代码顺手刷指纹（大小 / sha256 / 条目数 / 场景字节数 / 闸门条数）。

### 5.1 ★★★ 负向测试的六类「假绿」（2026-09-27 先抓 9 条、同日再补 3 条，务必逐条自查）

写下任何断言后，**先问「把实现改坏了这条会不会响」**，再问「有没有可能实现和断言同时被改」：

1. **断言与实现同源**：`if _num(cfg, "physique") != seg["physique"]` —— 而 `seg["physique"]`
   就是写进 cfg 的那个值 ⇒ 常量改成 5（= `CAR`）也 0 条失败。
   ⇒ **断言必须比 `GOLD_*` 字面量**，不能比生成常量。（本仓铁律：`GOLD_` 声明了却没人引用 = 红旗。）
2. **★★ 断言里内联了同一个字面量**（比 `GOLD_` 前缀稀释更隐蔽）：
   `position = Vector2(12, 36)`、`Size = Vector2({seg['w']}, {seg['h']})`、
   `LocalTransform = Transform2D(1, 0, 0, 1, 0, 0)`、`unlockCheckList = []` 这类字面量
   在源码里**各出现两次** —— 一次在**模板**（实现）、一次在**断言**里内联（金标）。
   `str.replace(old, new)` 全量替换会把两边一起改 ⇒ 恒真 ⇒ 假绿。
   ⇒ 锚点必须带**模板专属上下文**（例：`'position = Vector2(12, 36)\n\n[node name="{key}" parent="{base}"'`）。
   （`GOLD_` 前缀 / 行首 `\n` 只能挡住第一类，挡不住这一类。）
3. **判据抓不到被测常量**：`len(SEGS)` 只由上游 json 决定 ⇒ `N_SEG` 常量被改时抓不到；
   段号判据又用 `N_SEG - 1` 跟着漂。
   ⇒ 直接钉常量本身（`if N_SEG != GOLD_N_SEG`），并用 `GOLD_N_SEG - 1` 判段号。
4. **★★ 只钉「属性行」不钉「节点行」**（同日新增，代价最大的一次）：
   自检只断言 `sprite = NodePath("SpriteGroup/TransformPoint/<Key>")` 这一**属性行**，
   **没断言**承载它的 `[node name="<Key>" parent="…"]` **节点声明行** ⇒
   节点挂错层（`parent` 多一级）时自检**全绿**，而实机「图鉴空 + 场上不画」。
   ⇒ 场景这类**结构化文本**必须连**整行声明**一起断言 + 一条负向（禁止复刻该形态）。
   （详见 §4b.7。）
5. **期望值双重换算**（同日新增）：断言里写成 `_scaled(w * GOLD_GAME_SCALE)` —— 而
   `_scaled()` 内部已经乘过一次 `GAME_SCALE` ⇒ 结果 ×S² 。这类错误**自检会当场抓住**
   （实测 11 条 `[受击盒]` 全红），但若只在负向测试里跑就会误判成「断言太严」。
   ⇒ 期望值用**纯格式化 helper**（只补 `.0`，不乘任何系数）+ **独立基线常量**算。
6. **★★ 只断言「抄写目标 vs 金标」，放空「实现产物」**（同日新增，最贵的一条）：
   生成器**算出来的**值（如段间距数组 `LINKS_GAME`）与"产物里**手写**的常量"是**两份东西**。
   只断言「产物里手写的 == 金标期望」会放空前者：实测把整体布局倍率常量改了却 **0 条失败**
   —— 因为抄写目标与金标两边都没动，而生成器算的那个数组漂了、**没有任何断言盯着**（它此前
   只是"给人抄的草稿"）。
   ⇒ **凡是"算出来准备给别人抄"的值，也要有一条对它自己的断言** —— 实现产物必须自我对账。

⚠️ **断言自身也会被负向测试打崩**（同日实证）：
- ★ 循环上界若用**金标常量**、却索引**由被测常量决定的数组** ⇒ 上游一改就 `IndexError`
  **把整个 `self_check` 崩掉**（而不是报 FAIL）。
  ⇒ 先做**长度守卫**（长度不符就 `append` 一条 FAIL），再 `range(min(金标长度, 实际长度))`。
- ★ 新增断言后，老负向用例的 `expects` 可能从"命中 A"变成"命中 B"（更早更准）⇒
  看到"未命中"先确认是不是新旧断言换位，别急着删用例。

⚠️ **环境坑**：测试脚本 `main()` 里既有读全局 `BAD` 又有 `BAD += 1` ⇒ 不写 `global BAD`
会让它变成局部名，末尾 `if BAD:` 抛 `UnboundLocalError` —— 而报错发生在**所有步骤都跑完之后**，
日志看起来全绿，极易误判成「通过」。（另外：`verify_pmod.py` 在 `ModWorkspace/` **根**，不在 `.cache/`。）

---

## 6. references 索引

| 文件 | 内容 |
|---|---|
| **`references/zombie-asset-requirements.md`** | ★★★ **开工先读**：素材问卷（可照抄的问法）、官方规格基准（`tileSize 105×134` / 图集 `3360×1206` / `columns 32` / `trueFrameRate 180.0` / 各插槽 `offset`）、素材来源地图（哪些原版资源可直接复用）、三种素材形态的处理路径、**没有素材时的降级路线**、素材自检四步 |
| `references/zombie-package-and-gates.md` | 包结构逐字段、护具体系、卡库/图鉴、闸门与幂等脚本写法 |
| `references/zombie-fire-and-marker.md` | 发射体系（僵尸侧）、真源链路、**子弹生成点对齐炮口**（两层实现 + 判据写法） |
| `references/zombie-stats-and-cc.md` | **数值/速度/伤害类型**的字段落点（`attackType`、`hitpointsNearDeath` 语义）、手册五行属性块格式、**`.tres` 裸换行陷阱**、**★★★ `_ground` 根运动层（僵尸为什么不走 + 配方 + 移速怎么调 + 闸门）**、**用 buff 让植物停止发射**（`timeScaleInit=0` 为何无效 + 全路径杠杆 + 三条纪律），全部附源码行号 |
| `references/zombie-skin-and-head.md` | 换外观：经典 reanim 直转、`.dat` 二进制规格、换头三节点与头对位反解、**§9 子精灵排序（`insertLayerId`）**、**§10 尺寸类需求的量化流程**、**§11 换「被跟随层」修抬头段头/身接缝（1D+2D 判据）**、**§11b 位姿必须同帧（`process_frame` 早于 `_process` ⇒ 头落后 1 帧，用 `CallDeferred` 覆盖）** |
