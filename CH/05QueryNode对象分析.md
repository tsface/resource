# QueryNode 对象分析

## 概述

`QueryNode` 是 `IQueryTreeNode` 的一个具体子类，对应枚举类型 `QueryTreeNodeType::QUERY`。它表示查询树中的一个**查询节点**，对应一个完整的 `SELECT` 查询（包括子查询、CTE 等）。

它是整个查询树中最核心、最复杂的节点类型，承载了一个 SQL 查询的全部语义信息。

---

## 1. 类继承关系

```
TypePromotion<IQueryTreeNode>
        │
  IQueryTreeNode              ← 抽象基类（模板方法模式）
        │
   QueryNode                 ← final 具体子类
```

- 标记为 `final`，不允许进一步继承
- 继承 `IQueryTreeNode` 的所有能力：克隆、比较、哈希、AST 转换等

---

## 2. 核心设计：children 索引布局

`QueryNode` 的一个独特设计是将 SQL 查询的所有子句（clause）映射到 `children` 数组的固定索引位置上：

```cpp
static constexpr size_t children_size = 16;  // 0 ~ 15

索引  │ 名称                  │ 类型         │ 说明
──────┼───────────────────────┼──────────────┼────────────────────────
  0   │ with_child_index      │ ListNode     │ WITH 子句（CTE 列表）
  1   │ projection_child_index│ ListNode     │ SELECT 投影列
  2   │ join_tree_child_index │ QueryTreeNode│ FROM/JOIN 表树
  3   │ prewhere_child_index  │ QueryTreeNode│ PREWHERE 条件
  4   │ where_child_index     │ QueryTreeNode│ WHERE 条件
  5   │ group_by_child_index  │ ListNode     │ GROUP BY 表达式列表
  6   │ having_child_index    │ QueryTreeNode│ HAVING 条件
  7   │ window_child_index    │ ListNode     │ WINDOW 窗口定义列表
  8   │ qualify_child_index   │ QueryTreeNode│ QUALIFY 条件
  9   │ order_by_child_index  │ ListNode     │ ORDER BY 排序列表
  10  │ interpolate_child_index│ QueryTreeNode│ INTERPOLATE 插值
  11  │ limit_by_limit_child  │ QueryTreeNode│ LIMIT BY 的 LIMIT
  12  │ limit_by_offset_child │ QueryTreeNode│ LIMIT BY 的 OFFSET
  13  │ limit_by_child_index  │ ListNode     │ LIMIT BY 表达式列表
  14  │ limit_child_index     │ QueryTreeNode│ LIMIT 值
  15  │ offset_child_index    │ QueryTreeNode│ OFFSET 值
```

**设计特性**：
- 通过 `constexpr` 编译时常量定义索引，访问时用 `children[idx]` 直接下标访问，**没有运行时查找开销**
- 有些位置用 `nullptr` 表示"该子句不存在"（例如 `WHERE` 为空），通过 `hasXxx()` 方法判断
- `ListNode` 类型的位置始终是有效的 `ListNode` 对象（即使为空列表），其余位置可能为 `nullptr`

---

## 3. 布尔标志位

QueryNode 使用 11 个布尔标志位记录查询的修饰属性：

| 标志位 | 对应 SQL 语法 | 说明 |
|--------|--------------|------|
| `is_subquery` | `SELECT ... FROM (SELECT ...)` | 是否为子查询 |
| `is_cte` | `WITH cte AS (SELECT ...)` | 是否为 CTE |
| `is_recursive_with` | `WITH RECURSIVE ...` | 是否为递归 CTE |
| `is_distinct` | `SELECT DISTINCT ...` | 是否去重 |
| `is_limit_with_ties` | `LIMIT n WITH TIES` | LIMIT 是否带 TIES |
| `is_group_by_with_totals` | `GROUP BY ... WITH TOTALS` | 是否含合计行 |
| `is_group_by_with_rollup` | `GROUP BY ... WITH ROLLUP` | ROLLUP 修饰 |
| `is_group_by_with_cube` | `GROUP BY ... WITH CUBE` | CUBE 修饰 |
| `is_group_by_with_grouping_sets` | `GROUP BY GROUPING SETS(...)` | GROUPING SETS 修饰 |
| `is_group_by_all` | `GROUP BY ALL` | GROUP BY ALL 修饰 |
| `is_order_by_all` | `ORDER BY ALL` | ORDER BY ALL 修饰 |

---

## 4. 其他成员变量

| 成员 | 类型 | 说明 |
|------|------|------|
| `cte_name` | `std::string` | CTE 的名称（仅当 `is_cte == true` 时有意义） |
| `projection_columns` | `NamesAndTypes` | 投影列的「列名 + 数据类型」解析结果，在语义分析阶段填充 |
| `context` | `ContextMutablePtr` | 查询上下文（设置、权限等） |
| `settings_changes` | `SettingsChanges` | 查询级别的 SETTINGS 修改 |

---

## 5. 核心接口分析

### 5.1 子句访问器

对于每个子句，QueryNode 提供三组访问模式：

```cpp
// 模式1：是否存在
bool hasWhere() const;

// 模式2：返回 ListNode 引用（适合 WITH、PROJECTION、GROUP BY 等列表类型）
const ListNode & getWith() const;
ListNode & getWith();

// 模式3：返回 QueryTreeNodePtr 引用（适合单个表达式或 nullptr 类型）
const QueryTreeNodePtr & getWhere() const;
QueryTreeNodePtr & getWhere();
```

- **列表类型子句**（WITH、PROJECTION、GROUP BY、WINDOW、ORDER BY、LIMIT BY）：返回 `ListNode &`，因为即使为空也有一个空的 `ListNode`
- **可选表达式子句**（PREWHERE、WHERE、HAVING、QUALIFY、INTERPOLATE、LIMIT、OFFSET 等）：返回 `QueryTreeNodePtr &`，可能为 `nullptr`

### 5.2 投影列解析

```cpp
const NamesAndTypes & getProjectionColumns() const;
void resolveProjectionColumns(NamesAndTypes projection_columns_value);
void removeUnusedProjectionColumns(const std::unordered_set<std::string> & used_projection_columns);
void removeUnusedProjectionColumns(const std::unordered_set<size_t> & used_projection_columns_indexes);
```

- **`getProjectionColumns()`**：获取解析后的投影列描述（名称 + 类型）
- **`resolveProjectionColumns()`**：在语义分析阶段填充投影列信息
- **`removeUnusedProjectionColumns()`**：优化阶段移除未被后续操作使用的投影列（列裁剪优化）

### 5.3 上下文与设置

```cpp
ContextPtr getContext() const;
const ContextMutablePtr & getMutableContext() const;
ContextMutablePtr & getMutableContext();
bool hasSettingsChanges() const;
const SettingsChanges & getSettingsChanges() const;
void clearSettingsChanges();
```

- 每个 QueryNode 持有自己的 `Context`（查询级别上下文）
- `settings_changes` 记录查询中的 `SETTINGS` 子句修改，例如：
  ```sql
  SELECT * FROM test_table SETTINGS prefer_column_name_to_alias = 1, join_use_nulls = 1;
  ```

### 5.4 IQueryTreeNode 接口实现

```cpp
QueryTreeNodeType getNodeType() const override { return QueryTreeNodeType::QUERY; }
```

子类实现的虚方法：

| 方法 | 职责 |
|------|------|
| `isEqualImpl()` | 比较所有布尔标志位、`cte_name`、`projection_columns`、`context`、`settings_changes` |
| `updateTreeHashImpl()` | 将上述状态更新到哈希中 |
| `cloneImpl()` | 深拷贝所有成员，包括 `context`（浅拷贝指针，CTX 本身是 `shared_ptr` 管理的） |
| `toASTImpl()` | 将 QueryNode 还原为 AST 的 `SELECT` 节点 |
| `dumpTreeImpl()` | 输出调试信息 |

---

## 6. 一个 SELECT 查询在 QueryNode 中的表示示例

以这条 SQL 为例：

```sql
WITH cte AS (
    SELECT 1 AS id
)
SELECT DISTINCT t1.id, t1.value
FROM test_table_1 AS t1
INNER JOIN test_table_2 AS t2 ON t1.id = t2.id
WHERE t1.value > 10
GROUP BY t1.id
HAVING COUNT(*) > 1
ORDER BY t1.id DESC
LIMIT 10
OFFSET 5
SETTINGS prefer_column_name_to_alias = 1;
```

QueryNode 的结构：

```
QueryNode
├── is_subquery = false
├── is_cte = false
├── is_distinct = true
├── is_limit_with_ties = false
├── children[0]  WITH:   ListNode [CTE QueryNode("cte")]
├── children[1]  PROJECTION: ListNode [column("t1.id"), column("t1.value")]
├── children[2]  JOIN TREE: JoinNode(INNER) ─┬─ TableNode("test_table_1" AS t1)
│                                              └─ TableNode("test_table_2" AS t2)
├── children[3]  PREWHERE:  nullptr
├── children[4]  WHERE:     FunctionNode(greater, column("t1.value"), constant(10))
├── children[5]  GROUP BY:  ListNode [column("t1.id")]
├── children[6]  HAVING:    FunctionNode(greater, FunctionNode(count), constant(1))
├── children[7]  WINDOW:    ListNode []
├── children[8]  QUALIFY:   nullptr
├── children[9]  ORDER BY:  ListNode [SortNode(column("t1.id"), DESC)]
├── children[10] INTERPOLATE: nullptr
├── children[11] LIMIT BY LIMIT:   nullptr
├── children[12] LIMIT BY OFFSET:  nullptr
├── children[13] LIMIT BY:  ListNode []
├── children[14] LIMIT:     constant(10)
├── children[15] OFFSET:    constant(5)
├── projection_columns: [("t1.id", UInt32), ("t1.value", String)]
├── settings_changes: [prefer_column_name_to_alias = 1]
└── context: <QueryContext>
```

---

## 7. 设计亮点

### 7.1 固定索引布局 vs 运行时映射

使用 `constexpr` 固定索引而不是运行时 `enum/map` 查找：
- 编译期确定访问路径
- 没有哈希查找或分支预测开销
- 代码可读性好，索引名称自文档化

### 7.2 可选子句的 nullptr 语义

对于 WHERE、HAVING 等可选子句，用 `nullptr` 表示"不存在"，而不是用空节点占位：
- 减少不必要的对象创建
- `hasXxx()` 方法只需检查 `nullptr`，非常高效

### 7.3 投影列的双阶段处理

1. 构造阶段：只保存投影表达式（`ListNode`）
2. 语义分析阶段：`resolveProjectionColumns()` 填入解析后的 `NamesAndTypes`
3. 优化阶段：`removeUnusedProjectionColumns()` 裁剪未使用的列

这种分离使得查询优化可以在语义信息就绪后进行。

### 7.4 Context 与 Settings 的分离

- `context`：全局查询上下文（共享）
- `settings_changes`：仅记录 `SETTINGS` 子句中的覆盖项
- 设计上保持了"全局设置 + 局部覆盖"的清晰模式

---

## 8. 与其他节点的关系

```
QueryNode
  ├── children[WITH] ──────────────────▶ ListNode ──▶ [QueryNode(CTE1), QueryNode(CTE2), ...]
  ├── children[PROJECTION] ────────────▶ ListNode ──▶ [ColumnNode, FunctionNode, ConstantNode, ...]
  ├── children[JOIN TREE] ─────────────▶ JoinNode / TableNode / TableFunctionNode / QueryNode(子查询)
  ├── children[GROUP BY] ──────────────▶ ListNode ──▶ [ColumnNode, ...]
  ├── children[WINDOW] ────────────────▶ ListNode ──▶ [WindowNode, ...]
  ├── children[ORDER BY] ──────────────▶ ListNode ──▶ [SortNode, ...]
  ├── children[LIMIT BY] ──────────────▶ ListNode ──▶ [ColumnNode, ...]
  ├── children[PREWHERE/WHERE/HAVING] ▶ [FunctionNode, ConstantNode, ...]
  └── children[LIMIT/OFFSET] ─────────▶ ConstantNode
```

---

## 9. 总结

`QueryNode` 是 ClickHouse 查询树的**核心节点类型**，完整地表示一个 `SELECT` 语句的所有组成部分：

| 维度 | 设计 |
|------|------|
| **子句表示** | 固定索引 `children[]` 数组，`constexpr` 编译时常量 |
| **列表子句** | 使用 `ListNode` 包装，永远有效 |
| **可选子句** | 使用 `QueryTreeNodePtr`，`nullptr` 表示不存在 |
| **修饰属性** | 11 个布尔标志位 |
| **投影信息** | 先存表达式 → 后解析类型 → 最后裁剪未用列 |
| **上下文** | 每个查询节点持有独立 Context 和局部 Settings |
| **CTE** | 支持命名 CTE、递归 CTE、子查询标识 |


