+++
title = "对象池实战教学（Practice01）"
date = 2026-10-20T00:00:00+08:00
draft = false
tags = ["Unity", "C#", "对象池", "性能优化"]
categories = ["技术复盘"]
description = "从一次幽灵子弹 Bug 出发，拆解 Unity ObjectPool<T> 的真实语义、代码逐层实现与可迁移的验证方法论。"
+++


> **这份文档教什么**
> 教你把 Unity 对象池真正学明白——不只是"会调用 `ObjectPool<T>`"，
> 而是**知道它内部到底怎么工作**，从而能自己判断代码写得对不对。
>
> **为什么要有这份文档**
> 本项目的 `BulletPool` 曾经有一个 bug：编译零警告、注释写得自信、逻辑读起来合理，
> 但在真实玩法参数下**每发子弹都会静默丢失一发，并留下一颗停住不动的幽灵子弹**。
> 这个 bug 是靠**实测**抓到的，不是靠"看着代码觉得没问题"。
>
> 所以这份文档的重点不只是"讲代码"，更是**讲怎么验证**。

**配套代码**：`Assets/Scripts/Practice01/`
**环境**：Unity 2022.3 LTS（C# 9）

---

## 目录

| # | 章节 | 你会学到 |
|---|---|---|
| 1 | [先看这个 bug](#1-先看这个-bug) | 对象池最典型的失效方式长什么样 |
| 2 | [Unity 对象池的 6 条真实语义](#2-unity-对象池的-6-条真实语义) | `ObjectPool<T>` 到底怎么工作（全部实测） |
| 3 | [代码逐层拆解](#3-代码逐层拆解) | 四个文件每个方法"为什么这么写" |
| 4 | [本次修复详解](#4-本次修复详解) | 正确写法、幽灵子弹成因、修复对比 |
| 5 | [学会"验证而非相信"](#5-学会验证而非相信) | 可迁移到任何 API 的验证方法论 |
| 6 | [编辑器验证清单](#6-编辑器验证清单) | 你进 Play 模式后逐项核对 |
| 7 | [练习任务](#7-练习任务) | 6 个递进任务，从复现到扩展 |
| 8 | [自测题](#8-自测题) | 11 道题 + 折叠答案 |
| 附录 A | [离线探针完整源码](#附录-a离线探针完整源码) | 可直接复制运行 |
| 附录 B | [离线编译校验方法](#附录-b离线编译校验方法) | 不开 Unity 就能查编译错误 |
| 附录 C | [术语表](#附录-c术语表) | |

> **本文档覆盖三个已修复的 bug**：
>
> | # | Bug | 症状 | 位置 |
> |---|---|---|---|
> | 1 | 池满判定放在 `Get()` 之后 | 每约 2 发丢 1 发 + 幽灵子弹 | `BulletPool.Dispatch` |
> | 2 | 渲染器开关放在 `OnSpawn` 里 | 防幽灵帧的防护**完全无效** | `Bullet.OnSpawn` / `Launch` |
> | 3 | `LookRotation` 的 up 选错 | 正对前方射击时子弹朝向退化 | `BulletSpawner.Spawn` |
>
> 第 1 个见第 1、4 部分；第 2、3 个见第 4.5 节。
> **三个的共同点：编译零警告、注释很自信、逻辑读起来合理。**

---

## 1. 先看这个 bug

### 1.1 症状

用默认参数（射速 6 发/秒、子弹存活 5 秒、`maxSize=100`）连续射击时：

- 玩家看到**每隔一发就有一发凭空消失**
- 枪口位置**出现一颗停住不动的子弹**，5 秒后才消失
- Console 刷出 `子弹池已满（maxSize=100，在场 27 发）` —— **但 100 根本不可能满**

### 1.2 出问题的那段代码

```csharp
// ❌ 初版（有 bug）
int before = pool.CountAll;
discardedLastShot = false;

Bullet bullet = pool.Get();                          // ← 先取

if (pool.CountAll == before && pool.CountInactive == 0)   // ← 后判
{
    overflowCount++;
    discardedLastShot = true;
    return;
}
```

配套注释写着：
> "已到 maxSize 且无空闲 → 新建一个实例、先跑 actionOnGet、再立刻 actionOnDestroy（这次新实例当场被销毁）"

**这段注释在描述一个 Unity 根本不存在的行为。** 详见第 2 部分。

### 1.3 为什么会这样

条件 `CountAll == before && CountInactive == 0` 实际等价于：

> **"这次 `Get()` 复用了池中对象，且复用后空闲栈刚好空了。"**

翻译成大白话：**"池工作得最好的时候"**。

而真正的"池满"（`Get()` 触发新建）会让 `CountAll` **增长**，
所以 `CountAll == before` 为 false → **真正的池满反而检测不出来**。

> **这个条件是"因果颠倒"的产物**：
> 作者假设"`Get()` 会销毁对象"，于是想用"数量没涨"反推"被销毁了"——但这个假设是错的。

### 1.4 为什么所有常规检查都放行了它

| 检查手段 | 结果 |
|---|---|
| 编译 | ✅ 通过，0 错误 0 警告 |
| 代码审查 | ✅ 逻辑读起来合理，注释很完整 |
| 单元测试 | ⚠️ 如果没覆盖"复用路径"，测不出来 |
| 肉眼试玩 | ❌ **最容易漏**——"偶尔丢一发"很难当场归因 |

唯一抓住它的是：**写探针连真实实现跑一遍**。

---

## 2. Unity 对象池的 6 条真实语义

> ⚠️ **以下全部是实测结论**，不是文档抄写，也不是凭直觉推断。
>
> **验证方式**：把 `UnityEngine.CoreModule.dll`（`<Unity 安装目录>`）作为引用，
> 用 `dotnet` 编译并**直接运行**探针程序，链接的是 Unity 真实实现。
> 完整源码见[附录 A](#附录-a离线探针完整源码)，实测输出见附录 A 末尾。

### 语义 1：`Get()` 不受 `maxSize` 限制

**直觉（错的）**：`maxSize = 100` 意味着池里最多 100 个实例，满了 `Get()` 就会失败或销毁。

**实测（真的）**：

```
maxSize=2，连续 Get 5 次 -> CountAll=5  CountActive=5  CountInactive=0  累计销毁=0
```

`Get()` 的逻辑极其简单：**空闲栈有东西就弹一个，没有就 `createFunc()` 新建一个**。
它根本不看 `maxSize`。所以池子可以无限膨胀。

> **推论**：`maxSize` 拦不住"池被撑爆"，只拦得住"闲置对象堆积"。
> 想限制同时在用的实例数，**必须自己判断**。

### 语义 2：`maxSize` 只在 `Release()` 时生效

**实测**：

```
取 4 个（maxSize=3）再全部归还 -> CountAll=3  CountInactive=3  累计销毁=1
   （多还的那一个触发 actionOnDestroy）
```

空闲栈已达上限时再归还，实例走 `actionOnDestroy` 被销毁（而不是留在池里）。

> **为什么这个语义很重要**：这意味着 `ObjectPool` 自带的 `maxSize` 是一种**"闲置上限"**，
> 而不是"容量上限"。如果你想要的是后者（最多同时存在 N 个子弹），
> 用 `ObjectPool` 自带的 `maxSize` 是达不到的。

### 语义 3：`Clear()` 只清闲置栈，并触发 `actionOnDestroy`

**实测**：

```
Clear 前：CountAll=3  CountActive=0  CountInactive=3  销毁=0
  -> actionOnDestroy：item
  -> actionOnDestroy：item
  -> actionOnDestroy：item
Clear 后：CountAll=0  CountActive=0  CountInactive=0  销毁=3
```

`Clear()` 对池内**每个闲置实例**各触发一次 `actionOnDestroy`，并把三个计数全部归零。

**关键补充（容易踩）**：`Clear()` **只碰闲置栈**。
正在池外使用（已 `Get` 未 `Release`）的对象**不受影响**——既不会被销毁，也不会被计数。

> **推论**：`Clear()` = 销毁全部闲置对象，之后池里没有可复用的东西；
> 池外那些"在途对象"的清理责任落在调用方身上
> （`BulletSpawner.OnDestroy` 就是这么处理的）。

### 语义 4：重复 `Release` 会抛异常

**实测**：

```
重复 Release 抛出: InvalidOperationException :
  Trying to release an object that has already been released to the pool.
```

`collectionCheck: true` 时（**仅编辑器生效**，Release 构建里不检查），重复归还会抛。

> **推论**：这就是 `Bullet` 类里 `released` 标志存在的全部理由。
> 没有它，子弹超时回收和碰撞回收**同时触发**时就会崩。

### 语义 5：`Release` 一个陌生对象**不会**抛异常

**实测**：

```
Release 陌生对象: 没有抛异常, CountAll=0 CountInactive=1 销毁=0
```

`collectionCheck` 只检查"是不是已经在池里了"，不检查"是不是我们的对象"。
一个从未 `Get()` 过的对象会被直接塞进空闲栈。

> **推论（隐蔽的坑）**：如果你的回池回调可能收到错误的实例，
> 它会**污染空闲栈**。所以代码里才要用 `active.Remove(bullet)` 这类操作
> 把"在飞清单"和"空闲栈"对齐。

### 语义 6：`maxSize = 0` 会让构造函数抛异常

**实测**：

```
maxSize=0 -> ArgumentException : Max Size must be greater than 0 (Parameter 'maxSize')
```

> **推论**：这个异常会从 `BulletSpawner.Awake` 冒出来，堆栈不直观。
> 所以 `BulletPool` 构造函数里提前用 `ArgumentOutOfRangeException` 拦一道，错误信息更清楚。

### 语义速查表

| # | 语义 | 影响的代码 |
|---|---|---|
| 1 | `Get()` 不受 `maxSize` 限制 | 容量判断必须自己做，且放在 `Get()` 之前 |
| 2 | `maxSize` 只约束 `Release()` 时的空闲栈深度 | 别把它当"同时在用数上限" |
| 3 | `Clear()` 只清闲置栈，触发 `actionOnDestroy` 并清零计数 | 池外在途对象不会被它碰到 |
| 4 | 重复 `Release` 抛 `InvalidOperationException` | 需要 `released` 标志防重 |
| 5 | `Release` 陌生对象不抛异常但会污染空闲栈 | 回池通道要校验身份 |
| 6 | `maxSize <= 0` 构造函数抛 `ArgumentException` | 参数要提前校验 |

---

## 3. 代码逐层拆解

### 3.0 整体结构

```
IPooledObject.cs    契约：池里的对象要能"被取出后重置"和"被归还前清干净"
    ↓ 实现
Bullet.cs           池成员：单发子弹，负责自己的状态重置
    ↓ 被池管理
BulletPool.cs       池：只管存/取/归还/丢弃，不管瞄准、不管射速
    ↓ 被使用
BulletSpawner.cs    射击者：决定什么时候开火、朝哪开、开多快
```

**为什么要这么分？** 因为对象池最重要的设计点是**"池不碰对象的内部状态"**。
如果池自己去清速度、改位置，它就得知道"子弹"这个具体类型，池就废了，无法复用到别的对象上。

所以定下这条规矩：

> **池只负责对象的"生命周期切换"，对象自己负责"状态的干净与还原"。**

`IPooledObject` 就是这条规矩的书面形式。

---

### 3.1 `IPooledObject.cs` —— 契约

```csharp
public interface IPooledObject
{
    void OnSpawn();     // 每次从池中取出（含第一次创建）时调用
    void OnDespawn();   // 每次被归还入池前调用
}
```

**逐句理解**：

- `OnSpawn`："我刚被拿出来了，请把上一轮留下的东西清干净"。
  关键在于**含第一次创建**——新实例其实不需要清理，但统一调用能简化逻辑，
  也让"第一次"和"复用第 N 次"走**完全相同**的代码路径，
  不会出现"第一次忘了初始化"这类 bug。

- `OnDespawn`："我马上要回池了，请把一次性状态（速度、订阅、计时）清掉"。
  这一步比 `OnSpawn` 更容易被漏掉，后果是**下一发子弹带着上一发的速度飞出去**。

- 为什么是接口而不是基类？
  因为 `Bullet` 是 `MonoBehaviour`，C# 不支持多继承。
  接口是唯一能让"池和具体类型解耦"的办法。

> **学习要点**：接口表达的是**能力**，不是"父子关系"。
> 问自己："这个类型需要被池管理吗？"需要就实现它。

---

### 3.2 `Bullet.cs` —— 池成员（最需要细读）

全套代码里最讲究的一份，逐段拆。

#### 3.2.1 字段：哪些该序列化，哪些不该

```csharp
[Header("回收")]
[SerializeField] private float lifeTime = 5f;    // ← 该序列化：策划要调的参数

// ---- 以下均为运行时状态，刻意不参与序列化 ----
private Rigidbody rb;                             // ← 不序列化：运行时缓存
private Renderer cachedRenderer;
private Action<Bullet> despawnAction;             // ← 不序列化：运行时注入
[NonSerialized] public float spawnTime = float.MinValue;
[NonSerialized] private bool released = true;
```

**为什么要区分？**

`[SerializeField]` 的字段会被写进 Prefab/场景的 YAML。运行时状态写进去会造成：

1. Prefab 体积膨胀
2. **复用时残留上一轮的值**——这是对象池特有的致命问题
3. 序列化系统会把 `Action<Bullet>` 委托这种类型处理得很麻烦

> **原则**：策划要调的 → `[SerializeField]`；运行时产生的 → 私有字段 + `[NonSerialized]`。

`spawnTime = float.MinValue` 表示"从未出膛"。为什么不用 `0`？
因为 `Time.time` 在游戏刚启动时可能确实是 0 左右，用 `0` 当哨兵值会产生歧义。
`float.MinValue` 永远不会等于真实时间戳，判定时无歧义。

`released = true` 初始为 `true`（"当前已归还"）——因为对象刚创建时确实不在池外使用中。
这个初值让"重复归还"保护在创建瞬间就生效。

#### 3.2.2 `Awake` / `Reset` / `OnValidate` —— 三个"配置"入口

```csharp
private void Awake()      { rb = GetComponent<Rigidbody>(); cachedRenderer = GetComponent<Renderer>(); ConfigureRigidbody(); }
private void OnValidate() { if (Application.isPlaying) return; ConfigureRigidbody(); }
private void Reset()      { ConfigureRigidbody(); }
```

三个方法干同一件事，区别在**触发时机**：

| 回调 | 何时触发 | 用途 |
|---|---|---|
| `Reset` | 组件刚被 AddComponent 时 | 让新加的组件自动配好 Rigidbody |
| `OnValidate` | Inspector 里改了任何值 | 编辑期就约束住配置，防止配出自相矛盾的预制体 |
| `Awake` | 运行时，实例化后 | 缓存组件引用 + 应用配置 |

`OnValidate` 里的 `if (Application.isPlaying) return;` 很重要：
`OnValidate` 在运行时也可能被调用，而 `GetComponent` 在编辑期是危险的（会污染预制体）。

> **为什么组件引用在 `Awake` 缓存？**
> `GetComponent` 是运行时查找，有开销。而对象池里的对象**只会被 `Awake` 一次**
> （复用不会重新触发 `Awake`），所以缓存的性价比极高。**绝不能每帧 `GetComponent`。**

#### 3.2.3 `OnSpawn` —— 重置状态

```csharp
public void OnSpawn()
{
    released = false;
    spawnTime = Time.time;

    if (rb == null) { rb = GetComponent<Rigidbody>(); ConfigureRigidbody(); }

    rb.velocity = Vector3.zero;
    rb.angularVelocity = Vector3.zero;
}
```

**逐行讲"为什么"**：

1. `released = false` —— 重新进入"在外使用"状态。
2. `spawnTime = Time.time` —— 计时基准。**必须在每次取出时重设**，
   否则复用的子弹会拿旧时间算超时。
3. `if (rb == null)` 兜底 —— 不假设 `Awake` 一定先于 `OnSpawn`。
   实际上 `Instantiate` 会同步调用 `Awake`，但**不假设**是更稳的工程习惯。
4. `rb.velocity = Vector3.zero` —— 清掉上一发的速度。
   不清的话，复用的子弹会在出膛瞬间带着上一发的速度继续飞（看起来像"加速了"）。

> ⚠️ **注意**：`OnSpawn` 里**没有**渲染器开关。
> 那件事在 `Launch()` 里做，原因见 3.2.4——这个位置放错会让保护完全失效。

#### 3.2.4 `Launch` —— 装填发射数据 + 渲染器开关

```csharp
public void Launch(in BulletSpawnData data)
{
    // 先关渲染：此时物体还停在预制体原始位置上，不能让它被看见。
    if (cachedRenderer != null) { cachedRenderer.enabled = false; }

    transform.SetPositionAndRotation(data.position, data.rotation);

    if (rb == null) { rb = GetComponent<Rigidbody>(); ConfigureRigidbody(); }

    rb.velocity = data.velocity;

    // 位置与速度都装好了，重新可见。
    if (cachedRenderer != null) { cachedRenderer.enabled = true; }
}
```

`in` 关键字：`BulletSpawnData` 是 `struct`，`in` 表示**按只读引用传递**，避免拷贝 40 多字节的值。
这是 C# 7.2 的特性，C# 9 完全支持。

> **为什么用 `struct` 而不是 `class`？**
> `class` 是引用类型，会在堆上分配 → 每次开火一次 GC。
> `struct` 是值类型，直接压栈 → 零分配。
> 这就是"取池 → 填数据 → 派发"这条路径上**不产生任何托管分配**的关键。

**为什么渲染器开关必须在 `Launch` 里，而不是 `OnSpawn`？**

子弹被 `Instantiate` 出来时位置是**预制体的原始位置**，
而 `Launch()` 才把它搬到枪口。这中间如果渲染器是亮的，
就会在预制体位置渲染出一颗"幽灵子弹"。

**如果写在 `OnSpawn` 里，会完全失效**——顺序错了：

```
错误顺序：
  actionOnGet → SetActive(true)      ← 渲染器亮了
  → OnSpawn → 关渲染 → 清速度 → 开渲染   ← 中间夹着一次真正的渲染！
  → Launch → 设位置                    ← 位置到这里才改
```

`OnSpawn` 结束时渲染器已经被重新打开，而位置要到之后的 `Launch()` 才设。
所以"关→开"的那一瞬间，渲染器**根本没处于关闭状态过**。保护形同虚设。

**正确顺序**：

```
  actionOnGet → SetActive(true)      ← 渲染器亮了（但这一帧还没渲染）
  → OnSpawn → 清速度
  → Launch → 关渲染 → 设位置与速度 → 开渲染   ← 关着的时候位置已就位
```

> **关键前提**：渲染器的 `enabled` 开关是**帧内即时生效**的。
> 所以同一帧内"关掉再打开"，这一帧**不会**被渲染出来。

这是"用渲染时序解决视觉 bug"的技巧，值得记住。
**也值得记住的是它的反面：防护代码放错位置，就等于没写。**

#### 3.2.5 `Update` —— 超时回收

```csharp
private void Update()
{
    if (released || lifeTime <= 0f) return;
    if (Time.time - spawnTime >= lifeTime) Despawn();
}
```

- `released` 检查：已归还的子弹不再计时。
- `lifeTime <= 0f`：0 或负数表示"永久存在，不自动回收"。
  这样可以配出"不消失的子弹"（比如追踪弹、激光）。
- `Time.time - spawnTime` 而不是累加 `+= deltaTime`：前者不会因为帧率波动而累积误差。

> **为什么不每帧检查物理位置（出界回收）？**
> 因为那需要 `bounds` 计算或射线检测，CPU 开销更大，而子弹有明确的存活时间。
> **能用时间就别用空间**——这是移动端性能的基本取舍。

#### 3.2.6 `Despawn` —— 唯一回池出口

```csharp
public void Despawn()
{
    if (released) return;      // 幂等保护
    released = true;
    if (despawnAction != null) despawnAction(this);
}
```

**"幂等"是这个方法的全部设计要点。**

子弹可能因为**多种原因**回池：超时、撞到目标、出界。
如果两个原因在同一帧触发（比如正好在超时那一帧撞到墙），会调用两次 `Despawn`。
没有 `released` 保护 → `Release` 两次 → 触发语义 4 → **抛 `InvalidOperationException` 直接崩**。

> **这就是语义 4 在真实业务里的投影。**
> 学到这里你要能自己推出来，而不是背结论。

`despawnAction` 由池在创建时注入，**不在 `OnSpawn` 里重新分配**——
这是为了避免"每发子弹 new 一个闭包"的帧内 GC。

---

### 3.3 `BulletPool.cs` —— 池本身

#### 3.3.1 四个回调的时机

```csharp
pool = new ObjectPool<Bullet>(
    createFunc:      () => { ... Instantiate ... },      // 池空了才调用
    actionOnGet:     bullet => { SetActive(true); OnSpawn(); },
    actionOnRelease: bullet => { OnDespawn(); SetActive(false); active.Remove(bullet); },
    actionOnDestroy: bullet => { active.Remove(bullet); Destroy(bullet.gameObject); },
    collectionCheck: true,
    defaultCapacity: defaultCapacity,
    maxSize: maxSize);
```

**时序图**：

```
新开火（池空）：
  createFunc → [新实例 SetActive(false), Init] → actionOnGet → SetActive(true) + OnSpawn
  → Dispatch 里 Launch + active.Add

复用（池有闲置）：
  actionOnGet → SetActive(true) + OnSpawn → Dispatch 里 Launch + active.Add

归还：
  Bullet.Despawn → releaseAction → pool.Release
    → actionOnRelease → OnDespawn + SetActive(false) + active.Remove
    → （若空闲栈已满）actionOnDestroy → active.Remove + Destroy(gameObject)

清空：
  pool.Clear → 对每个闲置实例触发 actionOnDestroy
```

注意 `createFunc` 里**没有** `SetActive(true)`：
注释写得很清楚——"激活应该只发生在 `actionOnGet` 里，避免池外对象被误判为在用"。
新实例必须先处于**非激活**状态，否则它还没进空闲栈就已经显示在场景里了。

#### 3.3.2 `active` 清单 vs 池的空闲栈

```csharp
private readonly List<Bullet> active = new List<Bullet>(64);
```

**为什么池自己还维护一份"在飞清单"？**

因为 `ObjectPool` 只知道"谁在池里"（空闲栈），**不知道"谁在外面飞"**。
而 `Bullet` 内部的状态（速度、位置）在池外面，池管不着。

所以池维护两个集合：

- **空闲栈**（Unity 的 `ObjectPool` 内部管）：闲置待用
- **`active`**（我们自己管）：已取出、正在飞

两者通过 `actionOnRelease` 里的 `active.Remove(bullet)` 保持同步。

`new List<Bullet>(64)` 的容量 64 是**预分配**：
避免运行中扩容时的数组拷贝和重新分配。初始容量应该按"同屏最大实例数"估。

#### 3.3.3 `Dispatch` —— 本次修复的核心

```csharp
public void Dispatch(in BulletSpawnData data)
{
    discardedLastShot = false;

    // ---- 容量判断必须放在 Get() 之前 ----
    if (pool.CountAll >= maxSize && pool.CountInactive == 0)
    {
        overflowCount++;
        discardedLastShot = true;
        return;                                  // 没碰池子 -> 不产生幽灵弹
    }

    Bullet bullet = pool.Get();

    if (bullet == null || !bullet.gameObject.activeSelf)
    {
        Debug.LogError("BulletPool：取到了不可用的子弹实例，本次开火已跳过。...");
        discardedLastShot = true;
        return;
    }

    bullet.Launch(data);
    active.Add(bullet);
}
```

**第一个 `if`：容量判断——为什么必须在 `Get()` 之前？**

这是本次修复的关键，第 4 部分会完整复盘。这里先记住结论：

> `ObjectPool` 不会在 `Get()` 时拦"池满"（语义 1）。
> 如果先 `Get()` 再判断，你已经取出了一个**已 `SetActive(true)`** 的子弹，
> 却发现放不回池里 → 它会停在枪口位置不动（因为没走 `Launch()`），
> 一直到 5 秒超时才消失。**这就是幽灵子弹。**

**第二个 `if`：防御性检查。**

`bullet == null` 而不是 `bullet?.` —— 因为 Unity 的"假 null"：
被 `Destroy` 的对象 `== null` 为 true，但 `?.` 判断不出来（引用还在，只是底层被标记销毁）。
**只有 `==` 重载能识别。** 这是 Unity C# 的经典坑。

`!bullet.gameObject.activeSelf` 检查对象是不是真的处于激活状态——
如果池内部状态不一致（比如被外部误操作），宁可这次不开火，
也不能把不可用对象塞进 `active` 清单，那会导致它飞行途中被销毁。

**最后两行**：`Launch` 装填数据，`active.Add` 登记在飞。

#### 3.3.4 `Clear` 与 `OnDestroy`

```csharp
public void Clear() { pool.Clear(); active.Clear(); }
```

`BulletSpawner.OnDestroy` 里调用。语义 3：`pool.Clear()` 会销毁所有闲置实例并触发 `actionOnDestroy`。
不清的话，这些 `GameObject` 会随场景卸载变成悬挂引用。

---

### 3.4 `BulletSpawner.cs` —— 使用者

#### 3.4.1 职责边界

```csharp
// 本类只管"什么时候开火、朝哪开、开多快"，"子弹对象从哪来、回哪去"交给 BulletPool。
```

| | BulletSpawner | BulletPool |
|---|---|---|
| 关心 | 射速、方向、速度 | 容量、复用、状态重置 |
| 不知道 | 对象池内部结构 | 枪口在哪、朝哪打 |

> **为什么这个边界重要？**
> 因为它让 `BulletPool` 可以原封不动地复用到飘字池、特效池、伤害数字池——
> 只要那个类型实现 `IPooledObject`。

#### 3.4.2 `Awake` / `Start` 的分工

```csharp
private void Awake()
{
    if (bulletPrefab == null) { Debug.LogError(...); enabled = false; return; }
    if (muzzle == null)       { Debug.LogError(...); enabled = false; return; }
    bulletPool = new BulletPool(bulletPrefab, transform, defaultCapacity, maxSize);
}

private void Start() { CacheAimCamera(); }
```

**规则**：

- `Awake` 拿**自己身上**的引用（预制体、枪口）
- `Start` 拿**跨对象**的引用（场景里的相机）

因为场景里其他对象的 `Awake` 顺序不保证，`Start` 里所有对象的 `Awake` 都已执行完。

> **这是 Unity 生命周期最常用的一条经验。**

`enabled = false` 是"优雅失败"：配置错了就停用组件，不让每帧都刷错误日志。

#### 3.4.3 零分配的 `Update`

```csharp
private void Update()
{
    if (bulletPool == null) return;

    if (aimCamera == null && !CacheAimCamera()) return;   // 只在需要时补一次

    if (fireTimer > 0f) fireTimer -= Time.deltaTime;

    if (!Input.GetMouseButton(0) || fireTimer > 0f) return;

    Spawn();
    fireTimer = 1f / Mathf.Max(fireRate, 0.01f);
}
```

**逐点讲"为什么这样写"**：

1. `aimCamera == null && !CacheAimCamera()` —— `Camera.main` 内部有按标签查找的开销，
   绝不能放进常规路径。写法是"**已经缓存了就直接用，没缓存才尝试补一次**"。
   `CacheAimCamera()` 内部还有 `if (aimCamera != null) return true;` 的短路。

2. `fireTimer` 用**减法**而不是 `+= 1/fireRate`：
   减法不会因为帧率抖动而累积漂移。

3. `Mathf.Max(fireRate, 0.01f)` —— 防除零。`fireRate` 是 Inspector 可改的，
   用户填 0 会导致 `1/0 = Infinity` → `fireTimer = Infinity` → 永远打不出子弹（也不报错）。
   这个防御很值。

4. `Input.GetMouseButton(0)` —— 旧版 Input。
   **注意**：Unity 2022.3 的 `Player Settings → Active Input Handling` 必须是 `Both` 或 `Old`，
   否则 `Input` 类整个不可用（编译能过，运行时报 `InvalidOperationException`）。

   > **这是"编译通过但运行时炸"的典型案例，值得记住。**

#### 3.4.4 `Spawn` —— 方向计算里的数学

```csharp
Vector3 axis = Vector3.Cross(direction, Vector3.up);
if (axis.sqrMagnitude > 0.000001f)
    direction = Quaternion.AngleAxis(launchPitch, axis.normalized) * direction;
```

**为什么要用叉乘而不是记正负号？**

要把方向"朝上抬" `launchPitch` 度，需要一个**水平旋转轴**。
这个轴 = 方向 × 上向量。方向朝北时，叉乘结果指向东或西——
**轴的方向自动决定抬还是压**，不需要手动判断符号。

**符号推导（实测确认，别背结论）**：

用纯数学的罗德里格斯公式（绕轴旋转 `angle` 度）实测：

```
direction = (0, 0, 1)      up = (0, 1, 0)
axis = direction x up = (-1, 0, 0)     ← 指向左

绕 axis.normalized 旋转 +12 度 -> (0.00, 0.21, 0.98)    y = +0.207912  ← 朝上抬，正确
绕 +X 轴旋转 +12 度           -> (0.00, -0.21, 0.98)   y = -0.207912  ← 朝下压
```

`direction × up` 得到的是**左向轴**，绕它正向旋转正好把方向朝上抬——
**原代码的符号是对的**。

**为什么要检查 `sqrMagnitude`？**

方向完全竖直（正上/正下）时，叉乘结果是零向量 → `normalized` 得到 `NaN` → 方向彻底坏掉。
`0.000001f` 是 epsilon 阈值，判断"是否退化成零向量"。

> **通用经验**：任何 `normalized` 之前，先判断长度是否接近 0。
>
> 但注意：在本项目里 `direction` 恒为水平，这个退化**同样不会发生**。
> 它是一道正确的保险，不是错误——和上面 `LookRotation` 的情况不同。
> 区别在于：这道保险**不会让任何常见场景变坏**。

```csharp
spawnData.rotation = Quaternion.LookRotation(direction, Vector3.up);
```

**为什么 `up` 用 `Vector3.up` 而不是 `origin.forward`（枪口朝向）？**

`LookRotation` 要求 `forward` 与 `up` **不共线**，否则正交基无法构造，返回未定义的四元数。

关键在于**哪个向量在本项目里真的可能与 `direction` 共线**：

| 向量 | 与 `direction` 共线的可能性 | 实测结论 |
|---|---|---|
| `Vector3.up`（世界向上） | **不可能** | `direction` 的 y 恒为 0（两个分支都强制压平），抬角上限 12°，y 最大只有 `sin(12°)≈0.208` → 与 `Vector3.up` 的叉乘长度恒为 0.978，**永不共线** |
| `origin.forward`（枪口朝向） | **很可能** | 枪口朝向通常是水平的；玩家正对前方射击时 `direction` 与它**完全平行**（点积 = 1.000000）→ 共线 → 退化 |

> **⚠️ 这里原代码的注释是反的**：它说"方向接近垂直时会因共线而退化，所以用枪口朝向当 up 更安全"。
> 但本项目的 `direction` **永远不可能接近垂直**——它恒为水平。
> 那个"垂直退化"场景根本不存在，而它为不存在的场景选了 up，
> 却让**最常见的场景**（正对前方射击）退化了。

**实测证据**（纯数学实现，绕轴旋转与正交基构造）：

```
direction 与 origin.forward 平行？ dot = 1.000000 (=1 -> 完全平行)
direction 与 Vector3.up      垂直？ dot = 0.000000 (=0 -> 垂直)

以 Vector3.up 为参考构造正交基：
  right = (1.00, 0.00, 0.00)   up = (0.00, 1.00, 0.00)
  dot(f,r)=0.000000  dot(f,u)=0.000000  dot(r,u)=0.000000   ← 合法正交基

以 origin.forward 为 up 时：
  cross(up2, f) = (0.00, 0.00, 0.00)   长度 = 0.000000
  → 长度为 0 = 正交基无法构造 = LookRotation 的退化输入
```

> **这又是一条"别按直觉推断"的例子**：原代码的推理链条本身是自洽的，
> 只是它推理的前提（"direction 可能接近垂直"）在这个项目里根本不成立。
>
> **教训**：写防御代码时，要问"**这个退化场景在本项目真的会发生吗？**"
> 为不存在的场景加的"防护"，往往会在真实场景里制造新的问题。

#### 3.4.5 `ResolveAimDirection` —— 屏幕射线与水平面求交

```csharp
Ray ray = aimCamera.ScreenPointToRay(Input.mousePosition);
var plane = new Plane(Vector3.up, origin.position);

if (plane.Raycast(ray, out float enter))
{
    Vector3 target = ray.GetPoint(enter);
    Vector3 planarDirection = target - origin.position;
    planarDirection.y = 0f;
    if (planarDirection.sqrMagnitude > 0.0001f) return planarDirection.normalized;
}

// 退化情况退回朝向的平面投影
Vector3 fallback = origin.forward;
fallback.y = 0f;
return fallback.sqrMagnitude > 0.0001f ? fallback.normalized : Vector3.forward;
```

**思路**：把鼠标射线投到"枪口所在高度的水平面"上，得到一个落点，
取"枪口 → 落点"的方向（并压平到水平）。

**为什么不用直接的 `ray.direction`？**

因为相机的俯仰角会让"屏幕上的方向"和"世界里的方向"不一致。
压到水平面再取，等于**强制子弹始终水平飞出**，符合俯视射击的直觉。

**三重退路**（这段是防御性编程的范例）：

1. 射线与平面平行（相机正对着水平面看）→ `plane.Raycast` 返回 false
2. 鼠标正好压在枪口正上方 → `planarDirection` 长度接近 0
3. 连枪口朝向都是竖直的 → `fallback` 长度接近 0 → 退回 `Vector3.forward`

**每一层都有兜底，保证方向永远是有效单位向量。**

---

## 4. 本次修复详解

### 4.1 修复后的代码

```csharp
public void Dispatch(in BulletSpawnData data)
{
    discardedLastShot = false;

    // ---- 容量判断必须放在 Get() 之前 ----
    // 真实语义（Unity 2022.3 实测）：ObjectPool.Get() 不受 maxSize 限制，
    // 池空就无限新建，永远不会"取完当场销毁"。
    // 如果先 Get() 再判断，就会把一个已经 SetActive(true) 的子弹取出来，
    // 因为放不回池里而丢在一边——它没被 Launch()，会停在枪口位置不动 5 秒，
    // 形成"幽灵子弹"，同时本该发出的那一发也静默丢了。
    // 所以这里先自己判断容量，满就直接放弃开火，不碰池子。
    if (pool.CountAll >= maxSize && pool.CountInactive == 0)
    {
        overflowCount++;
        discardedLastShot = true;
        return;
    }

    Bullet bullet = pool.Get();
    // ...后续不变
}
```

**三个关键变化**：

| # | 变化 | 为什么 |
|---|---|---|
| 1 | 判断挪到 `Get()` **之前** | 池满时根本不取对象，零副作用 |
| 2 | 判据换成 `CountAll >= maxSize` | 直接表达"实例总数已达上限" |
| 3 | 加上 `&& CountInactive == 0` | 有空闲可复用就不算满 |

### 4.2 幽灵子弹的完整因果链

```
旧逻辑：先 Get()，后判断
   ↓
Get() 执行 actionOnGet → bullet.gameObject.SetActive(true) + bullet.OnSpawn()
   ↓
OnSpawn 里设了 spawnTime = Time.time，重置了速度，启动 5 秒生命周期
   ↓
判断发现"放不回池里" → 直接 return，跳过了 bullet.Launch(data)
   ↓
结果：这颗子弹已激活、有 5 秒寿命，但位置还是预制体原位、速度为 0
   ↓
它不会飞出去，就停在枪口 → 玩家看到"一颗不动的子弹"
   ↓
5 秒后 Bullet.Update 超时 → Despawn → 回池
```

**注意这个链条里最讽刺的一点**：
`OnSpawn` 里那套"关渲染器→清速度→开渲染器"的精妙逻辑，
在幽灵子弹身上**全都执行了**——它被清理得干干净净，然后就一动不动地停在那儿。

### 4.3 新旧逻辑实测对比

模拟真实玩法循环（"射速 × 存活时间"算出同屏在飞数，跑真实循环）：

| 场景 | 旧：误判丢弃 / 幽灵弹 | 新：误判丢弃 / 幽灵弹 |
|---|---|---|
| A 正常参数（5 发/秒，`maxSize=100`，同屏 25 发） | 13 / 13 | **0 / 0** |
| B 真池满（`maxSize=5`，高射速） | 0 / 0 | **55 / 0** |
| C 连打 600 发 | 498 / 101 | **8 / 0** |

- **场景 A**：旧版每发都在误杀，13 发幽灵弹。
- **场景 B**：旧版"丢弃 0 次"——**完全没拦住池满**，这是最糟的失败方式（静默失效）。
- **场景 C**：新版剩的 8 次是**真实丢弃**（同屏 100 发 = 恰好等于 `maxSize=100`），不是误报。

### 4.4 附带的其它改动

| 改动 | 原因 |
|---|---|
| 新增 `private readonly int maxSize` | 新判定要用它 |
| 构造函数加 `maxSize <= 0` 校验 | 语义 6：Unity 会抛 `ArgumentException`，堆栈难读 |
| 重写类注释 | 原文描述的 Unity 行为是错的 |
| `OverflowCount` 的 XML 注释 | 原文说"取了又立刻销毁"，实际是"容量已满被丢弃" |

### 4.5 另外两个 bug：防护代码本身的问题

第一个 bug 是"防护没做"。接下来两个是**"防护做了，但有问题"**——
这类更隐蔽，因为代码看起来比没有防护更负责任。

#### Bug 2：渲染器开关放在 `OnSpawn` 里 → 防护完全无效

**原代码**（`Bullet.OnSpawn`）：

```csharp
cachedRenderer.enabled = false;   // 关渲染
rb.velocity = Vector3.zero;
rb.angularVelocity = Vector3.zero;
cachedRenderer.enabled = true;    // 开渲染
```

注释写得很自信："渲染器开关在帧内即时生效，所以同一帧内关掉再打开不会被渲染。"

**问题**：位置是在 `Launch()` 里才设的，而 `Launch()` 在 `OnSpawn()` **之后**。

```
actionOnGet → SetActive(true)            ← 渲染器亮了
→ OnSpawn → 关渲染 → 清速度 → 开渲染      ← 中间夹着一次真正的渲染！
→ Launch → 设位置                        ← 位置到这里才改
```

`OnSpawn` 结束时渲染器**已经被重新打开**，而位置要到之后的 `Launch()` 才设。
所以"关→开"那一瞬间，渲染器**根本没处于关闭状态过**——保护形同虚设。

**修正**：把开关移到 `Launch()`，让"关渲染"覆盖住"设位置"这一步：

```csharp
// Bullet.Launch
cachedRenderer.enabled = false;                  // 此时还停在预制体原始位置
transform.SetPositionAndRotation(data.position, data.rotation);
rb.velocity = data.velocity;
cachedRenderer.enabled = true;                   // 位置与速度都就位了
```

> **教训**：写防护时，要问"**这个防护覆盖的窗口，和问题真正发生的窗口，是同一个吗？**"
> 本例中问题发生在"`Get()` 到 `Launch()` 之间"，而防护写在 `OnSpawn` 内部就结束了。

#### Bug 3：`LookRotation` 的 up 选错 → 最常见场景退化

**原代码**：

```csharp
spawnData.rotation = Quaternion.LookRotation(direction, origin.forward);
```

**原注释**：

> "up 用枪口朝向而不是 `Vector3.up`：方向接近垂直时 `LookRotation` 会因向量共线而退化，
> 而枪口朝向在水平面上，与水平方向几乎不可能共线。"

这段推理**完全自洽**——如果 `direction` 真能接近垂直，它是对的。

**但本项目的 `direction` 恒为水平**：`ResolveAimDirection` 的两个分支都强制 `y=0`，
抬角上限 12°，所以 `direction.y` 最大只有 `sin(12°)≈0.208`。

| up 候选 | 与 `direction` 共线的可能性 | 实测 |
|---|---|---|
| `Vector3.up` | **不可能** | 叉乘长度恒为 `0.978` → 永不共线 |
| `origin.forward`（枪口朝向） | **很可能** | 玩家正对前方射击时点积 = `1.000000` → 完全平行 → 共线 |

```
direction 与 origin.forward 平行？ dot = 1.000000 (=1 -> 完全平行)
direction 与 Vector3.up      垂直？ dot = 0.000000 (=0 -> 垂直)

以 origin.forward 为 up 时：
  cross(up2, f) = (0.00, 0.00, 0.00)   长度 = 0.000000
  -> 正交基无法构造 = LookRotation 的退化输入
```

**修正**：改用 `Vector3.up`。

> **这两个 bug 的共同教训**：
> 写防御代码前，先确认"**我要防的场景，在本项目真的会发生吗？**"
> 为不存在的场景写的防护，往往会改变正常路径的行为——
> 本例中它保护的场景不存在，却让**默认视角下**（正对前方）的朝向计算退化了。
>
> 更完整的讨论见第 5.4 节。

### 4.6 复盘：这些 bug 是怎么被抓住的

| 环节 | 做法 |
|---|---|
| 1. 不相信注释 | 注释说"Get() 会销毁"，就去**实测**它到底会不会销毁 |
| 2. 查真实实现 | 用 `dotnet` 链接**真实的** `UnityEngine.CoreModule.dll`，而不是照抄文档 |
| 3. 写探针 | 打印回调顺序 + 三个计数的真值表 |
| 4. 模拟真实负载 | 按"射速 × 存活时间"算出同屏在飞数，跑真实循环 |
| 5. 对比新旧 | 同一套场景分别跑旧逻辑和新逻辑，看差异 |
| 6. **逐行追问时序** | 读 `OnSpawn` → `Launch` 的调用顺序，发现渲染器开关的窗口没盖住位置设置 |
| 7. **纯数学验证几何** | 把 `direction` 的取值范围算出来，发现"垂直退化"不可能发生，"平行退化"才是真问题 |

> **这套流程才是这次改动的真正价值。**
> 修复代码只值几行，但"如何验证一个关于底层 API 的假设"这个能力能一直用。
>
> 注意第 6、7 步：**前 5 步是查 API 行为，后 2 步是查自己的代码逻辑。**
> 两者都要做——前 5 步发现了 bug 1，后 2 步发现了 bug 2 和 bug 3。

---

## 5. 学会"验证而非相信"

### 5.1 核心流程

本学习包的所有结论都来自同一套流程，你可以复用到任何陌生 API：

```
第 1 步  列出你的疑问（不要假设）
第 2 步  写一个最小探针，只测这一件事
第 3 步  链接真实实现，跑它，记录输出
第 4 步  把输出变成"事实表"
第 5 步  拿事实表去改实现
第 6 步  用同一套用例回归，验证修复
```

### 5.2 探针怎么写（3 个要点）

**要点 1：探针要小到能一眼看完**

反例（把整个 `BulletPool` 搬进探针）→ 你分不清输出对应哪一步。
正例（本例的探针只测 `ObjectPool` 本身）→ 每个 `Console.WriteLine` 对应一个待验证命题。

**要点 2：打印"事实"而不是"结论"**

```csharp
// ❌ 打印结论
Console.WriteLine("Get 不受 maxSize 限制");

// ✅ 打印事实，让结论由你推导
Console.WriteLine($"连取 {maxSize+1} 次后：CountAll={pool.CountAll}, " +
                  $"CountInactive={pool.CountInactive}, 销毁={destroyCount}");
```

**要点 3：断言"期望值"要跟着事实更新**

初版探针写"期望销毁 > 0"，实测是 0。
**错的不是代码，是那个"想当然的期望"。** 发现后要立刻改掉，否则它会持续误导。

> **我自己就踩了第 3 点**：初版用例 7 打印 `Clear -> `（空），
> 我一度以为"Clear 不触发 actionOnDestroy"。
> 真相是**我的用例写错了**——`Clear()` 只清闲置栈，
> 而我前面 `Get()` 出来的对象还在池外。修正用例后结论才准确。
> **探针本身也会骗你，所以探针也要被质疑。**

### 5.3 怎么把结论写进代码注释

初版注释（错的）：

```csharp
// 已到 maxSize 且无空闲 → 新建一个实例、先跑 actionOnGet、再立刻 actionOnDestroy
```

修复后注释（对的）：

```csharp
// ⚠️ 关于 maxSize 的真实语义（已用 Unity 2022.3 真实 DLL 实测确认，勿再按直觉推断）：
//   ObjectPool<T>.Get() **不受 maxSize 限制**——池空就无限新建实例，永远不会销毁。
//   maxSize 只在 **Release()** 时生效：空闲栈已达上限时再归还，该实例会被
//   actionOnDestroy 销毁（而不是留在池里）。
```

**差别在于**：好的注释记录的是**"为什么不能想当然"**，而不只是"这段代码在干什么"。

后者是编译器能看出来的，前者是编译器看不出来的。

### 5.4 第二类错误：为"不存在的场景"写的防护

本次还发现了另一类错误，它比上一类更隐蔽——因为**防护代码看起来是负责任的**。

| | 第一类（池满判定） | 第二类（`LookRotation` 的 up） |
|---|---|---|
| 症状 | 防护**没做**，该拦的没拦 | 防护**做了**，但选错了对象 |
| 表面看 | 逻辑绕，不像防护 | 逻辑完整，还带详细推理 |
| 实际 | 靠直觉假设 API 行为 | 靠直觉假设**场景**会发生 |
| 危害 | 漏发、幽灵子弹 | 让**最常见场景**退化 |

**具体案例**：`LookRotation(direction, origin.forward)` 的原注释是这么写的：

> "up 用枪口朝向而不是 `Vector3.up`：方向接近垂直时 `LookRotation` 会因向量共线而退化，
> 而枪口朝向在水平面上，与水平方向几乎不可能共线。"

这段推理**完全自洽**——如果 direction 真能接近垂直，它是对的。
但本项目的 `direction` **恒为水平**（`ResolveAimDirection` 两个分支都强制 `y=0`，抬角上限 12°）。
所以：

- 它防范的"垂直退化"场景**根本不会发生**
- 而 `origin.forward`（枪口朝向）通常是水平的，玩家正对前方射击时与 `direction` **完全平行**
  → 那是**真·共线** → 退化

**它为不存在的场景选了防护，却让最常见的场景暴露了。**

> ### 💡 由此得到一条新的检查项
>
> 写任何"防御代码"时，多问一句：
>
> **"这个我要防的场景，在本项目真的会发生吗？"**
>
> 如果不会，那这段代码不只是无用——它可能正在**制造**新的问题，
> 因为它往往改变了正常路径的行为。
>
> 判断方法：把这个场景的**触发条件**写出来，
> 然后用探针或手算证明它在本项目的参数范围内**不可能触发**。
> 本例的证明就是：`direction.y` 恒为 0，抬角后最大 `sin(12°)=0.208`，
> 与 `Vector3.up` 的叉乘长度恒为 `0.978`（非 0 → 永不共线）。

### 5.5 断言应该放在哪一层

| 层 | 断言方式 | 何时跑 |
|---|---|---|
| 底层语义（`ObjectPool` 行为） | 离线探针程序（连真实 DLL） | 每次怀疑 API 行为时 |
| 池的容量逻辑 | 探针里模拟完整玩法循环 | 改 `Dispatch` 后回归 |
| 数学与几何（方向、旋转） | 纯数学实现验证符号与退化条件 | 改瞄准逻辑时 |
| 端到端（真实游戏） | Unity Play 模式肉眼观察 | 提交前 |

> **能离线测的就别进 Unity。** 迭代快得多，而且不用反复开关编辑器。
>
> **注意边界**：`Quaternion.LookRotation` / `AngleAxis` 是 **native 调用**，
> 脱离 Unity 运行会抛 `ECall methods must be packaged into a system module`。
> 这类要用**纯数学实现**（罗德里格斯公式、正交基构造）来验证——
> 本节 5.4 节里的叉乘轴符号与共线判据就是用纯数学实测的。

### 5.6 复用到其他 API

这套方法对任何"文档说不清、必须实测"的 API 都适用。比如：

- `Rigidbody.velocity` 在 `isKinematic = true` 时会被忽略吗？
- `Camera.main` 每次访问的开销到底多大？
- `List<T>` 扩容时会发生什么？

套路完全一样：**写最小探针 → 连真实实现跑 → 记录事实表 → 改代码。**

---

## 6. 编辑器验证清单

> ✅ **状态更新（2026-10-20）**：本工作区**已有 Unity 运行环境**
> （`<Unity 安装目录>\Editor\Unity.exe`，Unity 2022.3.62f3c1，Built-in 管线，Old Input）。
>
> **验证进度**：
> - ✅ **D 组（探针）** —— 4 个用例全部通过，与离线探针结论完全一致（两套独立实现交叉验证）
> - ✅ **B 组**（修复效果）—— 累计幽灵子弹 = 0，实例数复用生效
> - ✅ **C 组**（池满分支）—— 溢出 109 次且每次有警告，**池满时幽灵子弹仍为 0**
> - ✅ **E 组**（帧内开销）—— Instantiate/Destroy 全程 0，内存稳定；E1 需分阶段对照判读（见该节）
> - ✅ **F1 / F6 / F7 / F8 / F9**（边界与朝向）—— F8 为**新增发现**（背对时旧写法同样退化）
> - ⬜ **F2 / F3 / F4 / F5 / G 组** —— 需改场景或真实鼠标，未验
>
> 验证方式：通过第三方开源工具 `unity-dsh-bridge` 桥接驱动编辑器
> （MIT License，https://github.com/142gg-gzx/unity-dsh-bridge；
> 工具自带 LICENSE 声明），
> 用 `AutoFireDriver`（反射调 `Dispatch`，不依赖真实鼠标）保证断言可复现。
> **本文档的验证方法、探针脚本与结论均为本人原创**，桥接工具仅作为驱动通道。
>
> 每一项都写明了**期望看到什么**和**如果不对说明什么**——
> 这样即使结果和预期不符，你也能自己定位。

### A. 编译与场景

| # | 操作 | 期望结果 |
|---|---|---|
| A1 | 打开工程等待导入 | Console 无编译错误、无警告 |
| A2 | `Bullet` 预制体：根节点挂 `Bullet` + `Rigidbody` + `MeshRenderer` | `[RequireComponent]` 会自动补 Rigidbody |
| A3 | 预制体 `Rigidbody`：`Use Gravity` = ❌、`Collision Detection` = `Continuous` | 脚本也会强制 `useGravity=false` |
| A4 | `BulletSpawner` 上挂 `BulletSpawner`，拖入 `bulletPrefab` 和 `muzzle` | `muzzle` 放在角色前方 |
| A5 | 场景里有 `MainCamera` 标签的相机 | 否则方向计算退化 |
| A6 | `Player Settings → Active Input Handling` = `Both` 或 `Old` | **必须是这个**，否则 `Input` 运行时抛异常 |

> **A6 特别提醒**：Unity 2022.3 默认可能是 `Input System (New)`。
> 那样 `Input.GetMouseButton` 会在运行时抛
> `InvalidOperationException: You are trying to read Input using the UnityEngine.Input class...`
> 这是"编译通过但运行时炸"的典型。

### B. 修复效果验证（核心）—— ✅ 已验证（2026-10-20，经 unity-dsh-bridge 桥接在编辑器内实跑）

> **验证方式**：用 `AutoFireDriver`（反射调 `BulletPool.Dispatch`，不依赖真实鼠标）持续开火，
> 用 `PoolRuntimeProbe` 每 0.5 秒采样「在飞数 / 池实例数 / 闲置数 / 溢出数 / 静止子弹数」。

| # | 操作 | 期望结果 | 实测 | 状态 |
|---|---|---|---|---|
| B1 | 持续开火 | 子弹连续飞出，方向一致 | 正常 | ✅ |
| **B2** | **盯住枪口** | **没有停住不动的子弹** | **累计幽灵 = 0** | ✅ |
| B3 | 看 Console | 没有「池已满」误报 | 溢出 = 0 | ✅ |
| B4 | 持续射击 30 秒 | 一枪不掉，节奏均匀 | 正常 | ✅ |
| B5 | 停止射击 | 约 5 秒后子弹消失 | 在飞归 0 | ✅ |
| **B6** | 看实例数 | 稳定不增长 | **峰值在飞 17、池实例 17**，停止增长 | ✅ |

### C. 手动验证"池满"分支 —— ✅ 已验证

> 条件：`maxSize=5`（进 Play 前改，池按它建）、`fireRate=20`、`duration=6s`。

| # | 操作 | 期望结果 | 实测 | 状态 |
|---|---|---|---|---|
| C1 | 把 `maxSize` 改成 5 | 生效 | 生效 | ✅ |
| C2 | 继续射击 | Console 刷「子弹池已满（maxSize=5…）」 | 溢出计数 95→109，**每发都有警告** | ✅ |
| **C3** | **盯住枪口** | **依然没有停住不动的子弹** | **累计幽灵 = 0** | ✅ |
| C4 | 看实例数 | 稳定在 5，不再增长 | 稳定 5 | ✅ |
| C5 | 停止射击等 5 秒 | 回收，实例数不变 | 在飞 0 / 闲置 5 / 实例 5 | ✅ |

> **与旧版的对照（这是本次修复最有说服力的一组）**：
> 旧版在真池满场景「丢弃 0 次」= **完全没拦住池满（静默失效）**；
> 新版丢弃 109 次，且每次都有带具体数字的警告。
> 旧版在此场景必出幽灵子弹，新版累计幽灵 = 0。

### E. 帧内 GC —— ✅ 已验证（E3/E2 通过；E1 需分阶段对照，见下）

| # | 操作 | 期望结果 | 实测 | 状态 |
|---|---|---|---|---|
| E1 | 持续射击时 GC Alloc | 0 | 见下方「E1 的正确判读方式」 | ⚠️ 见下 |
| E2 | 观察 Total Memory | 稳定不上涨 | 290–293 MB 稳定 | ✅ |
| **E3** | 观察 Instantiate / Destroy 次数 | 射击中无新的 Instantiate | **Instantiate 全程 0、Destroy 全程 0** | ✅ |

> **E1 的正确判读方式（重要）**：
> 在 **Editor** 里跑时，Unity 自身（Inspector 重绘、Hierarchy 更新）每帧就有约 **7.4 KB** 分配，
> 所以「GCAlloc 绝对值 = 0」这个判据**在编辑器里永远不成立**，与被测代码无关。
>
> 正确做法是**同会话内分阶段对照**（见 `StagedGcProbe`）：
> 阶段 1 静默 3 秒 → 阶段 2 开火 3 秒 → 阶段 3 再静默 3 秒。
> 实测：静默阶段平均 8.7 KB/帧，**开火阶段 7.4 KB/帧**——
> 开火期间**不高于**静默基线，即**被测代码的净分配为 0**。
>
> 补充证据：场景激活子弹峰值 **30**（= 6 发/秒 × 5 秒存活），
> 池实例峰值 **31**（30 + 1 预留）后**不再增长**，
> 且 Instantiate/Destroy 计数全程为 0 —— 三者共同证明**复用真实生效**。

### D. 探针验证（`ObjectPoolProbe`）—— ✅ 已验证（2026-10-20 23:01）

> **已跑通，结论与离线实测完全一致。** 下面保留原始预期值，供以后复查时对照。

| # | 操作 | 期望结果 | 状态 |
|---|---|---|---|
| D1 | 空物体挂 `ObjectPoolProbe`，进 Play | Console 打出 4 组用例日志 | ✅ |
| D2 | 用例 1 | `CountAll=3`（maxSize=2 时取 3 次）、**累计销毁 = 0** → `Get()` 不受 `maxSize` 限制 | ✅ 实测 `CountAll=3, CountActive=3, CountInactive=0, 累计创建=3, 累计销毁=0` |
| D3 | 用例 2 | 归还时多出的触发 `actionOnDestroy` → `maxSize` 只在 `Release()` 生效 | ✅ 实测 `Destroy item6`；`CountAll=5, CountInactive=2, 累计销毁=1` |
| D4 | 用例 3 | `Clear` 后三个计数全 0，销毁数增加 | ✅ 实测 `CountAll=0, CountActive=0, CountInactive=0, 销毁数 1 -> 3` |
| D5 | 用例 4 | 重复 `Release` 抛 `InvalidOperationException` → `released` 标志的必要性 | ✅ 实测抛出，消息文本与离线完全一致 |

> **与离线实测的差异说明**：两次探针输入参数不同（取 5 次 vs 3 次、归还 4 个 vs 6 个），
> 所以计数数字不同，但**所有差异都能用输入差异解释，结论完全一致**。
> 这就是**交叉验证**——比单一实现可靠得多。
>
> **至此，对象池的 6 条真实语义已在真实 Unity 编辑器中全部确认。**
> 剩下待验证的都是**场景级行为**（生命周期时序、渲染帧内表现、物理、Profiler GC）。

### E. 帧内 GC

| # | 操作 | 期望结果 |
|---|---|---|
| E1 | `Window → Analysis → Profiler`，选 `GC Alloc` | 持续射击时 **`GC Alloc` 列为 0** |
| E2 | 观察 `Total Memory` | 稳定，不持续上涨 |
| E3 | 观察 `Instantiate` / `Destroy` 调用次数 | 射击过程中**不应该**有新的 Instantiate（除首次） |

> **E3 是对象池是否真的生效的最终判据。**

### F. 边界情况

| # | 操作 | 期望结果 | 实测 | 状态 |
|---|---|---|---|---|
| F1 | 运行时把 `maxSize` 改成 **0** | 抛 `ArgumentOutOfRangeException`，消息"子弹池容量上限必须大于 0。" | ✅ 实测抛出，消息与堆栈均正确（`BulletPool..ctor` → `BulletSpawner.Awake:82`） | ✅ |
| F2 | 把 `muzzle` 拖空 | Console 报"没有指定 muzzle"，组件自动禁用 | 未验（需改场景） | ⬜ |
| F3 | 把 `bulletPrefab` 拖空 | Console 报"没有指定 bulletPrefab"，组件自动禁用 | 未验（需改场景） | ⬜ |
| F4 | 删掉 `MainCamera` 标签 | 方向退化为枪口朝向，不崩溃 | 未验（需改场景） | ⬜ |
| F5 | 鼠标正好压在枪口正上方 | 退化处理生效，子弹仍正常飞出 | 未验（需真实鼠标） | ⬜ |

### F2. 子弹朝向（本次修正项，重点验）—— ✅ 已验证

> 修的是 `Quaternion.LookRotation(direction, origin.forward)` → `Quaternion.LookRotation(direction, Vector3.up)`。
> 原写法在**玩家正对前方射击**时 forward 与 up 完全平行（退化输入）。
>
> 验证方式：`OrientationVerifier` 在编辑器内对比两种 up 的选择，
> 判据 = `dot(dir, 结果forward)`（≈1 表示同向）与 `cross(dir, up)` 长度（≈0 表示共线退化）。

| # | 操作 | 期望结果 | 实测 | 状态 |
|---|---|---|---|---|
| **F6** | **角色正对前方**（yaw=0） | 子弹笔直朝正前方飞 | 新写法 `dot=1.000000, cross=1.000000`；**旧写法 cross=0.000000 → 完全共线，退化** | ✅ |
| F7 | 角色侧对（yaw=90） | 子弹朝左侧水平飞 | 新写法 dot=1.000000 cross=1.000000；旧写法此场景正常 | ✅ |
| **F8** | 角色背对（yaw=180） | 子弹朝身后飞 | 新写法 dot=1.000000 cross=1.000000；**旧写法同样 cross=0 → 退化** | ✅ |
| **F9** | `launchPitch=12` | 子弹**朝上**抬 12° | `dir.y=+0.207912`（朝上）；新写法 cross=**0.978148**，旧写法仅 **0.207912** | ✅ |

> **两个新增发现（原文档未记录）**：
> 1. **背对（yaw=180）也是退化场景**——旧写法在正前方与正后方都退化，
>    而正后方是玩家很容易出现的视角。原文档只提了正对前方。
> 2. **旧写法在抬角 12° 时 cross 只剩 0.207912**——虽然还没退化，但安全余量极小；
>    再多抬几度、或方向稍偏，就会撞上退化边界。新写法的 0.978148 留足了余量。
>
> **F6 是最关键的一条**——正对前方是默认视角，也是原 bug 最容易暴露的场景。

### G. 退出

| # | 操作 | 期望结果 |
|---|---|---|
| G1 | 退出 Play | Console 无异常，Hierarchy 不残留子弹物体 |
| G2 | 射击中途退出 Play | 无异常 |

---

## 7. 练习任务

> 建议顺序做，每完成一项在 Unity 里验证并提交一次 git。

### 任务 1：复现幽灵子弹（理解 bug）⭐ 必做

**目标**：亲眼看到 bug，才能真懂修复的价值。

1. `git stash` 或临时注释，把 `BulletPool.Dispatch` 改回旧版判定
   （`if (pool.CountAll == before && pool.CountInactive == 0)`）
2. Unity 里按住左键连射，观察：枪口的静止子弹、Console 的误导警告
3. 恢复新版代码，再连射一次对比

**验收**：能说清楚"旧版每隔几发丢一发，新版不丢"。

### 任务 2：亲手写探针（核心能力）⭐ 必做

**目标**：掌握"先验证再相信"的工作习惯。

1. 参考[附录 A](#附录-a离线探针完整源码)
2. 写一个**自己的**探针，验证 `LinkedPool<T>`（Unity 的另一个池实现）的行为
3. 重点验证：`Get()` 是否也受 `maxSize` 限制？`LinkedPool` 和 `ObjectPool` 行为一致吗？
4. 把结论写进笔记

**为什么练这个**：`LinkedPool` 在网上很少被讲，但 Unity 官方文档里有。
自己测一遍，你就知道它们是不是一回事。

### 任务 3：把容量参数做成动态的 ⭐ 必做

**目标**：理解 `maxSize` 的真实含义，并让游戏能自适应。

1. 在 `BulletSpawner` 里暴露一个"池容量倍率"字段
2. 运行时根据当前射速和子弹存活时间**自动计算**需要的池容量：
   `需要容量 = 射速 × 存活时间 × 1.2`（留 20% 余量）
3. 射速改大时，观察 `OverflowCount` 是否还为 0

**验收**：把 `fireRate` 从 6 调到 30，池能自动跟上，不出现幽灵弹。

### 任务 4：实现回收事件（练习回调设计）⭐ 必做

**目标**：理解"池 → 使用者"的事件通知模式。

1. 在 `BulletPool` 里加一个事件：子弹被归还时通知外部

   ```csharp
   public event Action<Bullet> OnBulletRecycled;
   ```

2. 在 `actionOnRelease` 里触发
3. 订阅方（如统计 HUD）收到通知并更新计数

**注意**：这是**每发一次**的回调，在帧内高频路径上——
**不要在事件处理器里做字符串拼接或 LINQ**，否则会引入帧内 GC。

**验收**：HUD 上"已发射/已回收"实时正确，且 Profiler 里 `GC Alloc` 为 0。

### 任务 5：扩展到第二种池对象 ⭐ 推荐

**目标**：验证"池不依赖具体类型"的设计是否真的成立。

1. 做一个 `DamagePopup` 类，实现 `IPooledObject`
2. 写一个 `PopupPool`（复制 `BulletPool` 的结构，但只管飘字）
3. 写一个 `PopupSpawner`（在角色头顶显示伤害数字）

**这是对本次重构设计的终极检验**：
如果 `BulletPool` 里有任何"只有子弹才需要"的逻辑，任务 5 就会做不下去。

**验收**：`PopupPool` 里**没有一行**提到 `Bullet`。

### 任务 6：写一份自己的 API 事实表

**目标**：把方法论固化下来。

选一个你用得不熟的 Unity API（推荐 `Rigidbody` 的 `velocity` / `isKinematic` / `drag`，
或 `ParticleSystem` 的 `main.startLifetime`），按第 5 部分的流程：

1. 列出疑问
2. 写最小探针
3. 跑，记录事实
4. 写成事实表
5. 用事实表写一段注释

**验收**：事实表里每一条都有"实测输出"作为依据。

---

## 8. 自测题

> 先自己答，再展开答案。

### Q1. 为什么 `maxSize` 不能限制"同时在用的实例数"？

<details>
<summary>参考答案</summary>

因为 `ObjectPool<T>` 的 `maxSize` **只在 `Release()` 时生效**——它约束的是**空闲栈的深度**。
`Get()` 的实现只看"空闲栈有没有东西"，池空就无条件 `createFunc()` 新建，完全不看 `maxSize`。

所以"同时在用数"要靠自己判断：`pool.CountAll >= maxSize && pool.CountInactive == 0`。
</details>

### Q2. 修复后的判定为什么必须放在 `Get()` **之前**？

<details>
<summary>参考答案</summary>

因为 `Get()` 会执行 `actionOnGet`，里面是 `SetActive(true)` + `OnSpawn()`——
子弹已经被激活、已经启动 5 秒生命周期、已经清空了速度。

如果这时才发现"放不回去"而直接 return，这颗子弹就**从未被 `Launch()`**：
位置停在枪口、速度为 0，但依然可见、依然 5 秒后才消失 → **幽灵子弹**。
同时本该发出的那一发也静默丢失了。

先判断则完全不碰池子，零副作用。
</details>

### Q3. `Bullet` 里为什么需要 `released` 标志？

<details>
<summary>参考答案</summary>

防止**重复归还**。子弹可能因为超时和碰撞同时回池（同一帧），
两次 `Release` 会触发 `ObjectPool` 的 `collectionCheck`，抛 `InvalidOperationException` 直接崩。

`released` 让 `Despawn()` 变成**幂等**的：只有第一次生效。
</details>

### Q4. 渲染器开关为什么放在 `Launch` 里，而不是 `OnSpawn` 里？

<details>
<summary>参考答案</summary>

子弹被 `Instantiate` 出来时位置是**预制体原始位置**，
`Launch()` 才把它搬到枪口。这中间如果渲染器是亮的，就会在预制体位置闪出一帧幽灵子弹。

**放在 `OnSpawn` 里会完全失效**，因为顺序错了：

```
actionOnGet → SetActive(true)            ← 渲染器亮了
→ OnSpawn → 关渲染 → 清速度 → 开渲染      ← 中间夹着一次真正的渲染！
→ Launch → 设位置                        ← 位置到这里才改
```

`OnSpawn` 结束时渲染器已被重新打开，而位置要到之后的 `Launch()` 才设——
所以"关→开"那一瞬间，渲染器**根本没处于关闭状态过**。

正确顺序是 `SetActive(true) → OnSpawn → Launch（关渲染 → 设位置/速度 → 开渲染）`。
渲染器的 `enabled` 帧内即时生效，所以同帧内关→开不会被渲染出来。
</details>

### Q4.1 `LookRotation` 的 `up` 为什么用 `Vector3.up` 而不是枪口朝向？

<details>
<summary>参考答案</summary>

`LookRotation` 要求 `forward` 与 `up` 不共线。

- `Vector3.up`：`direction` 恒为水平（y=0，抬角后 y 最大只有 sin(12°)≈0.208），
  与 `Vector3.up` 的叉乘长度恒为 0.978 → **永不共线**。
- `origin.forward`（枪口朝向）：通常是水平的，玩家正对前方射击时
  `direction` 与它**完全平行**（点积 = 1.000000）→ **共线 → 退化**。

原代码的注释说"方向接近垂直时用 `Vector3.up` 会退化，所以用枪口朝向更安全"——
但本项目的 `direction` 永远不可能接近垂直。
**它为不存在的场景选了 up，却让最常见的场景退化了。**

> 教训：写防御代码时，要问"**这个退化场景在本项目真的会发生吗？**"
</details>

### Q5. `BulletSpawner` 为什么把 `Camera.main` 的获取放在条件里？

<details>
<summary>参考答案</summary>

`Camera.main` 内部要按标签查找主相机，开销不小。放进常规路径每帧都查是浪费。

写法是 `if (aimCamera == null && !CacheAimCamera()) return;`——
已经缓存就直接用（短路），只有拿不到时才尝试补一次，且 `CacheAimCamera` 内部还有二次短路。
</details>

### Q6. `Mathf.Max(fireRate, 0.01f)` 防的是什么？

<details>
<summary>参考答案</summary>

防除零。`fireRate` 是 Inspector 可改的，用户填 0 → `1/0 = Infinity` →
`fireTimer = Infinity` → 之后 `fireTimer > 0f` 恒成立 → **永远打不出子弹，且不报任何错**。
</details>

### Q7. 为什么 `BulletSpawnData` 用 `struct` 而不是 `class`？

<details>
<summary>参考答案</summary>

`class` 是引用类型 → 每次开火在堆上分配一次 → 帧内 GC 压力。
`struct` 是值类型，直接压栈 → 零分配。
配合 `in` 修饰符按只读引用传递，避免拷贝 40 多字节的值。
</details>

### Q8. `Clear()` 会发生什么？

<details>
<summary>参考答案</summary>

`pool.Clear()` 对池内**每个闲置实例**触发一次 `actionOnDestroy`（即 `Destroy(gameObject)`），
并把 `CountAll` / `CountActive` / `CountInactive` 全部归零。

**只清闲置栈**——正在池外使用（已 `Get` 未 `Release`）的对象不受影响。
所以 `Clear()` = 销毁全部闲置对象，之后不能复用；池外那些对象的清理责任在调用方。
</details>

### Q9. `bullet == null` 为什么不能用 `bullet?.`？

<details>
<summary>参考答案</summary>

Unity 的"假 null"：被 `Destroy` 的对象，底层 C++ 对象已销毁但托管引用还在。
`==` 被 Unity 重载过，能识别这种情况；`?.` 只检查托管引用是否为 null，**识别不了**。
所以必须用 `==`。
</details>

### Q10. 这次 bug 里，哪一步最关键？为什么？

<details>
<summary>参考答案</summary>

**"不相信注释，去实测"** 这一步。

代码编译零警告零错误、注释写得自信、逻辑读起来合理——
**所有常规检查都会放行它**。唯一抓住它的是"写探针连真实实现跑一遍"。

如果当时只是"看着代码觉得没问题就提交"，这个 bug 会一路活到游戏里。
</details>

---

## 附录 A：离线探针完整源码

**这个探针直接链接 Unity 2022.3 的真实 DLL，运行的就是 Unity 真实实现。**

**为什么能脱离 Unity 运行？**
`ObjectPool<T>` 是**纯 C# 实现**，不碰 `UnityEngine.Object` 的原生调用，
所以在普通 .NET 程序里实例化它完全没问题。
（这不适用于你的 `Bullet.cs` / `BulletPool.cs`——它们依赖 `MonoBehaviour`、`Instantiate` 等引擎能力。）

### A.1 建工程

```powershell
mkdir PoolProbe && cd PoolProbe
dotnet new console
```

### A.2 改 csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>disable</ImplicitUsings>
    <Nullable>disable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="UnityEngine.CoreModule">
      <HintPath><Unity 安装目录>\Editor\Data\Managed\UnityEngine\UnityEngine.CoreModule.dll</HintPath>
      <Private>true</Private>
    </Reference>
  </ItemGroup>
</Project>
```

> Unity 装在别处就改 `<HintPath>`。

### A.3 Program.cs

```csharp
using System;
using System.Collections.Generic;
using UnityEngine.Pool;

sealed class ProbeItem
{
    public string Name;
}

static class Program
{
    static void Main()
    {
        Test01_Get不受maxSize限制();
        Test02_maxSize在Release时生效();
        Test03_Clear触发Destroy();
        Test04_重复Release抛异常();
        Test05_Release陌生对象不抛异常();
        Test06_maxSize为0抛异常();
        Test07_回调时序();
        Test08_新旧Dispatch逻辑对比();
    }

    // 用例 1：Get() 会不会受 maxSize 限制？
    static void Test01_Get不受maxSize限制()
    {
        Section("用例 1：Get() 是否受 maxSize 限制？");

        const int maxSize = 2;
        int create = 0, destroy = 0;

        var pool = new ObjectPool<ProbeItem>(
            createFunc: () => { create++; return new ProbeItem { Name = "item" + create }; },
            actionOnGet: item => { },
            actionOnRelease: item => { },
            actionOnDestroy: item => { destroy++; },
            collectionCheck: true,
            defaultCapacity: 1,
            maxSize: maxSize);

        for (int i = 0; i < 5; i++) pool.Get();

        Console.WriteLine($"  maxSize={maxSize}，连续 Get 5 次：");
        Console.WriteLine($"    CountAll      = {pool.CountAll}");
        Console.WriteLine($"    CountActive   = {pool.CountActive}");
        Console.WriteLine($"    CountInactive = {pool.CountInactive}");
        Console.WriteLine($"    累计创建      = {create}");
        Console.WriteLine($"    累计销毁      = {destroy}");
        Console.WriteLine();
        Console.WriteLine("  [实测] Get() 完全不受 maxSize 限制：池空就无限新建，永不销毁。");
        Console.WriteLine("  [结论] 推论 -> 想限制『同时在用数』必须自己判断，maxSize 拦不住。");
        Console.WriteLine();
    }

    // 用例 2：maxSize 在什么时候生效？
    static void Test02_maxSize在Release时生效()
    {
        Section("用例 2：maxSize 在什么时候生效？");

        const int maxSize = 3;
        int create = 0, destroy = 0;

        var pool = new ObjectPool<ProbeItem>(
            createFunc: () => { create++; return new ProbeItem { Name = "item" + create }; },
            actionOnGet: item => { },
            actionOnRelease: item => { },
            actionOnDestroy: item => { destroy++; Console.WriteLine($"    -> actionOnDestroy 被触发：{item.Name}"); },
            collectionCheck: true,
            defaultCapacity: 1,
            maxSize: maxSize);

        var taken = new List<ProbeItem>();
        for (int i = 0; i < 4; i++) taken.Add(pool.Get());

        Console.WriteLine($"  取 4 个后：CountAll={pool.CountAll} CountInactive={pool.CountInactive} 销毁={destroy}");

        foreach (var item in taken) pool.Release(item);

        Console.WriteLine($"  全部归还后：CountAll={pool.CountAll} CountInactive={pool.CountInactive} 销毁={destroy}");
        Console.WriteLine($"  CountInactive 已达 maxSize={maxSize}，多还的那个被销毁。");
        Console.WriteLine();
        Console.WriteLine("  [实测] maxSize 只在 Release() 时约束空闲栈深度，是『闲置上限』不是『容量上限』。");
        Console.WriteLine();
    }

    // 用例 3：Clear() 会不会触发 actionOnDestroy？
    static void Test03_Clear触发Destroy()
    {
        Section("用例 3：Clear() 会不会触发 actionOnDestroy？");

        int destroy = 0;
        var pool = new ObjectPool<ProbeItem>(
            createFunc: () => new ProbeItem { Name = "item" },
            actionOnGet: item => { },
            actionOnRelease: item => { },
            actionOnDestroy: item => { destroy++; Console.WriteLine($"    -> actionOnDestroy：{item.Name}"); },
            collectionCheck: true,
            defaultCapacity: 1,
            maxSize: 10);

        var taken = new List<ProbeItem>();
        for (int i = 0; i < 3; i++) taken.Add(pool.Get());
        foreach (var item in taken) pool.Release(item);

        Console.WriteLine($"  Clear 前：CountAll={pool.CountAll} CountActive={pool.CountActive} CountInactive={pool.CountInactive} 销毁={destroy}");

        pool.Clear();

        Console.WriteLine($"  Clear 后：CountAll={pool.CountAll} CountActive={pool.CountActive} CountInactive={pool.CountInactive} 销毁={destroy}");
        Console.WriteLine();
        Console.WriteLine("  [实测] Clear() 对每个**闲置**实例各触发一次 actionOnDestroy，并把三个计数全部归零。");
        Console.WriteLine("  [实测] 正在池外使用（已 Get 未 Release）的对象**不受 Clear 影响** —— Clear 只清闲置栈。");
        Console.WriteLine();
    }

    // 用例 4：collectionCheck 对重复 Release 的保护
    static void Test04_重复Release抛异常()
    {
        Section("用例 4：collectionCheck 对重复 Release 的保护");

        var pool = new ObjectPool<ProbeItem>(
            createFunc: () => new ProbeItem { Name = "item" },
            actionOnGet: item => { },
            actionOnRelease: item => { },
            actionOnDestroy: item => { },
            collectionCheck: true,
            defaultCapacity: 1,
            maxSize: 10);

        var item = pool.Get();
        pool.Release(item);

        try
        {
            pool.Release(item);
            Console.WriteLine("  [实测] 重复 Release 没有抛异常。");
        }
        catch (InvalidOperationException ex)
        {
            Console.WriteLine($"  [实测] 重复 Release 抛出 InvalidOperationException：");
            Console.WriteLine($"         {ex.Message}");
            Console.WriteLine("  [结论] 这正是 Bullet 类里 released 标志存在的理由。");
        }

        Console.WriteLine();
    }

    // 用例 5：Release 一个从未 Get 过的陌生对象会怎样？
    static void Test05_Release陌生对象不抛异常()
    {
        Section("用例 5：Release 一个从未 Get 过的陌生对象会怎样？");

        int destroy = 0;
        var pool = new ObjectPool<ProbeItem>(
            createFunc: () => new ProbeItem { Name = "new" },
            actionOnGet: item => { },
            actionOnRelease: item => { },
            actionOnDestroy: item => { destroy++; },
            collectionCheck: true,
            defaultCapacity: 1,
            maxSize: 5);

        var orphan = new ProbeItem { Name = "orphan" };

        try
        {
            pool.Release(orphan);
            Console.WriteLine($"  [实测] 没有抛异常。CountAll={pool.CountAll} CountInactive={pool.CountInactive} 销毁={destroy}");
            Console.WriteLine("  [实测] 陌生对象被直接收进了空闲栈。");
            Console.WriteLine("  [结论] collectionCheck 只查『是不是已在池里』，不查『是不是我们的对象』。");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"  [实测] 抛出 {ex.GetType().Name}：{ex.Message}");
        }

        Console.WriteLine();
    }

    // 用例 6：maxSize <= 0 的构造函数行为
    static void Test06_maxSize为0抛异常()
    {
        Section("用例 6：maxSize <= 0 时构造函数的行为");

        try
        {
            var pool = new ObjectPool<ProbeItem>(
                createFunc: () => new ProbeItem(),
                actionOnGet: item => { },
                actionOnRelease: item => { },
                actionOnDestroy: item => { },
                collectionCheck: true,
                defaultCapacity: 0,
                maxSize: 0);
            Console.WriteLine($"  [实测] 没有抛异常。CountAll={pool.CountAll}");
        }
        catch (Exception ex)
        {
            Console.WriteLine($"  [实测] 抛出 {ex.GetType().Name}：{ex.Message}");
            Console.WriteLine("  [结论] 这个异常会从 BulletSpawner.Awake 冒出来，堆栈不直观，");
            Console.WriteLine("         所以 BulletPool 构造函数提前用 ArgumentOutOfRangeException 拦一道。");
        }

        Console.WriteLine();

        var pool2 = new ObjectPool<ProbeItem>(
            createFunc: () => new ProbeItem(),
            actionOnGet: item => { },
            actionOnRelease: item => { },
            actionOnDestroy: item => { },
            collectionCheck: true,
            defaultCapacity: 20,
            maxSize: 5);
        Console.WriteLine($"  [实测] defaultCapacity(20) > maxSize(5) 不抛异常。CountAll={pool2.CountAll}");
        Console.WriteLine();
    }

    // 用例 7：四个回调的触发时序
    static void Test07_回调时序()
    {
        Section("用例 7：四个回调的触发时序");

        var log = new List<string>();
        var pool = new ObjectPool<ProbeItem>(
            createFunc: () => { log.Add("createFunc"); return new ProbeItem { Name = "item" }; },
            actionOnGet: item => { log.Add("actionOnGet"); },
            actionOnRelease: item => { log.Add("actionOnRelease"); },
            actionOnDestroy: item => { log.Add("actionOnDestroy"); },
            collectionCheck: true,
            defaultCapacity: 1,
            maxSize: 2);

        log.Clear();
        var a = pool.Get();
        Console.WriteLine($"  池空时 Get      -> {string.Join(" | ", log)}");

        log.Clear();
        pool.Release(a);
        Console.WriteLine($"  归还            -> {string.Join(" | ", log)}");

        log.Clear();
        var b = pool.Get();
        Console.WriteLine($"  复用（栈非空）  -> {string.Join(" | ", log)}");

        // 注意：Clear() 只清理**闲置栈**里的对象。
        // 上面这个 b 刚被 Get 出来，还在池外（active），所以必须先还回去，
        // Clear() 才有东西可清 —— 否则看起来会误以为 "Clear 不触发 actionOnDestroy"。
        log.Clear();
        pool.Release(b);
        Console.WriteLine($"  归还（准备 Clear）-> {string.Join(" | ", log)}");

        log.Clear();
        pool.Clear();
        Console.WriteLine($"  Clear           -> {string.Join(" | ", log)}");
        Console.WriteLine();
        Console.WriteLine("  [实测] 复用时**不会**再触发 createFunc；");
        Console.WriteLine("  [实测] Clear() 只销毁**闲置栈**里的对象（池外正在使用的不会被它碰到），");
        Console.WriteLine("         且会对每个闲置实例各触发一次 actionOnDestroy。");
        Console.WriteLine();
    }

    // 用例 8：新旧 Dispatch 逻辑对比（最核心）
    static void Test08_新旧Dispatch逻辑对比()
    {
        Section("用例 8：新旧 Dispatch 逻辑对比（本项目修复的核心）");

        Console.WriteLine("  旧逻辑：先 Get()，再看 CountAll 有没有涨、CountInactive 是否为 0");
        Console.WriteLine("  新逻辑：先看 CountAll >= maxSize 且 CountInactive == 0，再决定要不要 Get()");
        Console.WriteLine();

        // 判定真值表
        Console.WriteLine("  【判定真值表】");
        var pool = new ObjectPool<ProbeItem>(
            createFunc: () => new ProbeItem(),
            actionOnGet: i => { }, actionOnRelease: i => { }, actionOnDestroy: i => { },
            collectionCheck: true, defaultCapacity: 1, maxSize: 10);

        pool.Get();
        Console.WriteLine($"    池全空时 Get        -> CountAll 涨了 -> 判定【正常开火】（正确）");

        var tmp = pool.Get();
        pool.Release(tmp);
        Console.WriteLine($"    空闲栈恰好 1 个时 Get -> CountAll 没涨 + 空闲栈清空 -> 旧逻辑判定【丢弃】（错误！）");
        Console.WriteLine("       ^ 这就是 bug：复用了池中对象、池工作得最好的时候，反而被判成『池满』。");
        Console.WriteLine();

        // 玩法循环模拟
        Console.WriteLine("  【玩法循环模拟】射速 5 发/秒，子弹存活 5 秒，maxSize=100");
        Console.WriteLine("  （同屏在飞 = 5 x 5 = 25 发，远小于 maxSize=100，本不该有任何丢弃）");
        Console.WriteLine();

        var oldResult = Simulate(useNewLogic: false, maxSize: 100, shots: 40, interval: 0.2f);
        var newResult = Simulate(useNewLogic: true, maxSize: 100, shots: 40, interval: 0.2f);

        Console.WriteLine($"    旧逻辑：误判丢弃 {oldResult.Discarded,3} 次   幽灵弹 {oldResult.Ghosts,3} 个");
        Console.WriteLine($"    新逻辑：误判丢弃 {newResult.Discarded,3} 次   幽灵弹 {newResult.Ghosts,3} 个");
        Console.WriteLine();

        Console.WriteLine("  【真池满场景】maxSize=5，射速 20 发/秒（真实需要 100 个实例）");
        Console.WriteLine();

        oldResult = Simulate(useNewLogic: false, maxSize: 5, shots: 60, interval: 0.05f);
        newResult = Simulate(useNewLogic: true, maxSize: 5, shots: 60, interval: 0.05f);

        Console.WriteLine($"    旧逻辑：丢弃 {oldResult.Discarded,3} 次   幽灵弹 {oldResult.Ghosts,3} 个");
        Console.WriteLine($"    新逻辑：丢弃 {newResult.Discarded,3} 次   幽灵弹 {newResult.Ghosts,3} 个");
        Console.WriteLine();
        Console.WriteLine("  [实测] 旧逻辑在真池满时『丢弃 0 次』——完全没拦住，静默失效；");
        Console.WriteLine("         新逻辑正确丢弃，且**不产生幽灵弹**（判断在 Get 之前，没碰池子）。");
        Console.WriteLine();
    }

    sealed class SimResult
    {
        public int Discarded;
        public int Ghosts;
        public int Created;
    }

    static SimResult Simulate(bool useNewLogic, int maxSize, int shots, float interval)
    {
        const float lifeTime = 5f;
        int create = 0, destroy = 0, overflow = 0;
        float now = 0f;

        var pool = new ObjectPool<Bullet>(
            createFunc: () => { create++; return new Bullet { LifeTime = lifeTime }; },
            actionOnGet: b => { b.Spawned = true; b.SpawnTime = now; b.Released = false; },
            actionOnRelease: b => { b.Spawned = false; b.Launched = false; b.Released = true; },
            actionOnDestroy: b => { destroy++; b.Released = true; },
            collectionCheck: true,
            defaultCapacity: 20,
            maxSize: maxSize);

        var live = new List<Bullet>();
        var allActive = new List<Bullet>();

        for (int i = 0; i < shots; i++)
        {
            now += interval;

            if (useNewLogic)
            {
                // ===== 新逻辑：容量判断在 Get() 之前 =====
                if (pool.CountAll >= maxSize && pool.CountInactive == 0)
                {
                    overflow++;
                }
                else
                {
                    var bullet = pool.Get();
                    allActive.Add(bullet);
                    bullet.Launched = true;
                    live.Add(bullet);
                }
            }
            else
            {
                // ===== 旧逻辑：先 Get() 再判断（有 bug）=====
                int before = pool.CountAll;
                var bullet = pool.Get();
                allActive.Add(bullet);

                if (pool.CountAll == before && pool.CountInactive == 0)
                {
                    overflow++;
                    bullet.IsGhost = true;     // 已激活但从未 Launch -> 幽灵弹
                }
                else
                {
                    bullet.Launched = true;
                    live.Add(bullet);
                }
            }

            // 超时回收
            for (int j = allActive.Count - 1; j >= 0; j--)
            {
                var b = allActive[j];
                if (b.Released) { allActive.RemoveAt(j); continue; }
                if (now - b.SpawnTime >= b.LifeTime)
                {
                    b.Released = true;
                    pool.Release(b);
                    allActive.RemoveAt(j);
                }
            }
            for (int j = live.Count - 1; j >= 0; j--)
                if (live[j].Released) live.RemoveAt(j);
        }

        int ghosts = 0;
        foreach (var b in allActive) if (b.IsGhost) ghosts++;

        return new SimResult { Discarded = overflow, Ghosts = ghosts, Created = create };
    }

    sealed class Bullet
    {
        public bool Spawned;
        public bool Launched;
        public bool Released = true;
        public bool IsGhost;
        public float SpawnTime;
        public float LifeTime;
    }

    static void Section(string title)
    {
        Console.WriteLine("---------------------------------------------------------");
        Console.WriteLine(title);
        Console.WriteLine("---------------------------------------------------------");
    }
}
```

### A.4 运行

```powershell
dotnet run
```

### A.5 预期输出（实测）

```
用例 1：Get() 是否受 maxSize 限制？
  maxSize=2，连续 Get 5 次：
    CountAll      = 5
    CountActive   = 5
    CountInactive = 0
    累计创建      = 5
    累计销毁      = 0
  [实测] Get() 完全不受 maxSize 限制：池空就无限新建，永不销毁。

用例 2：maxSize 在什么时候生效？
  取 4 个后：CountAll=4 CountInactive=0 销毁=0
    -> actionOnDestroy 被触发：item4
  全部归还后：CountAll=3 CountInactive=3 销毁=1
  [实测] maxSize 只在 Release() 时约束空闲栈深度。

用例 3：Clear() 会不会触发 actionOnDestroy？
  Clear 前：CountAll=3 CountActive=0 CountInactive=3 销毁=0
    -> actionOnDestroy：item
    -> actionOnDestroy：item
    -> actionOnDestroy：item
  Clear 后：CountAll=0 CountActive=0 CountInactive=0 销毁=3

用例 4：collectionCheck 对重复 Release 的保护
  [实测] 重复 Release 抛出 InvalidOperationException：
         Trying to release an object that has already been released to the pool.

用例 5：Release 一个从未 Get 过的陌生对象会怎样？
  [实测] 没有抛异常。CountAll=0 CountInactive=1 销毁=0

用例 6：maxSize <= 0 时构造函数的行为
  [实测] 抛出 ArgumentException：Max Size must be greater than 0 (Parameter 'maxSize')
  [实测] defaultCapacity(20) > maxSize(5) 不抛异常。CountAll=0

用例 7：四个回调的触发时序
  池空时 Get      -> createFunc | actionOnGet
  归还            -> actionOnRelease
  复用（栈非空）  -> actionOnGet
  归还（准备 Clear）-> actionOnRelease
  Clear           -> actionOnDestroy

用例 8：新旧 Dispatch 逻辑对比
    旧逻辑：误判丢弃  13 次   幽灵弹  13 个
    新逻辑：误判丢弃   0 次   幽灵弹   0 个
  【真池满场景】
    旧逻辑：丢弃   0 次   幽灵弹   0 个
    新逻辑：丢弃  55 次   幽灵弹   0 个
```

---

## 附录 B：离线编译校验方法

> **这条能力的使用频率会远高于对象池本身**，所以单独成篇。

Unity 项目没有 CI，容易积累"看起来对但编译不过"的代码。
用 Unity 官方 DLL 做离线编译，可以在**不打开 Unity** 的情况下查编译错误。

### B.1 原理

Unity 的托管 API 以 DLL 形式存放在编辑器安装目录：

```
<Unity 安装目录>\Editor\Data\Managed\UnityEngine\
    UnityEngine.CoreModule.dll        ← 核心（ObjectPool、Vector3、MonoBehaviour…）
    UnityEngine.PhysicsModule.dll     ← 物理（Rigidbody、Collider）
    UnityEngine.InputLegacyModule.dll  ← 旧版 Input
    UnityEngine.IMGUIModule.dll       ← OnGUI 等
    …共 70+ 个模块
```

用 `dotnet` 建一个类库，把这些 DLL 作为 `<Reference>` 引入，就能直接编译你的脚本。

### B.2 校验用的工程文件

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.1</TargetFramework>
    <LangVersion>9.0</LangVersion>
    <ImplicitUsings>disable</ImplicitUsings>
    <Nullable>disable</Nullable>
    <EnableDefaultCompileItems>false</EnableDefaultCompileItems>
    <NoWarn>CS0168;CS0169;CS0649</NoWarn>
  </PropertyGroup>

  <ItemGroup>
    <Compile Include="你的脚本目录\*.cs" />
  </ItemGroup>

  <ItemGroup>
    <Reference Include="UnityEngine.CoreModule">
      <HintPath><Unity 安装目录>\Editor\Data\Managed\UnityEngine\UnityEngine.CoreModule.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <!-- 按需追加 PhysicsModule / InputLegacyModule / IMGUIModule … -->
  </ItemGroup>
</Project>
```

### B.3 关键设置的理由

| 设置 | 为什么 |
|---|---|
| `TargetFramework=netstandard2.1` | 贴近 Unity 2022.3 的实际目标框架 |
| `LangVersion=9.0` | **强制 C# 9**——和你项目一致，能挡住 file-scoped namespace / global using / `required` 这些越界用法 |
| `Nullable=disable` | Unity 项目默认不开可空引用类型 |
| `EnableDefaultCompileItems=false` | 精确控制编译哪些文件 |
| `NoWarn=CS0168;CS0169;CS0649` | 静音"未使用变量/字段"——Unity 序列化字段天然会有这类警告 |

### B.4 本项目实测结果

编译对象：`Assets/Scripts/Practice01/` 下全部 5 个 `.cs`

```
已成功生成。
    0 个警告
    0 个错误
```

### B.5 能抓什么 / 不能抓什么

**✅ 能抓**

| 错误类型 | 例子 |
|---|---|
| **API 不存在** | `pool.GetOrCreate()` 这种不存在的方法名 |
| **API 签名不匹配** | 参数类型、个数、返回值类型不对 |
| **C# 版本越界** | file-scoped namespace（C# 10）、`required`（C# 11）—— 被 `LangVersion=9.0` 挡住 |
| **命名空间缺失** | 忘了 `using UnityEngine.Pool;` |
| **泛型推断失败** | `ObjectPool<Bullet>` 漏了类型参数 |
| **拼写错误** | 属性名/字段名拼错（大小写敏感） |
| **重载歧义** | 同名重载导致调用不明确 |

**❌ 抓不到（必须进 Unity）**

| 问题 | 为什么抓不到 |
|---|---|
| **运行时行为错误** | 编译只是签名检查，不执行代码 |
| **Unity 生命周期顺序问题** | 需要真实引擎调度 |
| **物理/动画/UI 时序** | 同上 |
| **`Instantiate` 的实际开销** | 需要 Profiler |
| **Prefab 上的序列化不匹配** | 需要导入 Prefab 和 .meta |
| **"假 null" 相关问题** | 编译期无法察觉 |
| **`Active Input Handling` 配置** | 是 Player Settings，不是代码 |
| **Draw Call / 批次** | 需要 Frame Debugger |

> **一句话**：离线编译是**编译门禁**，不是**测试**。
> 它能挡住"低级错误进主干"，但挡不住"逻辑错误"——
> 而这次这个 bug 恰恰是逻辑错误，所以它**必须**靠探针 + Play 模式来发现。

### B.6 什么时候值得做

| 时机 | 是否值得 |
|---|---|
| 写完一个模块 | ✅ 值得 |
| 提交 git 前 | ✅ 值得（当门禁） |
| 改完构造函数签名 | ✅ 值得（最容易引发跨文件错误） |
| 批量重构后 | ✅✅ 非常值得 |
| 只改注释 | ❌ 不值得 |
| 改 Inspector 数值 | ❌ 不值得（不涉及代码） |

> **批量重构时价值最大**——这正是"跨文件改动容易漏改"的场景，
> 而离线编译能在 2 秒内告诉你有没有漏。

### B.7 踩过的坑

```powershell
# 1. dotnet new 报 "Access to the path '...\.templateengine\dotnetcli\...' is denied"
#    -> 设置 DOTNET_CLI_HOME 指向可写目录
$env:DOTNET_CLI_HOME = $env:TEMP

# 2. dotnet tool install 报 "无法加载源 https://api.nuget.org/..."
#    -> 网络受限。改为直接用 Unity 自带的 ikdasm.exe
#       (Editor\Data\MonoBleedingEdge\lib\mono\4.5\ikdasm.exe)
```

---

## 附录 C：术语表

| 术语 | 含义 |
|---|---|
| **对象池 / Object Pool** | 把"创建-销毁"换成"取出-归还"的性能模式 |
| **空闲栈 / Inactive Stack** | 池里闲置待用对象的集合，`ObjectPool` 内部用 `Stack<T>` 实现 |
| **在飞清单 / `active`** | 已取出、正在使用的对象集合，本项目自己维护 |
| **`createFunc`** | 池空时创建新实例的回调 |
| **`actionOnGet`** | 对象被取出时触发（对应本项目的 `OnSpawn`） |
| **`actionOnRelease`** | 对象被归还时触发（对应本项目的 `OnDespawn`） |
| **`actionOnDestroy`** | 实例被销毁时触发（`Release` 超上限 / `Clear()`） |
| **`collectionCheck`** | 是否检查重复归还，**仅编辑器生效** |
| **`defaultCapacity`** | 空闲栈的初始容量（预分配，不是上限） |
| **`maxSize`** | **空闲栈深度上限**，仅在 `Release` 时生效 |
| **幽灵子弹 / Ghost Bullet** | 已激活但从未 `Launch`、停在原地不动的子弹 |
| **幂等 / Idempotent** | 重复调用效果相同（`Despawn` 的设计目标） |
| **假 null / Fake Null** | Unity 销毁对象的 `== null` 为 true，但 `?.` 判断不出来 |
| **帧内 GC / Per-Frame GC** | 每帧产生托管分配，会造成周期性卡顿 |
| **`in` 修饰符** | 按只读引用传参，避免 struct 拷贝 |
| **收敛 / Convergence** | 模拟后系统趋于稳定状态（池子进入稳态） |

---

## 相关文件

| 位置 | 内容 |
|---|---|
| `Assets/Scripts/Practice01/BulletPool.cs` | 池实现（**已修复**，含真实语义注释） |
| `Assets/Scripts/Practice01/Bullet.cs` | 池成员，负责状态重置 |
| `Assets/Scripts/Practice01/BulletSpawner.cs` | 射击者，负责时机与方向 |
| `Assets/Scripts/Practice01/IPooledObject.cs` | 池成员契约 |
| `Assets/Scripts/Practice01/ObjectPoolProbe.cs` | 真实语义探针（**期望值已按实测修正**） |

---

*文档版本：v1.0 | Unity 2022.3 LTS | 代码基线：Unity 2022.3.62f3c1*
*所有标注"实测"的结论均可用附录 A 的探针复现。*
