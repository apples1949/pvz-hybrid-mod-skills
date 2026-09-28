# 僵尸「数值 / 速度 / 伤害类型」规格 + 用 buff 做控场

> 本文覆盖两块内容，都是踩过坑之后才确认的：
> **A. 需求里那几个「像字段但不在 config 里」的属性到底落在哪儿**（速度、伤害类型、手册属性块）。
> **B. 让植物「停止发射」的正确做法**（`timeScale=0` 是不够的，必须用 buff）。
>
> 全部结论都对着 `D:\zzz\pvzHE\解包\植物大战僵尸杂交版V0.28` 的真实源码/资源核过，附文件与行号。

---

## A. 属性的真实落点

一个常见误区：把「移动速度 / 伤害类型 / 生命值」当成同一类字段去找。
实际上它们分散在**四个不同的地方**，其中两个完全不在配置文件里。

| 需求说法 | 真正的字段 | 文件 | 备注 |
|---|---|---|---|
| 生命值 | `hitpoints` | `Config/TowerDefenseZombie*.tres` | 继承自 `TowerDefenseCharacterConfig` |
| 啃食伤害 | `attack` | 同上 | `TowerDefenseZombieConfig.attack` |
| 濒死相关 | `hitpointsNearDeath` | 同上 | **不是阈值、是掉血速率**，见下 |
| 伤害类型 | `attackType` | **`Scene/*AttackComponentDefinition.tres`** | 不在 config、不在卡牌 |
| 移动速度 | `walkSpeedScale` | **角色 `.tscn` 的节点属性** | 不在 config |
| 手册里的各项 | 纯文本 | Packet 的 `handbookDescribe` | 只是展示，引擎不读 |

### A1. `hitpointsNearDeath` 是「濒死掉血速率」，不是阈值

`Prefab/TowerDefense/Character/TowerDefenseCharacter.cs` 里：

```csharp
if (nearDie && !die && ...)
    instance.DealHurt(config.hitpointsNearDeath * delta / 3.0, playSplatAudio: false);
```

⇒ 它决定的是「进入濒死之后每秒掉多少血」。
**别按血量比例去缩放它**：内置普通僵尸 `200 / 70`，参照 mod `1180 / 70`，都是 70。
需求说「生命值设为 X」时，把 `hitpoints` 改掉即可，这一项保持惯例值。

### A2. 伤害类型 = `attackType`，只有 4 个取值

`Script/Component/TowerDefense/Character/AttackComponent/AttackComponentDefinition.cs`：

```csharp
[Export(PropertyHint.Enum, "Default,Eat,Smash,Chomp")]
public string attackType = "Eat";
```

模组编辑器 `addons/ModEditor/ResourceEditors/GUI/Panels/XWAttackContactPresenter.cs` 给的标题：

| 值 | 编辑器标题 | 说明 |
|---|---|---|
| `Default` | 普通 | 默认伤害标签 |
| **`Eat`** | **啃咬** | 植物被吞食语义 ← 需求说「啃食」就是它 |
| `Smash` | 砸击 | 重击伤害标签 |
| `Chomp` | 吞咬 | 吞咬伤害标签 |

**改法**（不改 exe、不改内置资源）：在角色包里加一份攻击组件定义，挂进本地 ComponentSet。

1. 新建 `Scene/<角色>AttackComponentDefinition.tres`，**逐字照抄**内置
   `Script/Component/TowerDefense/Character/AttackComponent/AttackComponentZombieDefinition.tres`，
   只加一行 `attackType = "Eat"`。
2. 本地 `Scene/<角色>ComponentSet.tres` 里 `Components = [ExtResource("<该定义>")]`。

**为什么这样能覆盖内置**（`Script/Component/Runtime/CharacterComponentSet.cs`）：

- `AppendFlattenedDefinitions` = 父级摊平 → `MergeLocalDefinitions`；
- `MergeDefinition` 按 **`InstanceId`** 命中就替换；
- 但 `CanReplaceInheritedDefinition` **强制**要求 `ComponentTypeId` 与 `WireIndex` 与内置**完全一致**，
  否则只 `GD.PushError` 并且**不替换**。

⇒ 三个值必须逐字对齐（内置僵尸的值）：
`InstanceId = "character.attack.0"` / `ComponentTypeId = "AttackComponent"` / `WireIndex = 0`。

另外 `useParentHitBox = true`、`checkLine = true`、`StateMachineDefinition` 也要保留
——漏了会静默改掉啃食判定范围。**建议把这几条写进闸门断言。**

> 内置的类默认值本来就是 `"Eat"`，所以对普通僵尸这类覆盖是**等值覆盖**：行为零变化，
> 价值在于把规格显式固化进包、并让编辑器/工具能读到。

### A3. 移动速度 = **动画里 `_ground` 层的根运动**（⚠️ 不是 `walkSpeedScale`）

**先记住这条，能省半天**：`walkSpeedScale`（`TowerDefenseZombie.cs:108`）**不影响移速**。
基类从不读它（只有 `SwanRider` / `Bed` 等派生类把它乘到 `sprite.timeScale`），
而 `GroundMoveComponent.RefreshMoveScale` 算的 `_moveScale` 里也没有它。
**它在基类僵尸上唯一的作用是被 spawn 时随机化（`:322`），实质是死字段。**

真正的移速 = **动画里 `_ground` 层的逐帧位移速率**，见 §B0。改动只需改一个数。

手册里的「移速：慢 / 稍快 / 快 / 非常慢」是**手写展示文本**，与代码无映射关系；
内置口径供参考（`Asset/Translate/Translate.zh.translation`）：

| 内置僵尸 | 手册 `移速：` | 手册 `walkSpeedScale` |
|---|---|---|
| 床车僵尸 | 非常慢 | 0.25 |
| 巨人系 | 慢 | 0.4 |
| **普通僵尸** | **慢** | **1.0** |
| 摇旗僵尸 | 稍快（对比普通僵尸） | ~1.5 |
| **橄榄球僵尸** | **快** | **2.0** |
| 读报僵尸 | 慢，失去报纸后快 | — |
| 小鬼 / 撑杆系 | 快 | 2.0–3.0 |

⇒ 想要「快」的手册标签就照抄 `快`；想要**真正的快**得去调 `_ground` 的位移速率。

### A4. 手册属性块：照抄内置文案格式

内置僵尸的手册正文是**固定五行属性块**（`Asset/Translate/Translate.zh.translation` 里可直接读到，
键形如 `TOWERDEFENSE_ZOMBIE_*_HANDBOOK_EXPRESTION`），红色统一 `cc241d`：

```
血量：[color=cc241d]1350[/color]
伤害：[color=cc241d]100/s（啃食）[/color]
佩戴：[color=cc241d]---[/color]
类型：[color=cc241d]小型僵尸[/color]
移速：[color=cc241d]快[/color]
```

要点：

- 标签是 **`移速：`**（不是「移动速度」）；植物那边才用 `速度：`。
- 伤害类型用**括号后缀**表达：`100/s（啃食）`。混合伤害写 `100/s（啃食） + 20/1.5s（喷射）`。
- `类型：` 取 `小型僵尸 / 僵尸 / 大型僵尸…`，是**手写文本**，代码里没有枚举映射。
  同体型照抄内置即可（`ZOMBIE_PHYSIQUE.NORMAL` 的内置僵尸写的是「小型僵尸」）。
- 内置用翻译 key，**mod 直接写中文原文**（参照 mod `超级机枪读报僵尸` 就是明文）。
- 属性块里的数值应与实际 config **一致**——建议加闸门强制核对，防「手册写 1350、实际 200」。

### A5. ⚠️ `.tres` 字符串不能有裸换行（会整份资源解析失败）

多行手册文案是这里的头号陷阱。`.tres` 是**单行 `key = value`** 语法：

- ❌ `handbookDescribe = "第一行\n(真换行)第二行"` → Godot **解析不开整个资源**
- ✅ `handbookDescribe = "第一行\n第二行"`（字面 `\` + `n`）

包能打出来、闸门全绿、装机也成功 —— **只有进游戏才会发现读不了**。所以：

```python
def _tres_str(s):
    out = s.replace('\\', '\\\\').replace('"', '\\"')
    return out.replace('\r\n', '\\n').replace('\n', '\\n').replace('\r', '\\n')
```

并加机械闸门（任何 `key = "…"` 行必须以 `"` 收尾且引号数为偶数）：

```python
if ' = "' in line and not line.lstrip().startswith('['):
    if line.count('"') % 2 or not line.rstrip().endswith('"'):
        errs.append('quoted string must be single-line (use \\n escapes)')
```

---

## B0. ★★ 僵尸「怎么才能往前走」—— `_ground` 根运动层

**自建皮肤的僵尸一步都走不了，十次有十次是这个原因。** 排查顺序：

```
僵尸不动 → 皮肤里有没有 _ground 层？ → 没有 ⇒ 就是它
```

### 引擎的唯一前进机制

全仓 `grep TranslateForPhysicsFrame` **只命中 `GroundMoveComponent` 一处**
⇒ 它是唯一会平移角色的组件：

```csharp
// GroundMoveComponent.BatchUpdateValidated（:293）
if (sprite.pause || sprite.blend) { ResetGroundTracking(); return; }   // ★ 混合期间完全不推进
...
Vector2 vector = TryGetGroundPosition(sprite, out groundChanged);      // 取 _ground 层的位姿
if (!groundChanged) { return; }                                        // 没变就不动
Vector2 vector2 = groundPosSave - vector;                              // 本帧位移 = 上帧 − 本帧
groundPosSave = vector;
_moveDelta = new Vector2(vector2.X * _moveScale.X, vector2.Y * _moveScale.Y);
TranslateParent(_moveDelta);                                           // 僵尸跟着平移
```

它按**名字**解析那一层（`groundLayerName = &"_ground"`，来自内置
`GroundMoveComponentDefinition`）：

```csharp
// GroundMoveComponent.cs:222
if (groundLayerName == null || groundLayerName.IsEmpty || ...
    || !parent.sprite.TryResolveLayerIdForRender(groundLayerName, out var layerId))
    _usingGroundLayerSource = false;      // 退化成「没有移动源」⇒ 永远不动
```

而 `TryResolveLayerIdForRender` 就是查 `layerDictionary`：

```csharp
if (... || !_flashAnimeData.layerDictionary.ContainsKey(layerName)) return false;
layerId = _flashAnimeData.layerDictionary[layerName].AsInt32();
return layerId >= 0;
```

⇒ **`layerDictionary` 里必须有 `"_ground"`，且该层每帧要有切片。**

### 内置僵尸的真实数据（实测，用 `probe_ground_motion.py`）

解析 `Asset/Anime/Character/Zombie/Chapter1/Normal/ZombieNormal.tres`：

| 项 | 值 |
|---|---|
| `_ground` 层 id | **0**（`layerDictionary` 里 `"_ground": 0`），`layerVisible[0] = false` ⇒ 不可见 |
| `_ground` 切片出现范围 | 帧 44..503（**Idle 段 0..43 完全没有** ⇒ 站着时不滑） |
| Walk1 (44..90, 47 帧) | `originX` 从 **-10 线性走到 +40** ⇒ **+50 px / 47 帧** |
| 换算 | 12 fps ⇒ **12.77 px/s**（≈ 一格草坪 6.3 s，数值合理） |
| 回跳 | 帧 90→91 从 +40 掉回 -10（Δ = −50），而 **90→91 正好是 Walk1→Walk2 的剪辑边界** |
| 元素形态 | `a=1, b=0`（单位旋转），`originY` 恒 40，`mediaId` 任意 |

### 回跳为什么看不见 —— 铁律

任何锯齿都有一次不连续。`vector2 = 上帧 − 本帧`，所以：

- 正常递增帧：`vector2 = −step` → 往一个方向平移；
- 回跳帧：`vector2 = +D` → **反方向平移一整条**，若不被吃掉就是明显的倒跳一大步。

引擎吃掉它的方式是 `BatchUpdateValidated` 开头那一句：
剪辑切换会置 `sprite.blend` ⇒ 该帧直接 `ResetGroundTracking()`（清基线）⇒ 回跳不产生位移。

**⇒ 铁律：一个剪辑 = 一条完整锯齿，回跳必须落在剪辑边界上。**
放剪辑中间 = 视觉上倒着走一大步。

### 落地配方

1. `.dat` 是「**每层 × 每帧 × 若干元素**」的原生结构 ⇒ 加一层**不需要改格式**，
   只要在层表里多写一层、并在对应帧写元素。
2. 该层元素必须：
   - **`alpha = 0`** —— 只做根运动，不能画出来；
   - **变换 ≠ `Transform2D.Identity`** ——
     `TryGetManagedLayerPositionForRender` 会把 Identity 判为无效；
     所以第一帧的 origin 别取 0（配方：第 k 帧 = `(k+1)·step`）。
3. 只给**真正会走路**的剪辑加：`Walk*` / `Swim` / 「边笑边走」这类；
   `Idle*` / `Eat` / `Death*` **千万别加**，否则站着、啃食、倒下都会自己往前滑。
4. 多帧动画里可以有任意多个**视觉**步态周期，但根运动**只放一条锯齿**
   （步态周期与锯齿周期是两回事）。
5. `.tres` 侧要同步：`layerDictionary` 加 `"_ground"`、`sliceLayerIds` / `sliceKeys`
   （`layerId << 16`）/ `sliceDrawOrders` / `sliceTransforms` / `sliceAlpha` 都要补，
   并让 `frameOffsets` 成为 `frameCounts` 的前缀和（多切片帧）。
6. **`EnsureLaughClip` / 任何每帧强制切剪辑的地方，用 `SetClip` 而不是 `SetAnimation`** ——
   后者会置 `sprite.blend`，而 blend 期间位移整段被丢 ⇒ **每帧都切会彻底走不动**。

### 移速怎么调

`实际移速(px/s) = 每帧位移(px) × frameRate(12) × sprite.timeScale(≈1)`。
拿内置普通僵尸当基准（50/47 px/帧 = 12.77 px/s），按目标倍率缩放即可。
改一个常量就能全局生效（本包是 `nailong_skin.GROUND_PX_PER_FRAME`）。

### 自检（一定要写成闸门）

直接读**落盘的 `.dat` 字节**验（别读内存里的模型 —— 那样「模型对了但写歪了」抓不到）：

1. 存在名为 `_ground` 的层；
2. 该层元素 `alpha == 0`；
3. 每个带根运动的剪辑里位移**单调**；
4. 总位移 == 帧数 × 每帧位移；
5. **负位移只出现在剪辑边界**（否则倒跳）；
6. 不带根运动的剪辑里**完全零切片**。

反向测试直接改 `.dat` 二进制：把层数减 1、把 alpha 改 255、在剪辑中间插一次回跳
—— 三条都必须被抓到。

---

## B. 用 buff 让植物「停止发射」（控场）

### B1. ❌ 先记住这个错误做法

把植物的 `timeScaleInit` 压成 0 —— 它对**动画/移动**有效，对**发射无效**。

原因：有独立发射计时器的组件按 `GetTimerRunScale()` 推进
（`Script/Component/TowerDefense/Character/FireComponent/FireComponent.cs:1683`）：

```csharp
private float GetTimerRunScale()
{
    if (TowerDefenseProcessModeDispatch.IsIZMModeForCurrentPhysicsFrame)
        return (float)(parent?.buff?.GetAttackSpeedMultiplier() ?? 1.0);   // ← timeScale 根本不参与
    if (parent == null) return 0f;
    return (float)(parent.timeScale * (double)timeScale
                   * (parent.buff?.GetAttackSpeedMultiplier() ?? 1.0));
}
```

至少两条路径**绕开 `timeScale`**：

- `FireComponent.cs:1687`（IZM 模式）：只取 buff 倍率；
- `CannonComponent.cs:1344`：`flag ? 1.0 : parent.timeScale`，`flag` 为真直接当 1.0。

⇒ `timeScale = 0` 在这两条路径上**完全无效**，植物照打。

### B2. ✅ 正确做法：挂一个攻速倍率为 0 的 buff

被**所有**发射路径共同乘进去的只有 `buff.GetAttackSpeedMultiplier()`：

| 位置 | 表达式 |
|---|---|
| `FireComponent:1687` | 仅 `GetAttackSpeedMultiplier()` |
| `FireComponent:1693` | `timeScale × timeScale × GetAttackSpeedMultiplier()` |
| `FireComponent:3298` | `timeScale × GetAttackSpeedMultiplier() × …` |
| `CannonComponent:943` | 仅 `GetAttackSpeedMultiplier()` |
| `CannonComponent:1344` | `(flag ? 1.0 : timeScale) × GetAttackSpeedMultiplier()` |
| `CatapultComponent:538` | `… × GetAttackSpeedMultiplier() × …` |

而它只认一个 key（`Script/Component/TowerDefense/Character/BuffComponent/BuffComponent.cs:410`）：

```csharp
public double GetAttackSpeedMultiplier()
{
    if (!(BuffGet("AttackSpeedDown") is TowerDefenseCharacterBuffAttackSpeedDown b))
        return 1.0;
    return b.GetAttackSpeedMultiplier();      // = Math.Max(0.0, timeScaleValue)
}
```

**⇒ 挂 `TowerDefenseCharacterBuffAttackSpeedDown { timeScaleValue = 0, time = N }`，
倍率 = 0 ⇒ 发射计时器永不推进 ⇒ 不挑组件、不挑模式，一颗子弹都打不出来。**

引擎自带先例（照抄它的写法即可）——
`Asset/Anime/Character/Zombie/Boss/EdgarII/Scene/TowerDefenseZombieBossEdgarII.cs:534`：

```csharp
towerDefensePlant.buff.AddBuff(new TowerDefenseCharacterBuffAttackSpeedDown
{
    timeScaleValue = 0.5,
    time = 15.0
});
```

### B3. 实现时的三条纪律

1. **只为「自己挂的」植物摘 buff。**
   `DeleteBuff(key)` 只认 key，所以插件要自己记一份 `List<TowerDefenseCharacter>`。
   挂之前先查 `buff.BuffGet("AttackSpeedDown")`：**若已存在且不是自己挂的 ⇒ 跳过**
   （否则会把游戏原本的减速效果覆盖掉、或回收时把它删掉）。
   埃德加二世的火球就给植物挂 `0.5×/15s`，是真实会撞上的场景。
2. **每次给一份新实例。**
   `BuffComponent.EnterBuff` 是 `buffDictionary[buff.key] = buff`，**不克隆**。
   共用一份会让 `buff.character` / `currentTime` 被后一个植物冲掉。
   （同一株上重复 `AddBuff` 会走 `Refresh` 分支，把 `currentTime` 归零 = 延长窗口，可以用来续时长。）
3. **双保险过期。**
   `AttackSpeedDown.Step()` 在 `currentTime >= time` 时返回 `true`，
   由 `BuffUpdate(delta)` 以 `BuffRemovalReason.Expired` 自动摘除；
   插件同时显式 `DeleteBuff`，并在 `Shutdown()` 里兜底摘一次
   ——**否则关掉 Mod 会留下「植物永远开不了火」**。

### B4. 两个编译期就撞上的事实

- **`BuffComponent` 不是 `GodotObject`**（`CharacterComponentRuntime` 是纯 C# 类）
  ⇒ 不能用 `GodotObject.IsInstanceValid(buff)`，用 `buff == null || buff.IsReleased`。
- `BuffUpdate(delta)` 收到的 `delta` 是**原始物理帧步长，不被 timeScale 缩放**
  ⇒ buff 的 `time` 是真实秒，控场时长可以直接按秒填。

### B5. 想「彻底控场」就两件一起做

- **buff 倍率 0** → 停止发射（真正的修复，全路径有效）；
- **`timeScaleInit = 0`** → 整株僵住、动画停住（观感上的「暂停行动」）。

两者互不干扰：`PrepareCharacterTimeScaleForPhysics` 里 `timeScale = timeScaleInit` 之后
才会跑 `buffComponent.BuffUpdate(delta)`，且 `delta` 不受 `timeScale` 影响
⇒ **`timeScaleInit=0` 不会妨碍 buff 计时/过期**。

### B6. 自建 buff 类型（一般不需要）

`TowerDefenseCharacterBuffConfig.CreateBuffByKey(string)` 是**白名单 switch**，
自定 key 一律返回 `null` ⇒ 想让引擎按 key 认出新 buff，只能改游戏本体。

但引擎**留了外挂口子**：`MayModifyIncomingDamage => GetType().Assembly != typeof(TowerDefenseCharacterBuffConfig).Assembly`
（外部程序集的 buff 被特别对待），以及 `CreateRuntimeInstance()` 的报错文案
「必须 override `CreateRuntimeInstance()`；运行期 buff 不做自动 Resource 复制」。
⇒ 插件可以**自己 `new` 一个派生自 `TowerDefenseCharacterBuffConfig` 的类**再 `AddBuff`，
绕过 `CreateBuffByKey`。**但只要能复用内置 buff 类型（本例就是），就别自建。**

---

## 速查：本节所有行号对应的文件

| 项 | 文件 |
|---|---|
| `attackType` 定义 | `Script/Component/TowerDefense/Character/AttackComponent/AttackComponentDefinition.cs` |
| 组件集合并/覆盖规则 | `Script/Component/Runtime/CharacterComponentSet.cs` |
| 内置僵尸攻击定义 | `Script/Component/TowerDefense/Character/AttackComponent/AttackComponentZombieDefinition.tres` |
| 内置僵尸组件集 | `Prefab/TowerDefense/Character/ComponentSets/TowerDefenseZombieComponentSet.tres` |
| `walkSpeedScale` 字段 | `Prefab/TowerDefense/Character/TowerDefenseZombie.cs:108` |
| 濒死掉血 | `Prefab/TowerDefense/Character/TowerDefenseCharacter.cs`（`hitpointsNearDeath * delta / 3`） |
| 发射计时器缩放 | `Script/Component/TowerDefense/Character/FireComponent/FireComponent.cs:1683/1687/1693/3298` |
| 炮类旁路 | `Script/Component/TowerDefense/Character/CannonComponent/CannonComponent.cs:943/1344` |
| 投掷类 | `Script/Component/TowerDefense/Character/CatapultComponent/CatapultComponent.cs:538` |
| buff 倍率取值 | `Script/Component/TowerDefense/Character/BuffComponent/BuffComponent.cs:410` |
| 攻速 buff 本体 | `Resource/TowerDefense/Character/Buff/TowerDefenseCharacterBuffAttackSpeedDown.cs` |
| buff 挂载/自过期 | `BuffComponent.cs` 的 `AddBuff` / `EnterBuff` / `RemoveBuffCore` / `BuffUpdate` |
| 挂 buff 的官方先例 | `Asset/Anime/Character/Zombie/Boss/EdgarII/Scene/TowerDefenseZombieBossEdgarII.cs:534` |
| 手册文案 | `Asset/Translate/Translate.zh.translation` |
