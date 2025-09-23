好的，我们来深入、详细地讲解 transaction.ts 这个文件，特别是 `Transaction` 类。

在 prosemirror-transform 中，我们有 `Transform`，它是一个纯粹的、用于描述**文档内容**变化的工具。然而，一个完整的编辑器状态 `EditorState` 不仅仅包含文档，还包含选区、存储的标记、插件状态等。

`Transaction` 就是为了描述**从一个 `EditorState` 到下一个 `EditorState` 的完整变化**而设计的。它是一个更高级的抽象，是 prosemirror-state 的核心。

---

### 第一部分：`Transaction` 的核心定位 - “状态变化的描述符”

可以从以下几个角度理解 `Transaction`：

1.  **它是 `Transform` 的超集**: `export class Transaction extends Transform` 这行代码是理解它的关键。`Transaction` **继承**了 `Transform` 的所有能力，因此你可以对一个 `Transaction` 实例调用所有文档修改方法，如 `tr.replace(...)`, `tr.split(...)`, `tr.addMark(...)` 等。
2.  **它管理非文档状态**: 除了文档变化，`Transaction` 还负责记录和管理其他状态的变化，最主要的是：
    - **选区 (`Selection`)**: 用户可能只移动了光标，而没有修改文档。
    - **存储的标记 (`Stored Marks`)**: 用户可能激活了“加粗”但还未输入文本。
3.  **它是插件间通信的载体**: `Transaction` 拥有一个“元数据 (metadata)”存储区。插件可以通过 `tr.setMeta(...)` 将信息附加到事务上，其他插件或 UI 层可以通过 `tr.getMeta(...)` 读取这些信息。这是实现复杂交互（如协同编辑、评论功能）的关键。
4.  **它是 `Command` 的执行产物**: `Command`（命令）是 ProseMirror 中执行操作的标准模式。一个命令函数接收 `state` 和 `dispatch`。如果命令决定执行操作，它会构建一个 `Transaction`，然后调用 `dispatch(tr)` 将其分发出去。

---

### 第二部分：`Transaction` 的核心属性与方法

#### 1. 继承自 `Transform`

`Transaction` 自动获得了 `steps`, `docs`, `mapping`, `doc` 等属性，以及所有文档操作方法。这意味着你可以像操作 `Transform` 一样操作 `Transaction`。

#### 2. 选区管理 (`selection` 和 `setSelection`)

- **`get selection()`**: 这是一个非常智能的 getter。

  - 当你创建一个事务时 (`state.tr`)，它会记录下初始的选区 `curSelection`。
  - 之后，你可能会向事务中添加一些 `Step`（例如 `tr.delete(5, 10)`）。这些 `Step` 会改变文档，从而使旧的选区位置失效。
  - 当你访问 `tr.selection` 时，getter 会检查 `curSelectionFor < this.steps.length`，即“是否有新的 `Step` 被添加了，而我还没有更新选区？”
  - 如果是，它会用 `this.curSelection.map(...)` 将旧选区通过新增的 `Step` 所对应的 `Mapping` 进行映射，计算出它在当前文档中的新位置，然后缓存这个结果。
  - 这个**惰性计算 (lazy evaluation)** 的设计非常高效，避免了每次添加 `Step` 都重新计算选区。

- **`setSelection(selection)`**: 这个方法允许你**显式地**设置一个全新的选区。这会覆盖掉通过 `map` 自动计算的选区。例如，在一个操作后，你希望将光标移动到某个特定位置，就需要调用 `tr.setSelection(...)`。调用它会自动将 `storedMarks` 设为 `null`，因为显式移动选区通常意味着用户放弃了之前激活的标记。

#### 3. 存储标记管理 (`storedMarks` 和 `setStoredMarks`)

- `storedMarks` 属性用于记录用户希望在下一次输入时应用的标记。
- `setStoredMarks(marks)`: 显式设置存储的标记。
- `addStoredMark(mark)` / `removeStoredMark(mark)`: 在现有标记基础上进行增删的便捷方法。
- **自动重置**: 当你向事务中添加任何改变文档内容的 `Step` 时 (`addStep` 方法被重写)，`storedMarks` 会被自动设为 `null`。这是因为一旦内容发生变化，存储的标记通常就应该被应用或丢弃，而不是继续“悬浮”着。

#### 4. 元数据 (`setMeta` 和 `getMeta`)

这是 `Transaction` 作为通信载体的核心。

- **`setMeta(key, value)`**: 将任意数据 `value` 存储在事务中，用一个 `key` 来标识。这个 `key` 可以是一个字符串，也可以是一个 `Plugin` 或 `PluginKey` 实例（内部会使用 `plugin.key` 作为真正的键）。使用 `PluginKey` 作为键可以有效避免命名冲突。
- **`getMeta(key)`**: 读取存储的元数据。

**常见用例**:

- **历史插件**: 当用户执行撤销操作时，历史插件会创建一个事务，并附加元数据 `tr.setMeta("addToHistory", false)`，告诉历史插件本身：“不要把这个撤销操作本身再记录到历史记录里了！”
- **协同编辑插件**: 当收到来自服务器的步骤时，协同编辑插件会创建一个事务，并附加元数据 `tr.setMeta("isRemote", true)`，以便其他部分（如历史记录）可以区分本地用户操作和远程同步操作。
- **UI 事件**: ProseMirror 视图层会附加 `"pointer": true` 来标记由鼠标点击产生的选区变化，或 `"uiEvent": "paste"` 来标记粘贴操作。

#### 5. UI 信号 (`scrollIntoView`)

- **`scrollIntoView()`**: 这是一个信号方法。调用它并不会真的执行滚动，它只是在事务的 `updated` 位掩码中设置一个标志位 `UPDATED_SCROLL`。
- 当 `EditorView` 接收到一个带有这个标志位的事务并更新其状态后，它会检查 `tr.scrolledIntoView`，如果为 `true`，`EditorView` 就会负责执行将选区滚动到可见区域的 DOM 操作。
- 这是一种非常好的解耦设计：状态管理（prosemirror-state）只负责发出“意图”，而 UI 渲染（prosemirror-view）负责实现这个意图。

### 总结

- `Transaction` 是 `EditorState` 变化的**完整描述**，它在 `Transform` 的基础上增加了对**选区**、**存储标记**和**元数据**的管理。
- 它的 `selection` getter 使用**惰性映射**来高效地追踪选区在文档变化中的位置。
- 它的 `meta` 属性是插件系统实现复杂交互和通信的**关键机制**。
- 它通过 `scrollIntoView` 等方法向 UI 层发出**意图信号**，实现了状态逻辑与视图逻辑的解耦。
- 在 ProseMirror 中，你永远不应该直接修改 `EditorState`。相反，你应该创建一个 `Transaction` (`state.tr`)，在上面记录下你所有的修改意图（包括文档、选区、元数据等），然后将这个完整的 `Transaction` 对象分发出去，由 `state.applyTransaction` 来计算出最终的新状态。
