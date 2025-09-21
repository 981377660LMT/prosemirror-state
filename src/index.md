好的，我们来系统性地、有逻辑地讲解 prosemirror-state 这个至关重要的模块，并为您规划一条清晰的代码阅读顺序。

prosemirror-state 是 ProseMirror 编辑器的“大脑”和“中枢神经系统”。它定义了编辑器的**唯一数据源 (Single Source of Truth)**，并管理着从一个状态到下一个状态的所有变化。理解它，是掌握 ProseMirror 工作流的关键。

### 宏观逻辑：状态、变化与插件

prosemirror-state 的设计哲学可以概括为三个核心概念：

1.  **不可变的状态 (Immutable State)**: 编辑器的所有状态——包括文档内容、选区、激活的标记、以及插件的自定义数据——都被封装在一个单一的、不可变的对象 `EditorState` 中。你**永远不会直接修改**一个 `EditorState` 对象。
2.  **事务驱动的变化 (Transaction-Driven Changes)**: 任何对状态的修改都必须通过创建一个 **`Transaction`** 来完成。一个 `Transaction` 对象描述了从一个旧状态到新状态的**所有变化**，包括文档修改（它继承自 `Transform`）、选区变化、以及其他元数据。当一个 `Transaction` 被应用 (`apply`) 到一个 `EditorState` 时，它会生成一个**全新的** `EditorState` 对象。
3.  **可扩展的插件系统 (Extensible Plugin System)**: ProseMirror 的大部分功能，包括历史记录、键盘绑定、甚至是光标的显示，都是通过 **`Plugin`** 实现的。插件可以监听 `Transaction`，可以拥有自己的私有状态（存储在 `EditorState` 中），还可以向编辑器视图（View）注入行为。

这个模型（灵感来源于 Redux）带来了巨大的好处：可预测性、可追溯性（因为每个状态都是一个快照）、以及强大的功能（如协同编辑和时间旅行调试）。

---

### 核心组件与代码阅读顺序

根据上述逻辑，我们从最核心的 `EditorState` 开始，然后理解驱动它变化的 `Transaction`，最后看如何通过 `Plugin` 来扩展它。

#### 第 1 站：`EditorState` - 唯一的数据源

**目标**：理解编辑器的完整状态是如何被组织的。

**阅读文件**：`state.ts`

`EditorState` 是所有信息的聚合体。

**关键点**：

- **核心属性**:
  - `doc: Node`: 当前的文档内容，来自 prosemirror-model。
  - `selection: Selection`: 当前的选区。
  - `storedMarks: readonly Mark[] | null`: 当选区为空（光标状态）时，下次输入将带有的激活标记。
  - `plugins: readonly Plugin[]`: 当前激活的插件列表。
- **不可变性**: 注意 `EditorState` 的属性都是只读的。
- **创建**: `EditorState.create(config)` 是创建初始状态的入口。它接收一个 `EditorStateConfig` 对象，其中包含初始的 `doc` (或 `schema`) 和 `plugins`。
- **演变**: `state.apply(tr)` 是状态演变的核心方法。它接收一个 `Transaction`，然后返回一个**新的** `EditorState` 实例。这是整个状态管理的心跳。
- **`tr` getter**: `state.tr` 是一个非常方便的快捷方式，它等同于 `new Transaction(state)`，用于开始一个新的变换。
- **插件状态**: `EditorState` 实例上还会挂载由插件定义的自定义状态字段。例如，如果一个插件的 `key` 是 `"myPlugin$"`，那么你可以通过 `state.myPlugin$` 来访问它的状态。

#### 第 2 站：`Transaction` - 变化的描述者

**目标**：理解从一个状态到另一个状态的所有变化是如何被打包和描述的。

**阅读文件**：`transaction.ts`

`Transaction` 是 prosemirror-transform 中 `Transform` 的一个**超集**。它不仅包含了对文档的修改（继承自 `Transform` 的 `steps` 和 `docs`），还包含了其他非文档状态的变化。

**关键点**：

- **继承关系**: `export class Transaction extends Transform`。这意味着一个 `Transaction` 对象拥有所有 `Transform` 的方法，如 `replace`, `split`, `addMark` 等。
- **状态变化**:
  - `setSelection(selection)`: 明确地设置一个新的选区。如果没有调用，选区会根据文档的变化自动映射 (`map`)。
  - `setStoredMarks(marks)`: 设置激活的标记。
  - `scrollIntoView()`: 一个标志，告诉视图（View）在更新后需要将选区滚动到可见区域。
- **元数据 (Metadata)**:
  - `setMeta(key, value)` 和 `getMeta(key)`: 这是插件间通信和向系统传递信息的关键机制。插件可以给 `Transaction` 附加任意的元数据，其他插件可以在处理流程中读取这些元数据来决定自己的行为。例如，历史记录插件会检查一个 `Transaction` 是否有 `"addToHistory": false` 的元数据来决定是否要记录这次变化。
- **`docChanged`**: 一个简单的 getter，判断这次 `Transaction` 是否包含了文档修改的 `Step`。

#### 第 3 站：`Selection` - “焦点”的抽象

**目标**：理解编辑器中的选区是如何被表示和操作的。

**阅读文件**：`selection.ts`

`Selection` 是一个抽象基类，定义了所有选区类型的通用接口。

**关键点**：

- **核心子类**:
  - `TextSelection`: 最常见的选区，有一个固定的 `anchor` 和一个可移动的 `head`。当 `anchor` 和 `head` 相同时，它就是一个光标。
  - `NodeSelection`: 选中一个单一的、完整的节点（例如一张图片）。
  - `AllSelection`: 选中整个文档。
- **`map(doc, mapping)`**: `Selection` 最重要的方法之一。当文档发生变化时（由一个 `mapping` 描述），这个方法计算出选区在新文档中的对应位置。
- `toJSON()` 和 `Selection.fromJSON()`: 用于序列化和反序列化选区，是保存和恢复状态的一部分。
- `SelectionBookmark`: 一个轻量级的、文档无关的选区“书签”。当历史记录需要撤销/重做时，它存储的是 `Bookmark` 而不是完整的 `Selection` 对象，通过 `bookmark.resolve(doc)` 可以在新文档上恢复选区。

#### 第 4 站：`Plugin` - 功能的扩展器

**目标**：理解 ProseMirror 的功能是如何通过插件机制来构建和组织的。

**阅读文件**：`plugin.ts`

`Plugin` 是 ProseMirror 模块化设计的核心。几乎所有高级功能都是通过插件实现的。

**关键点**：

- **`PluginSpec`**: 创建插件时传入的配置对象，它定义了插件的所有行为。
- **`state: StateField`**: 这是插件拥有自己状态的核心。
  - `init(config, state)`: 初始化插件的状态。
  - `apply(tr, value, oldState, newState)`: 这是插件状态更新的逻辑。每当一个 `Transaction` 应用时，这个函数会被调用，它接收旧的插件状态 `value` 和 `tr`，然后计算并返回**新的**插件状态。
- **`props: EditorProps`**: 插件可以通过这个属性向 `EditorView` 注入“属性”（props），例如 `handleKeyDown`, `handleClick`, `decorations` 等，从而实现对用户交互的响应和对视图的渲染控制。
- **`appendTransaction`**: 插件的“反应”机制。在一个或多个 `Transaction` 被应用后，这个钩子会被触发，允许插件根据刚刚发生的变化，创建并追加一个新的 `Transaction`。例如，当用户输入 `(c)` 时，一个插件可以在 `appendTransaction` 中检测到这个变化，并创建一个新的 `Transaction` 将其替换为 `©`。
- **`filterTransaction`**: 在一个 `Transaction` 被应用**之前**提供一个“否决权”。如果任何一个插件的 `filterTransaction` 返回 `false`，整个 `Transaction` 就会被取消。
- **`PluginKey`**: 一个唯一的键，用于标识一类插件。它允许你在不知道插件实例的情况下，从 `EditorState` 中安全地获取插件或其状态（`key.get(state)` 或 `key.getState(state)`）。

### 总结与回顾

1.  **起点**: `index.ts` - 快速浏览模块导出的所有核心类：`EditorState`, `Transaction`, `Selection`, `Plugin`, `PluginKey`。
2.  **核心数据结构**: `state.ts` - 理解 `EditorState` 的构成和 `apply` 方法。
3.  **变化单元**: `transaction.ts` - 理解 `Transaction` 如何继承 `Transform` 并添加了 `selection` 和 `meta` 等状态。
4.  **焦点**: `selection.ts` - 了解不同类型的选区及其 `map` 方法。
5.  **扩展机制**: `plugin.ts` - 这是最复杂但也是最强大的部分。重点理解 `PluginSpec` 中的 `state`, `props`, `appendTransaction` 和 `key`。

遵循这个顺序，您将建立起一个关于 ProseMirror 状态管理的清晰心智模型：**`EditorState` 是静态的快照，`Transaction` 是动态的变化描述，而 `Plugin` 则是响应这些变化并创造新变化的规则引擎。**
