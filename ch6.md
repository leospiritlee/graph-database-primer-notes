# 第6章 Cypher 入门：把图模式写出来

**一句话**：`Cypher` 用 `()` 表示点、`-[]->` 表示边，把 `图模式` 画成 `ASCII 艺术`；`MATCH`、`WHERE`、`RETURN` 负责读，`CREATE`、`MERGE`、`SET`、`DELETE` 负责写，其中 `MERGE` 是导入时保证 `幂等` 的关键。

**为什么需要它**：`Cypher` 属于 `声明式语言`（Declarative query language），用户只需要告诉系统「要什么」，而不管「怎么做」；与之相对 `Gremlin` 是 `命令式语言`（Imperative query language），用户会显式地指定系统的操作。资料把 `Cypher` 定位为入门首选：语法就是「画图」，节点用圆括号、关系用方括号、箭头表方向，和脑海里的图结构一一对应。2015 年 `Neo4j` 将 `Cypher` 开源为 `openCypher`，并成立 `openCypher Implementors Group`（`oCIG`）以开源方式治理语言演化；`Cypher` 至今仍是最受欢迎的图查询语言。第 7 章再谈别的语言，这一章只把最小可用链写熟。

**核心结构**：按学习顺序，`CREATE` 与 `MERGE` 负责写入，`MATCH` 与 `WHERE` 负责 `模式匹配` 与过滤，`RETURN`、`ORDER BY`、`LIMIT`、`SKIP` 负责投影与分页，`SET`、`REMOVE`、`DELETE`、`DETACH DELETE` 负责修改与删除。进阶部分包括 `count`、`collect`、`DISTINCT` 等聚合，`WITH` 做流程控制，`可变长路径` 写成 `*1..3`，`shortestPath` 求最短路径，以及 `子查询`、`UNION`、`IN`。资料给的示例都是从真实查询形态出发的：

```text
MATCH (zz:Person {name:'张三'})-[:FRIEND]->(f:Person)
MATCH (f)-[:RECOMMENDED]->(b:Book)
RETURN b.title
```

```text
MERGE (p:Person {name: 'Alice'})
ON CREATE SET p.age = 25
ON MATCH SET p.lastLogin = timestamp()
RETURN p
```

**关键设计**：**`MERGE` 是 `幂等` 的，导入必用；用 `CREATE` 而非 `MERGE` 导入会产生 `重复节点`**。这是资料反复点名的高频坑，也是最容易被新手忽略的一条：`CREATE` 每次执行都新增，重跑一次导入就把图翻倍。`MERGE` 配合 `唯一性约束` 才能既保证不重复又保证性能——第 10 章会把 `LOAD CSV`、`apoc` 与 `唯一性约束` 串成完整导入流程。另一条设计线索是历史：`Cypher` 第一个版本只实现对图的读取，需要用户说明从哪些节点开始查询；`Neo4j 2.0` 引入 `标签` 后，查询引擎可以利用 `索引` 选择 `模式匹配` 命中的节点，用户不必再指定起点。今天的「写起来像自然语言」，是 `标签` 与 `索引` 打底之后的结果。

**拆解**：

### 从图模式到结果：一条语句的四个动作

`MATCH` 描述要找的 `图模式`，`WHERE` 追加过滤，`RETURN` 投影，`ORDER BY` 与 `LIMIT` 收尾。执行上对应第 1 章的流水线：解析成 `AST`、生成 `逻辑计划`、优化器把标签过滤下推到 `Expand` 并用 `属性` 索引选入口、再生成 `物理计划`。`NodeIndexSeek` 出现说明 `锚点` 命中索引；`Expand` 之后是 `Filter`、`Project`、`Sort`。理解这条链，第 7 章的 `EXPLAIN` 与 `PROFILE` 就不是死记命令，而是「看流水线在哪一步爆了」。

### 写操作的边界

`SET` 改 `属性`，`REMOVE` 删 `属性` 或 `标签`，`DELETE` 删 `节点` / `关系`，`DETACH DELETE` 先删关系再删节点。资料把 `MERGE` 的两种分支写法也给了：`ON CREATE SET` 与 `ON MATCH SET`，正好对应「不存在则建、存在则更新」的 `幂等` 语义。第 2 章说过 `关系` 只能有一个 `标签`、`节点` 可以有零个或者多个标签，这些约束在写操作里会直接体现。

### 为什么 ASCII 艺术重要

`Cypher` 的语法基础是用 `ASCII 艺术`（`ASCII art`）描述 `图模式`，最初来源于 `Neo4j` 工程师团队在源代码中评注如何描述图模式：`Cypher` 文本用 `()` 表示点，`-[]->` 表示边。资料里的经典例子是 `(query)-[modeled as]->(drawing)` 表示起点 query、终点 drawing、边 modeled as。这不只是一个漂亮的写法，它让第 8 章的「按查询建模型」变得可操作：把业务问题写成一句自然语言，看它能否直接翻译成一条 `图模式`；翻译不了，通常说明模型不对。

### 与其他语言的分工

`Gremlin` 基于 `Apache TinkerPop`，风格接近一连串的函数（过程）调用，`图灵完备`，灵活但难优化；它的原型借鉴 `XPath` 作为图查询，用 `Groovy` 提供循环结构、分支和计算，后来改为基于 `Groovy` 的一种 `领域特定语言`（`DSL`）。`SPARQL` 面向 `RDF`。厂商语言还有 `nGQL`（`NebulaGraph` Query Language，2019 年以 `openCypher` 为基础发布）、`GSQL`（`TigerGraph`）、`AQL`（`ArangoDB`）、`PGQL`（Oracle PGX）、`G-CORE`（`LDBC` 发布）。资料也说 `Cypher` 启发了一系列后续的图查询语言。第 7 章会把 `GQL` 标准单独拆开。

**今天怎么做**：按资料给的顺序，每个子句手写至少三遍：`CREATE` / `MERGE`，`MATCH` / `WHERE`，`RETURN` / `ORDER BY` / `LIMIT`，`SET` / `REMOVE` / `DELETE`；再进阶聚合、`WITH`、`可变长路径`、`shortestPath`。**延伸，非语料原文**：练习时固定用一份小而能看懂结果的数据集，每写一条就先口述它应该返回哪几行，比追求语法覆盖面更能建立手感。

<div class="chapend" markdown="1">

**容易误解的地方**：把 `CREATE` 当 `MERGE` 用——重跑导入就产生 `重复节点`；以为 `Cypher` 是 `过程式` 语言要自己指定遍历顺序——它是 `声明式语言`；把 `-[]->` 里的方向当成可选装饰——第 2 章说过边一定有方向；还有把「`Cypher` 是最受欢迎的图查询语言」误解为「所有图库都用它」，`JanusGraph` 用 `Gremlin`、`RDF` 系用 `SPARQL`。

**记住这一句**：`MATCH` 画 `图模式`、`WHERE` 过滤、`RETURN` 投影，`MERGE` 加 `唯一性约束` 保证 `幂等`；`Cypher` 的好写来自 `标签` 与 `索引` 打底，而不是语法本身有什么魔法。

</div>
