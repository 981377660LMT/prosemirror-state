好的，我们来深入、详细地讲解 state.ts 这个文件，特别是 `EditorState` 类。这是整个 prosemirror-state 模块的基石，也是任何 ProseMirror 编辑器的**唯一数据源 (Single Source of Truth)**。

理解 `EditorState` 的关键在于掌握它的两个核心特性：

1.  **不可变性 (Immutability)**: `EditorState` 对象是持久化的数据结构。你永远不会直接修改一个 `EditorState` 实例。相反，你会通过应用一个 `Transaction` 来计算并生成一个**全新的** `EditorState` 实例。这使得状态管理、历史追溯（撤销/重做）和协同编辑变得极其可靠和可预测。
2.  **中心化 (Centralization)**: 它包含了描述编辑器 UI 所需的所有信息：文档内容 (`doc`)、用户选区 (`selection`)、激活的插件 (`plugins`) 以及由这些插件管理的各种状态（如历史记录、装饰等）。

---

### 第一部分：`EditorState` 的核心构成 - `FieldDesc` 和 `baseFields`

`EditorState` 本身是一个容器，它的具体内容由一系列“状态字段 (State Fields)”定义。`FieldDesc` 类就是这些字段的描述符。

```typescript
// ...existing code...
class FieldDesc<T> {
  init: (config: EditorStateConfig, instance: EditorState) => T
  apply: (tr: Transaction, value: T, oldState: EditorState, newState: EditorState) => T

  constructor(readonly name: string, desc: StateField<any>, self?: any) {
// ...existing code...
```

每个 `FieldDesc` 都有两个关键方法：

- `init`: 如何在创建全新的 `EditorState` 时**初始化**这个字段的值。
- `apply`: 当一个 `Transaction` 应用时，如何根据旧的状态值和 `Transaction` 信息，计算出这个字段在**新状态**中的值。

ProseMirror 内置了四个基础字段：

1.  **`doc`**: 文档内容 (`Node`)。它的 `apply` 方法非常简单：`return tr.doc`。因为 `Transaction` 的结果就是新文档。
2.  **`selection`**: 用户选区 (`Selection`)。同样，它的 `apply` 方法是 `return tr.selection`。
3.  **`storedMarks`**: 存储的标记 (`Mark[]`)。当光标在一个没有文本的位置时（例如，在一个空段落的开头），ProseMirror 会用 `storedMarks` 来记住用户激活的标记（如加粗、斜体），以便下次输入时应用。它的 `apply` 方法会检查 `tr.storedMarks`。
4.  **`scrollToSelection`**: 一个计数器，每当 `Transaction` 包含“滚动到视图”的元信息时，它就会加一。UI 层可以监听这个值的变化来触发滚动。

**插件也可以定义自己的状态字段**。例如，`history` 插件会添加一个 `history` 字段来存储撤销栈。这些自定义字段与基础字段一视同仁，都通过 `FieldDesc` 进行管理。

---

### 第二部分：创建与配置 - `create()` 和 `Configuration`

#### `EditorState.create(config)`

这是创建编辑器初始状态的唯一入口。它接收一个 `EditorStateConfig` 对象。

1.  **`new Configuration(...)`**: 它首先创建一个内部的 `Configuration` 对象。这个对象是**跨状态共享**的，它只在 `create` 或 `reconfigure` 时被创建。
    - `Configuration` 的构造函数会解析传入的 `plugins` 数组。
    - 它将 `baseFields` 和所有插件定义的 `state` 字段收集到 `this.fields` 数组中。
    - 它还会建立一个按 `key` 索引的插件映射 `pluginsByKey`，用于快速查找。
2.  **`new EditorState($config)`**: 创建 `EditorState` 实例，并将 `Configuration` 对象存入其 `config` 属性。
3.  **初始化字段**: 循环遍历 `Configuration` 中的所有 `fields`，并调用每个字段的 `init` 方法来计算初始值，然后将该值直接赋给 `instance` 的同名属性（例如 `instance.doc = ...`, `instance.selection = ...`）。

最终，`EditorState.create` 返回一个被完全初始化的、可用的状态对象。

---

### 第三部分：核心生命周期：更新状态 - `apply` 和 `applyTransaction`

这是 `EditorState` 最核心、最复杂的部分。

#### `applyInner(tr)` - 纯粹的状态计算

这是一个内部方法，它执行最纯粹的“状态转移”计算。

1.  `new EditorState(this.config)`: 创建一个新的、空的 `EditorState` 实例，复用旧状态的 `config`。
2.  **应用字段**: 遍历 `config.fields`，对于每个字段，调用其 `apply` 方法：`field.apply(tr, this[field.name], this, newInstance)`。
3.  `apply` 方法会返回该字段在新状态中的值，这个值被赋给新实例的同名属性。
4.  循环结束后，`newInstance` 就是一个应用了 `tr` 之后全新的状态。

#### `apply(tr)` - 简单的应用入口

这是一个便捷的公共 API，它直接调用 `this.applyTransaction(tr).state`，只返回最终的新状态，忽略中间过程。

#### `applyTransaction(rootTr)` - 完整的“事务循环”

这是 ProseMirror 插件系统强大功能的体现，它允许插件对事务进行过滤、响应甚至追加新的事务。

**这个过程可以理解为一个“反馈循环”**:

1.  **过滤 (`filterTransaction`)**: 首先，它会询问所有插件：“你们是否允许这个 `rootTr` 通过？”。任何一个插件的 `filterTransaction` 钩子返回 `false`，整个应用过程就会被中止。

2.  **应用初始事务**: 如果 `rootTr` 通过了过滤，就调用 `applyInner(rootTr)` 计算出一个临时的 `newState`。

3.  **进入反馈循环 (`for(;;)`):**
    a. **追加 (`appendTransaction`)**: 现在，它会遍历所有插件，并调用它们的 `appendTransaction` 钩子。这个钩子接收到目前为止发生的所有事务（`trs`），以及旧状态和新状态。
    b. 插件可以在这里进行响应。例如，`history` 插件看到一个撤销事务，它可能会创建一个新的事务来将文档恢复到上一个版本。
    c. 如果一个插件返回了一个新的事务 `tr`，这个新事务也会经过 `filterTransaction` 的过滤。
    d. 如果通过，这个新事务 `tr` 会被 `push` 到 `trs` 数组中，并再次调用 `applyInner(tr)` 来更新 `newState`。
    e. **循环**: 因为有新的事务被追加了，所以整个循环会重新开始。它会再次遍历所有插件，询问它们是否要对**新追加的**事务做出反应。
    f. 这个过程会一直持续，直到某一次完整的循环中，没有任何插件再追加新的事务为止。

4.  **返回结果**: 循环结束后，返回最终的 `newState` 和整个过程中产生的所有事务 `trs`。

这个机制使得插件之间可以解耦，但又能相互协作，创造出非常强大的功能。

---

### 第四部分：其他重要方法

- **`tr` (getter)**: 一个极其方便的属性，`state.tr` 等同于 `new Transaction(state)`，是开始一次新变换最常用的方式。
- **`reconfigure(config)`**: 允许你基于当前状态，用一套新的插件来创建一个新状态。它会智能地保留那些在新旧插件集中都存在的插件的状态。
- **`toJSON()` / `fromJSON()`**: 提供了将状态序列化为 JSON 和从 JSON 反序列化的能力。这对于保存文档或在服务器和客户端之间传递状态至关重要。插件也可以通过实现自己的 `toJSON` 和 `fromJSON` 方法来参与这个过程。

### 总结

- `EditorState` 是 ProseMirror 的核心，它是一个**不可变的、中心化的状态容器**。
- 它的内容由一系列**状态字段**定义，包括内置的 `doc`, `selection` 等，以及由插件贡献的自定义字段。
- 状态的创建通过 `EditorState.create()` 完成，它会初始化所有字段。
- 状态的更新通过 `applyTransaction()` 完成，这是一个强大的**反馈循环**，允许插件过滤、响应和追加事务，是 ProseMirror 插件系统交互的核心。
- `applyInner()` 是纯粹的状态计算函数，它通过调用每个字段的 `apply` 方法来生成一个全新的状态实例，完美体现了其**不可变性**。
