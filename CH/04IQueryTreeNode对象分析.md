# IQueryTreeNode 对象分析

## 概述

`IQueryTreeNode` 是 ClickHouse 查询树（Query Tree）语义表示中的**基类**，位于 `src/Analyzer/` 目录下。它是查询树（Query Tree）的核心抽象，作为 AST（抽象语法树）的语义化升级版本存在。

> 查询树（Query Tree）是查询的语义表示，而 AST 是语法表示。Query Tree 在 AST 之上增加了语义信息（类型、依赖关系等），并且必须可逆地转换回 AST。

---

## 1. 类继承关系

```cpp
class IQueryTreeNode : public TypePromotion<IQueryTreeNode>
```

- 继承自 `TypePromotion` —— 一个 CRTP 模式模板类，提供安全的向下转型能力（如 `as<T>()`、`asType<T>()`）

---

## 2. 核心类型定义

### 2.1 枚举：`QueryTreeNodeType`

标识 17 种不同的节点类型：

| 类型 | 说明 |
|------|------|
| `IDENTIFIER` | 标识符（表名、列名等） |
| `MATCHER` | 模式匹配相关 |
| `TRANSFORMER` | 变换节点 |
| `LIST` | 列表节点 |
| `CONSTANT` | 常量 |
| `FUNCTION` | 函数调用 |
| `COLUMN` | 列 |
| `LAMBDA` | Lambda 表达式 |
| `SORT` | 排序 |
| `INTERPOLATE` | 插值 |
| `WINDOW` | 窗口函数 |
| `TABLE` | 表 |
| `TABLE_FUNCTION` | 表函数 |
| `QUERY` | 查询（SELECT/子查询） |
| `ARRAY_JOIN` | ARRAY JOIN |
| `JOIN` | JOIN |
| `UNION` | UNION |

### 2.2 智能指针别名

```cpp
using QueryTreeNodePtr = std::shared_ptr<IQueryTreeNode>;
using QueryTreeNodes = std::vector<QueryTreeNodePtr>;
using QueryTreeNodeWeakPtr = std::weak_ptr<IQueryTreeNode>;
using QueryTreeWeakNodes = std::vector<QueryTreeNodeWeakPtr>;
```

- **强引用** `shared_ptr`：表示所有权关系，父节点拥有子节点
- **弱引用** `weak_ptr`：用于表示"引用"关系（如列引用其来源表），不增加引用计数，避免循环引用

---

## 3. 核心成员变量

### 3.1 公有成员（从 protected 继承）

| 成员 | 类型 | 说明 |
|------|------|------|
| `children` | `QueryTreeNodes` | 子节点列表（`shared_ptr` 数组） |
| `weak_pointers` | `QueryTreeWeakNodes` | 弱引用指针列表（指向其他节点的非拥有引用） |

### 3.2 私有成员

| 成员 | 类型 | 说明 |
|------|------|------|
| `alias` | `String` | 节点别名（可被查询 pass 替换） |
| `original_alias` | `String` | 原始别名（来自 SQL 查询，不可变，用于 `additional_table_filters`） |
| `original_ast` | `ASTPtr` | 原始 AST 指针（不可修改的原始语法表示） |

---

## 4. 核心接口分析

### 4.1 类型识别（纯虚函数）

```cpp
virtual QueryTreeNodeType getNodeType() const = 0;
const char * getNodeTypeName() const;
```

- 子类必须实现 `getNodeType()` 返回自己的节点类型
- `getNodeTypeName()` 调用 `toString()` 将枚举转换为字符串

### 4.2 类型系统接口

```cpp
virtual DataTypePtr getResultType() const;
virtual void convertToNullable();
```

- **`getResultType()`**：获取节点的结果数据类型（用于表达式节点，如常量、函数、列），非表达式节点调用则抛异常
- **`convertToNullable()`**：将结果类型转换为可空类型，用于 `NULL` 值传播处理
- > TODO 注释提到：这些方法未来可能放入 `ExpressionQueryTreeNode` 子类

### 4.3 树比较与哈希

```cpp
struct CompareOptions {
    bool compare_aliases = true;
    bool compare_types = true;
};

bool isEqual(const IQueryTreeNode & rhs, CompareOptions ...) const;
Hash getTreeHash(CompareOptions ...) const;
```

- **`isEqual`**：递归比较两棵查询树是否相等，可以控制是否比较别名和类型
- **`getTreeHash`**：计算整棵树的哈希值（`uint128`），用于快速比较、去重、缓存等
- 两者均通过委托子类的 `isEqualImpl` / `updateTreeHashImpl` 实现

> 注意：原始 AST（`original_ast`）**不参与** `isEqual` 和 `getTreeHash` 的比较/哈希。

### 4.4 克隆与替换

```cpp
QueryTreeNodePtr clone() const;
QueryTreeNodePtr cloneAndReplace(const ReplacementMap & replacement_map) const;
QueryTreeNodePtr cloneAndReplace(const QueryTreeNodePtr & node_to_replace, QueryTreeNodePtr replacement_node) const;
```

- **`clone()`**：深拷贝整棵子树
- **`cloneAndReplace`**：深拷贝的同时，支持将特定节点替换为其他节点
  - 重载1：传入 `ReplacementMap`（节点→替换节点的映射表）
  - 重载2：传入单对替换关系

> 这是实现查询优化 Pass（如表达式简化、谓词下推）的关键机制。

### 4.5 别名管理

```cpp
bool hasAlias() const;
const String & getAlias() const;           // 获取当前别名
const String & getOriginalAlias() const;    // 获取原始别名
void setAlias(String alias_value);          // 设置别名，保存原始别名
void removeAlias();                         // 清除别名
```

**设计要点**：
- 原始别名 `original_alias` 在第一次 `setAlias` 时被保存，之后不会改变
- `getOriginalAlias()` 返回原始别名（如果存在），否则返回当前别名
- 这种设计支持查询 pass 修改别名，同时保留原始信息用于 `additional_table_filters`

### 4.6 AST 转换

```cpp
struct ConvertToASTOptions {
    bool add_cast_for_constants = true;
    bool fully_qualified_identifiers = true;
    bool qualify_indentifiers_with_database = true;
};

ASTPtr toAST(const ConvertToASTOptions & options) const;
String formatASTForErrorMessage() const;
```

- **`toAST()`**：将 Query Tree 节点转换回 AST 节点，支持配置
  - `add_cast_for_constants`：常量类型与列类型不匹配时添加 `_CAST`
  - `fully_qualified_identifiers`：使用 `database.table.column` 全限定名
  - `qualify_indentifiers_with_database`：是否包含 database 前缀
- **`formatASTForErrorMessage()`**：优先使用原始 AST 格式化错误信息，若无则使用转换后的 AST

### 4.7 树结构访问

```cpp
QueryTreeNodes & getChildren();
const QueryTreeNodes & getChildren() const;
String dumpTree() const;
void dumpTree(WriteBuffer & buffer) const;
virtual void dumpTreeImpl(WriteBuffer & buffer, FormatState & format_state, size_t indent) const = 0;
```

- `getChildren()` 返回子节点列表（强引用）
- `dumpTree()` 系列方法用于调试和日志输出

---

## 5. Template Method 模式分析

`IQueryTreeNode` 使用 **Template Method** 设计模式：

| 公有方法 | 调用的纯虚方法 | 子类职责 |
|---------|--------------|---------|
| `isEqual()` | `isEqualImpl()` | 比较子类自身的状态，不需要处理 children |
| `getTreeHash()` | `updateTreeHashImpl()` | 将子类自身状态更新到哈希中 |
| `clone()` → `cloneAndReplace()` | `cloneImpl()` | 克隆子类自身的状态 |
| `toAST()` | `toASTImpl()` | 将子类自身及 children 转换为 AST |
| `dumpTree()` | `dumpTreeImpl()` | 输出子类自身的调试信息及 children |

**关键约定**：
- 子类只处理**自身特有的状态**
- 公有方法自动处理 `children`、`weak_pointers`、`alias`、`original_ast` 的克隆/比较/哈希
- 子类通过构造函数传入 `children_size` 和 `weak_pointers_size` 来预留空间

---

## 6. 设计亮点

### 6.1 双指针体系（强引用 + 弱引用）

```
父节点 ──shared_ptr──▶ 子节点     (所有权)
列节点 ──weak_ptr────▶ 表节点     (引用，非所有权)
```

- **`children`**（`shared_ptr`）：表示树形结构的父子关系，父节点拥有子节点
- **`weak_pointers`**（`weak_ptr`）：表示节点间的交叉引用，如：
  - 列节点弱引用其来源表/子查询
  - 不增加引用计数，避免循环引用导致内存泄漏
  - 简化了查询计划阶段的依赖追踪

### 6.2 原始别名与动态别名分离

- `original_alias`：SQL 中定义的原始别名（不可变）
- `alias`：当前别名（可在优化 Pass 中修改）
- 用途：`additional_table_filters` 功能需要基于原始别名进行匹配

### 6.3 原始 AST 保留

- 保留 `original_ast` 用于错误信息格式化（提供更符合用户原始输入的 SQL 报错）
- 不参与树比较和哈希，减少比较开销

### 6.4 克隆时替换机制

- 支持在深拷贝过程中对特定节点进行替换
- 用于实现查询优化 Pass 中的子树替换，无需手动遍历和替换

---

## 7. 与其他组件的关系

```
SQL Parser ──▶ AST ──▶ Query Tree (IQueryTreeNode)
   ▲                      │
   │                      ▼
   └────── toAST() ──── Query Plan (Planning)
```

- **输入**：AST 经过语义分析（Analyzer）转换为 Query Tree
- **输出**：Query Tree 可以转换回 AST（无损），或作为 Query Plan 的输入
- **与 Analyzer 的关系**：IQueryTreeNode 是 Analyzer 模块的核心数据结构
- **与 Planning 的关系**：Query Tree 上的弱引用机制简化了计划阶段的依赖追踪

---

## 8. 总结

`IQueryTreeNode` 是 ClickHouse 查询优化与分析管线的**核心数据结构**，它：

1. **桥接 AST 与查询计划**：提供从语法到语义的中间表示
2. **支持高效遍历与变换**：通过 `cloneAndReplace` 实现"拷贝+替换"的不可变变换模式
3. **保留原始信息**：原始别名 + 原始 AST 的双重保留确保错误信息准确
4. **弱引用解耦**：通过 `weak_ptr` 实现节点间引用而不产生循环依赖
5. **Template Method 规范子类行为**：子类只需关注自身状态，公共行为由基类统一处理
