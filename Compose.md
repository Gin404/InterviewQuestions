## 什么是Jetpack Compose

Jetpack Compose 是 Google 官方推出的现代声明式 UI 工具包，用于构建原生 Android 界面 。它完全抛弃了传统的 XML 布局文件，让你使用纯 Kotlin 代码来定义 UI。

**核心特征**

1. 声明式编程：描述 UI 应该是什么样子，而不是一步步告诉程序如何构建它
2. 基于函数：UI 由带有 @Composable 注解的 Kotlin 函数组成
3. 响应式更新：当状态变化时，UI 自动更新

**和传统的xml对比**

1. 不需要xml和findViewById，用函数直接定义UI。UI即代码。
2. 不需要setText手动更新文本，状态变化自动触发UI刷新。
3. RecyclerView需要单独声明Adapter；LazyColumn直接声明列表。
4. UI和逻辑紧密联系，而不是分离在不同地方。

**解决了什么问题**

1. 传统 Android 开发中，一个简单的列表需要 XML 布局文件、Adapter 类、ViewHolder、Item 布局……而在 Compose 中，只需几行代码。

```python
LazyColumn {
    items(20) { index ->
        Text("Item #$index")
    }
}
```

2. 消除findViewById和视图绑定。不需要维护视图引用。
3. 避免手动更新UI。
4. UI状态集中管理，更少的bug来源。
5. 与Kotlin完美集成。（协程、Flow、高阶函数等等）

**Compose函数如何转化为屏幕像素的？**

三大阶段：组合（Composition）、布局（Layout）和绘制（Drawing）

**第一阶段：组合-生成UI蓝图**

1. 输入与输出：当你的 `@Composable` 函数首次执行时，它就像一个蓝图生成器。函数的输入是一些数据（参数、状态），输出则是一个用 **`LayoutNode`** 构成的虚拟 UI 树。这个树描述了“界面上有什么”，比如一个 `Column` 里有一个 `Text` 和一个 `Button`。
2. 背后的功臣**SlotTable**：组合阶段产生的信息，包括 UI 的结构、你通过 `remember` 存储的状态等，并不会随着函数执行完毕而消失。它们会被保存在一个名为 **`SlotTable`** 的高效数据结构中。
3. 与传统的View对应：这个过程对应了传统 View 系统通过解析 XML 并在代码中 `inflate` 生成 `View` 对象树的过程。但 Compose 的效率更高，因为它直接在 Kotlin 代码中完成，并且有了 `SlotTable` 这个强大的“记事本”。

**第二阶段：布局-确定位置和大小**

有了 UI 树的蓝图后，下一步就是确定每个元素应该有多大、放在哪里。

1. **如何工作**：Compose 会遍历由 `LayoutNode` 构成的树，执行三大布局操作：
   1. **测量**：父节点会询问每个子节点需要多大的空间。子节点根据父节点施加的约束（例如“宽度不能超过 200dp”）和自己要显示的内容（比如文字的多少），计算出自己的尺寸。
   2. **摆放**：所有节点都确定尺寸后，父节点开始为它们指定在屏幕上的具体位置坐标。
2. **与传统 View 的对应**：这一步和传统 View 系统的 **`onMeasure()`** 和 **`onLayout()`** 流程在概念上非常相似。不同之处在于，Compose 的布局过程完全由 Kotlin 代码控制，更灵活，也更容易实现自定义布局（只需实现一个 `MeasurePolicy`）。

**第三阶段：绘制-渲染到屏幕**

这是最后一步，将布局好的 UI 元素真正画到屏幕上。

1. **如何工作**：Compose 再次遍历 `LayoutNode` 树，每个节点会执行自己的绘制逻辑。例如，`Text` 节点会调用底层的文本绘制 API 来画出文字，`Canvas` 节点则会执行你在 `drawBehind` 或 `drawWithContent` 中定义的各种绘制指令
2. **与传统 View 的对应**：这一步完美对应传统 View 的 **`onDraw()`** 过程。区别在于，Compose 提供了一个更强大、更现代化的声明式 `Canvas` API，让你可以用 Kotlin 代码来描述所有绘制操作。

## 结合简单例子说明原理

```
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Clicked $count times")
    }
}

```

### 组合阶段

**1. 编译器插桩**

```
@Composable
fun Counter($composer: Composer<*>, $changed: Int) {
    // 1. 开始一个可以重组的 Group
    $composer.startRestartGroup(/* 唯一key */)
  
    // 2. 处理 remember 和 mutableStateOf
    var count by $composer.cache(false) { mutableStateOf(0) }
  
    // 3. 判断是否需要执行具体内容（智能重组的关键）
    if (/* 参数或状态有变化 */) {
        Button(/* ... */, $composer, 0) {
            Text("Clicked $count times", $composer, 0)
        }
    } else {
        $composer.skipToGroupEnd() // 跳过，实现“智能重组”
    }
  
    // 4. 结束 Group，并注册重组回调
    $composer.endRestartGroup()?.updateScope { nextComposer ->
        Counter(nextComposer, 0 or 0b0001)
    }
}
```

编译器主要做了这几件事:

1. **增加参数**：每个 `@Composable` 函数都被悄悄塞进了两个参数：`$composer: Composer<*>` 和 `$changed: Int`。`$composer` 是连接组合与渲染的桥梁，`$changed` 用来标记参数是否变化。
2. **包裹 `RestartGroup`**：函数体的首尾被 `startRestartGroup` 和 `endRestartGroup` 包裹。这定义了一个 **“重组作用域”**。当 `count` 这个 `State` 变化时，只有这个作用域内的代码会被重新执行，从而实现“智能重组” 。
3. **插入跳转逻辑**：通过 `if` 判断和 `skipToGroupEnd()`，Compose 可以在参数没有变化时，跳过整个函数的执行，极大提升性能 。

**2.Composer 与 SlotTable：数据的“账本”**

那么，`$composer` 拿着这些信息做了什么？它将这些信息记录在了一个叫 **`SlotTable`** 的核心数据结构里。

`SlotTable` 可以理解为一个高效的线性内存“账本”，由两个数组组成：

**`groups: IntArray`**：这是一个 `Int` 数组，每 5 个 `Int` 为一组，记录一个 `Group` 的元信息。比如 Group 的类型、父子关系、在 `slots` 数组中的位置等。

**`slots: Array<Any?>`**：这是一个对象数组，用来存放真实的数据。比如我们 `remember` 的 `count` 变量、`Button` 和 `Text` 的引用、lambda 表达式等，都存在这里 。

当 `Counter` 函数首次执行时，`$composer` 就像一位精明的会计，在 `SlotTable` 里按顺序“记账” ：

* `$composer.startRestartGroup()` 在 `groups` 数组中创建一个新的 `Group` 条目。
* `$composer.cache()` 负责在 `slots` 数组中存储 `count` 这个可变状态。
* 调用 `Button` 和 `Text` 时，会重复上述过程，形成嵌套的 `Group` 结构。`Button` 的 `Group` 会成为 `Counter` `Group` 的子节点，`Text` 的 `Group` 又成为 `Button` `Group` 的子节点，最终在 `SlotTable` 中构建出一棵完整的虚拟 UI 树。

  `SlotTable` 的妙处在于，它用**线性数组**模拟了树形结构，既保留了树的语义，又获得了数组的高效访问速度，为后续快速的重组和布局绘制打下了坚实基础 。

### 布局阶段

`SlotTable` 中记录的 `LayoutNode` 会被用来构建一棵真正用于渲染的 `LayoutNode` 树 。布局阶段就是遍历这棵树，确定每个节点的大小和位置。

我们以 `Button` 为例，它内部最终会调用 `Layout` 可组合函数 ：

```
@Composable inline fun Layout(
    content: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    // ... 获取资源 ...
    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = ComposeUiNode.Constructor, // 创建 LayoutNode 的工厂
        update = {
            set(measurePolicy, ComposeUiNode.SetMeasurePolicy) // 设置测量策略
            // ... 设置其他参数 ...
        },
        content = content
    )
}
```

`Layout` 的核心是创建并配置一个 `LayoutNode`，并为其指定一个 **`measurePolicy`**。这个 `measurePolicy` 定义了该节点如何进行测量和摆放。以 `Column` 的 `measurePolicy` 为例，它的逻辑大致如下 ：

```
// 简化的 Column 测量策略
MeasurePolicy { measurables, constraints ->
    // 1. 测量阶段：遍历所有子节点 (measurables)，并测量它们
    val placeables = measurables.map { measurable ->
        measurable.measure(constraints)
    }
  
    // 2. 布局阶段：计算自身高度 (所有子节点高度之和) 和宽度 (最宽的子节点)
    layout(constraints.maxWidth, totalHeight) {
        var y = 0
        // 3. 摆放阶段：将每个子节点按顺序放置在垂直方向上
        placeables.forEach { placeable ->
            placeable.place(x = 0, y = y)
            y += placeable.height
        }
    }
}
```

整个过程和我们熟悉的 `View` 的 `onMeasure` 和 `onLayout` 非常相似，但 Compose 将其提升到了纯 Kotlin 代码的层面，更灵活，也更容易实现自定义布局 。

### 绘制阶段

布局完成后，就到了最后一步：绘制。Compose 再次遍历 `LayoutNode` 树，这次是调用每个节点的绘制方法。

以上面的 `Text` 为例，它的绘制逻辑最终会走到底层的 `Canvas` 绘制 API。在 Compose 中，我们可以通过 `Canvas` 可组合函数或 `Modifier.drawWithContent` 等修饰符来直接控制绘制 。

```
// 一个使用 Canvas 的例子
Canvas(modifier = Modifier.size(100.dp)) {
    drawCircle(Color.Red, radius = 50f)
}
```

当 `LayoutNode` 执行绘制时，Compose 会将我们定义的高级绘制指令（如 `drawCircle`）转换成底层的平台绘制调用（在 Android 上就是 `Canvas.drawCircle`），最终将内容渲染到屏幕上。

这个过程对应了传统 `View` 系统的 `onDraw` 阶段。

### 状态更新触发重组

最后，也是 Compose 最迷人的部分。当用户点击按钮，`count++` 执行时，发生了什么？

1. `mutableStateOf(0)` 创建的 `State` 对象的值发生了变化。
2. Compose 的**快照系统 (Snapshot system)** 会检测到这个变化，并找到所有读取过这个 `State` 的重组作用域 。
3. 在我们的例子中，读取 `count` 的地方在 `Counter` 函数体内，它所在的 `RestartGroup` 被标记为“无效”。
4. Compose 调度器会在下一帧到来前，再次调用 `Counter` 函数进行**重组**。
5. 重组时，`$composer` 再次与 `SlotTable` 交互。由于 `count` 变了，`if` 判断条件为真，函数体会被执行。
6. Compose 在执行 `Button` 和 `Text` 时，会比较 `SlotTable` 中已有的内容和当前执行的差异。它发现 UI 结构没有变化（没有增加或移除控件），只是 `Text` 的 `text` 参数从 "Clicked 0 times" 变成了 "Clicked 1 times"。因此，它不会重建整个 `LayoutNode` 树，而是高效地只更新 `Text` 节点上需要重新绘制的部分。
7. 随后，**布局**和**绘制**阶段再次启动，屏幕上就显示了最新的计数 。

这个过程，就是 Compose 声明式 UI 的核心：**你只管改状态，UI 自动响应**。

## 关于`mutableStateOf`

前面执行count++的时候，实际操作的是SnapshotMutableStateImpl的getter和setter。

### 关于SnapShot

快照是 Compose 实现隔离、并发的核心机制。每个读/写操作都在一个快照中进行。活跃的快照可以通过 `Snapshot.current` 访问。这是通过ThreadLocal实现的。

* 当你在 Composable 函数中读取 `count`（即执行 `count` 的 getter）时，Compose 会调用 `Snapshot.current.recordRead`，将当前读取的状态和读取者（重组作用域）关联起来。**这样就建立了“状态 → 重组作用域”的订阅关系。**
* 当你修改 `count`（执行 `count++` 触发 setter）时，会调用 `Snapshot.current.recordModified`，通知快照系统该状态已改变。

### 如何判断是否需要重组？

1. 在组合（Composition）阶段，Composer 会执行 Composable 函数。当 Composer 遇到读取某个 `MutableState` 的代码时，它会通过 `Snapshot.current.recordRead` 将这个**读取操作**与当前正在执行的**重组作用域**（即当前的 Composable 函数对应的 Group）绑定在一起。

这里的“重组作用域”就是我们在上一篇源码分析中看到的 `startRestartGroup` 和 `endRestartGroup` 包裹的范围。每个可组合函数或代码块都可能被包裹成一个作用域。

2. 当 `count.value` 被修改时（setter 被调用），`Snapshot.current.recordModified` 会通知快照系统：这个状态变了。快照系统会查找所有之前读取过该状态的重组作用域，并将它们标记为“无效”（invalid）。
3. Compose 的调度器会在下一帧之前，收集所有被标记为无效的作用域，并按照一定的优先级重新执行它们（即重组）。重组的执行会再次进入 Composable 函数，但此时 Compose 会利用 `SlotTable` 中保存的信息，只更新真正变化的部分。
4. 在重组过程中，Composer 会检查每个作用域的输入参数（包括读取的状态）是否发生了变化。如果没有变化，Composer 可以跳过整个作用域的执行，直接复用之前的 `SlotTable` 内容。这就是 Compose 高效的核心——**细粒度的重组 + 智能跳过**。

### 源码层面的关键类

1. SnapShot

* `current`: `ThreadLocal<Snapshot>`，每个线程有自己的快照。
* `recordRead(state: StateObject)`：记录对 `state` 的读取，将当前线程的读取操作与状态关联。
* `recordModified(state: StateObject)`：记录状态的修改，并触发后续的观察者（即重组作用域）的失效。

2. Composer

* `startRestartGroup()` / `endRestartGroup()`：定义一个可重组的作用域。
* 内部维护了当前正在执行的组合信息，并与快照系统交互。

3. Recomposer

* 负责管理重组队列和调度重组。

4. StateObject & StateRecord

* 实现多版本并发控制，允许多个快照同时存在（例如后台预重组）。

### startRestartGroup和endRestartGroup是怎么定义重组作用域的？

`Composer` 的核心操作对象是 `SlotTable`。`SlotTable` 可以理解为一个由两个并行数组构成的“账本”：

* **`groups` 数组**：记录每个“组”的元信息（类型、在 slots 中的起始位置、节点数量等）。
* **`slots` 数组**：记录组内的具体数据（状态、可组合项引用、子组位置等）。

**startRestartGroup干了什么？**

当调用 `startRestartGroup()` 时，`Composer` 会在 `groups` 数组中**追加一条新记录**。这条记录描述了即将开始的组的各种属性，比如：

* **组类型**：标记这是一个可重组的组（`RESTART_GROUP`）。
* **组在 `slots` 中的起始索引**：方便后续定位数据。
* **父组信息**：通过栈或链表维护嵌套关系。

同时，`Composer` 内部维护了一个“组栈”，将当前组压入栈顶，这样后续的代码就知道自己属于哪个组。

这个过程类似于在内存中**创建一个容器**，并告诉系统：“从这里开始，接下来的所有代码和产生的数据，都属于这个新的容器。”

**endRestartGroup干了什么？**

当执行到 `endRestartGroup()` 时，`Composer` 会：

1. 从组栈中弹出当前组。
2. 在 `groups` 数组中**补全该组的完整信息**，例如该组在 `slots` 中占用了多少个位置、子节点的数量等。
3. 返回一个可选的 **`ScopeUpdateScope`** 对象，该对象包含了这个组对应的重组作用域的更新回调。

这个返回的 `ScopeUpdateScope` 非常重要，它相当于一个“遥控器”，Compose 运行时可以通过它来**标记该组需要重组**，并指定重组时应该调用哪个函数（即我们 `Counter` 函数本身）。

**作用域是如何被包裹起来的**

你可能想问：“一条 `groups` 记录怎么就能包裹一个作用域呢？”关键在于 `groups` 数组不仅记录了组的开始，还通过 `slots` 数组记录了组内产生的所有数据。更重要的是，`groups` 数组记录了每个组的**子节点数量**和**数据占用长度**，从而形成了一个**闭区间**。

例如，假设我们的 `Counter` 组在 `groups` 中的索引是 `g`，那么 `groups[g]` 存储了该组的起始信息，而通过后续的一些字段（如 `size`），系统就能知道这个组在 `groups` 数组中占据了多少个条目（通常一个组在 `groups` 中占用连续多个条目来存储完整信息），以及在 `slots` 数组中对应了哪一段连续的区域。

这样一来，当需要重组时，`Composer` 只需要找到这个组对应的 `groups` 条目，然后根据记录的信息，**重新执行该组范围内的所有代码**。因为组内所有的代码和产生的数据都已经被“框”在了这个连续的内存块中，所以执行起来非常高效，而且可以确保不干扰其他组。

**组与重组的关系**

当 `count` 这个 `MutableState` 发生变化时，快照系统会找到所有读取过它的组，并将这些组标记为“无效”（invalid）。这里的“组”就是通过 `startRestartGroup` 创建的那些组。

* 每个组都有一个唯一的 `key` 和一个关联的 `recomposeScope`。
* 当组被标记为无效时，Compose 调度器会在下一帧前，调用该组的 `recomposeScope` 所指向的更新函数，即之前 `updateScope` 中注册的 lambda。
* 这个 lambda 会再次调用我们的 `Counter` 函数，并传入一个新的 `Composer`，开始新一轮的组合。

由于组在 `SlotTable` 中的信息是完整且连续的，重组时 Compose 可以直接定位到组在 `groups` 和 `slots` 中的位置，**复用已有的存储空间**，只更新变化的部分。这就实现了细粒度的重组。

### 流程示例

```
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Clicked $count times")
    }
}
```

**首次组合：**

1. 执行 `Counter()`，Composer 调用 `startRestartGroup()` 标记一个作用域。
2. 执行 `remember { mutableStateOf(0) }`，创建 `SnapshotMutableStateImpl`。
3. 执行 `Button` 和 `Text`，Composer 递归调用它们的代码。
4. 当执行到 `Text` 时，需要读取 `count`（因为字符串模板中用了 `$count`），此时 `count` 的 getter 被调用。
5. getter 内部调用 `Snapshot.current.recordRead(this)`，快照系统记录：“状态 X 被作用域 S（当前 Composable 对应的 Group）读取了”。
6. 组合结束，UI 显示出来。

**点击按钮：**

1. 执行 `count++`，即 `count.value = count.value + 1`。
2. setter 内部调用 `Snapshot.current.recordModified(this)`，快照系统标记“状态 X 被修改了”。
3. 快照系统查找所有之前读取过状态 X 的作用域，发现作用域 S，将其标记为无效。
4. 调度器在下一帧触发重组，重新执行作用域 S 内的代码（即 `Counter` 函数的一部分，可能只重新执行 `Text` 部分）。
5. 重组过程中，再次读取 `count` 的新值，UI 更新。

//LaunchedEffec

//rememberCoroutineScope

//DisposableEffect
