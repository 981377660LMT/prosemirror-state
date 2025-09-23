好的，我们来深入、详细地讲解 selection.ts 这个文件。这是 prosemirror-state 模块的另一个支柱，它定义了编辑器中“选区”这一核心概念。

与 `doc`（文档内容）一样，`selection` 也是 `EditorState` 的一个核心组成部分。它不仅告诉我们用户光标在哪里或选择了哪段文本，还驱动着大量的编辑行为。

---

### 第一部分：`Selection` - 抽象基类与核心概念

selection.ts 的核心是抽象类 `Selection`。它定义了所有选区类型都必须遵守的“契约”和共享的行为。直接实例化这个类是没有意义的，你必须使用它的一个具体子类。

#### 1. 核心属性：`anchor` 与 `head`

这是理解 ProseMirror 选区的最关键概念：

- **`anchor` (锚点)**: 选区的固定端。当你按住 `Shift` 键并移动光标时，`anchor` 就是你开始选择的地方，它保持不变。
- **`head` (头)**: 选区的活动端。它是随着你移动光标而变化的那一端。

`anchor` 和 `head` 的相对位置决定了选区的**方向**。

- 如果 `head < anchor`，选区是“反向”的（例如，从右向左选择文本）。
- 如果 `head > anchor`，选区是“正向”的。

#### 2. 派生属性：`from` 与 `to`

虽然 `anchor` 和 `head` 很重要，但在很多操作中，我们只关心选区所覆盖的范围，不关心其方向。

- **`from`**: `Math.min(anchor, head)`，选区范围的起始位置。
- **`to`**: `Math.max(anchor, head)`，选区范围的结束位置。

`from` 和 `to` 总是满足 `from <= to`。它们定义了选区的**主范围 (main range)**。

#### 3. `$` 前缀的“已解析”位置

你会看到 `$anchor`, `$head`, `$from`, `$to` 这些带有 `$` 前缀的属性。它们是 `ResolvedPos` 类型的实例。

- `anchor` (数字) 只是一个位置坐标。
- `$anchor` (`ResolvedPos`) 是一个包含了该位置所有上下文信息的对象，比如它所在的父节点、深度、路径等。这使得对选区进行复杂的结构分析和操作成为可能。

#### 4. 核心方法

- **`map(doc, mapping)`**: 这是选区最重要的抽象方法。当文档内容通过一个 `Transaction` 发生变化时，旧的选区坐标就失效了。`map` 方法的作用就是：接收新的文档 `doc` 和一个位置映射 `mapping`，然后计算并返回一个在**新文档中**有效的新 `Selection` 对象。这是 `EditorState` 不可变性的关键一环。
- **`eq(other)`**: 判断两个选区是否相等。
- **`content()`**: 获取选区所覆盖内容的 `Slice`。
- **`replace(tr, content)`**: 在一个事务 `tr` 中，用给定的 `content` (`Slice`) 替换选区的内容。这是实现删除（不提供 `content`）和粘贴的基础。
- **`toJSON()`**: 将选区序列化为 JSON 对象，用于保存状态或网络传输。

---

### 第二部分：具体的选区类型

ProseMirror 内置了三种具体的 `Selection` 子类，以应对不同的场景。

#### 1. `TextSelection` - 最常见的选区

- **描述**: 这是我们最熟悉的经典文本选区。它可以是一个**光标**（`anchor == head`），也可以是一段高亮的文本。
- **约束**: `TextSelection` 的 `anchor` 和 `head` **必须**指向拥有 `inlineContent` 的节点内部（通常是段落、标题等文本块）。
- **`$cursor`**: 这是一个非常有用的 getter。如果选区是一个光标（`empty` 为 `true`），它返回光标的 `$head` 位置 (`ResolvedPos`)；否则返回 `null`。这使得判断一个选区是否为光标变得非常简单。

#### 2. `NodeSelection` - 节点选区

- **描述**: 当你需要选中一个“原子”块级节点时，例如一张图片、一个视频嵌入或一个水平分割线，就会使用 `NodeSelection`。
- **结构**: 对于一个 `NodeSelection`，它的 `from` 指向该节点之前，`to` 指向该节点之后。`anchor` 等于 `from`，`head` 等于 `to`。
- **`isSelectable(node)`**: 一个静态方法，用于判断一个节点是否可以被节点选中。这通常由节点 `Schema` 中的 `selectable` 属性控制。
- **`visible = false`**: `NodeSelection` 在浏览器中通常没有可见的高亮区域，而是通过给选中的 DOM 节点添加一个特殊的 CSS 类（如 `ProseMirror-selectednode`）来在视觉上表示选中状态。

#### 3. `AllSelection` - 全文选区

- **描述**: 选中整个文档。这对于实现“全选” (`Cmd/Ctrl+A`) 功能至关重要。
- **结构**: 它的 `from` 永远是 `0`，`to` 永远是文档的末尾 (`doc.content.size`)。

---

### 第三部分：关键机制与辅助工具

#### 1. `Selection.jsonID` 和 `fromJSON` - 序列化与反序列化

- 为了能从 JSON 数据中恢复选区，每种选区类型都需要一个唯一的字符串 ID。
- `Selection.jsonID("text", TextSelection)` 就是在“注册”`TextSelection`，告诉系统，当 `toJSON()` 生成的 JSON 对象中 `type` 属性为 `"text"` 时，应该使用 `TextSelection.fromJSON` 这个静态方法来反序列化它。
- 这使得 ProseMirror 的选区系统是**可扩展的**。你可以创建自己的选区类型，并通过这个机制让它支持序列化。

#### 2. `SelectionBookmark` - 选区书签

- **问题**: 在实现撤销/重做功能时，历史插件不仅需要恢复文档内容，还需要恢复操作前的**选区**。但历史插件只存储 `Step` 和 `StepMap`，它没有完整的 `doc` 对象，因此无法直接存储一个完整的 `Selection` 对象。
- **解决方案**: `SelectionBookmark`。它是一个轻量级的、与文档无关的选区表示。
  - `getBookmark()`: 将当前选区转换为一个书签。对于 `TextSelection`，它只存储 `anchor` 和 `head` 两个数字。
  - `bookmark.map(mapping)`: 书签的核心能力。它可以像 `Selection` 一样通过 `mapping` 进行位置映射。
  - `bookmark.resolve(doc)`: 当需要恢复选区时，历史插件会取出映射后的书签，并用**当前**的 `doc` 对象来“解析”它，从而得到一个有效的 `Selection` 对象。

#### 3. 静态查找方法 - `findFrom`, `near`, `atStart`, `atEnd`

这些是强大的工具函数，用于在文档中寻找一个**有效**的选区位置。

- **背景**: 并非文档中的每个位置都是一个有效的选区位置。例如，你不能将光标放在 `<ul>` 和 `<li>` 标签之间。
- **`findFrom($pos, dir, textOnly)`**: 从 `$pos` 开始，沿着 `dir` 方向（`1` 为向前，`-1` 为向后）寻找第一个有效的选区位置。
- **`near($pos, bias)`**: 在 `$pos` 附近寻找一个有效的选区。这是最常用的“兜底”方法。当一次变换导致旧的选区位置变得不再合法时，ProseMirror 就会用 `Selection.near` 来为用户找到一个最接近的、合理的新光标位置。
- **`atStart(doc)` / `atEnd(doc)`**: 快速找到文档开头或结尾的第一个有效选区。

### 总结

- `Selection` 是一个**抽象概念**，由 `TextSelection`, `NodeSelection`, `AllSelection` 等具体类实现，以适应不同的交互场景。
- `anchor` 和 `head` 定义了选区的**方向和范围**，而 `from` 和 `to` 提供了无方向的范围表示。
- 选区是**不可变的**，通过 `map` 方法在文档变化后计算出新的选区，这是状态管理的核心。
- `SelectionBookmark` 机制解耦了历史记录与文档状态，使得**选区的撤销/重做**成为可能。
- 强大的**静态查找方法** (`near`, `findFrom` 等) 保证了编辑器在各种复杂的结构变化后，总能为用户提供一个合理的、有效的选区。
