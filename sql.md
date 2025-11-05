好的！下面按你给的6条“验证思路”，分别给出**可直接在 Oracle 执行的 SQL**。我也尽量考虑到了你提到的“非主键、可能重复”的情况，所以在多数地方先做了去重/预聚合，既稳又快。

---

### 0) 小约定（便于读写和性能）

* 用一个规范化的交叉表 `XREF_N`：对 `transformation_model_xref` 先做去重。
* 用两个“状态视图”汇总：把 `transformation`、`data_model` 里同一 ID 的状态聚合成单值（这里用 `MAX(current_indicator)` 代表是否存在 active 版本）。

```sql
-- 规范化交叉表（去重）
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
),

-- 转换状态：每个 transformation_id 一行，1=有active版本，0=都inactive
TX AS (
  SELECT transformation_id, MAX(current_indicator) AS transformation_active
  FROM transformation
  GROUP BY transformation_id
),

-- 模型状态：每个 model_id 一行，1=有active版本，0=都inactive
DM AS (
  SELECT model_id, MAX(current_indicator) AS model_active
  FROM data_model
  GROUP BY model_id
)
SELECT 1 FROM dual;  -- 仅占位，便于你在下面各段直接复用这三个CTE的写法
```

> 下面各条验证里，如需复用上述 CTE，直接把对应的 `WITH ...` 放到每条查询最上面即可。

---

## 1) 「inactive 的 transformation 是否可能含有至少一个 active 的 data model？」

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
)
SELECT DISTINCT t.transformation_id
FROM transformation t
JOIN XREF_N x  ON x.transformation_id = t.transformation_id
JOIN data_model m ON m.model_id        = x.model_id
WHERE t.current_indicator = 0      -- transformation 是 inactive
  AND m.current_indicator = 1;     -- 至少一个关联的 model 是 active
```

> 有返回行即说明“**存在这种可能性**”。

---

## 2) 「若其中至少一个 data model 是 inactive，则 transformation 也应是 inactive」——找**违反**该规则的异常

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
)
SELECT DISTINCT t.transformation_id
FROM transformation t
JOIN XREF_N x  ON x.transformation_id = t.transformation_id
JOIN data_model m ON m.model_id        = x.model_id
WHERE t.current_indicator = 1      -- transformation 是 active
  AND m.current_indicator = 0;     -- 却含有 inactive 的 model（违规）
```

> **返回0行**则该规则在现有数据中成立；若有行，逐条即为**违规的 transformation**。

---

## 3) 「一个 SOURCE data model 是否能对应多个 TARGET data model？」

（给出**全局**和**按 transformation 维度**两种验证）

**全局（同一 SOURCE 跨多个 transformation 也计入）：**

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
)
SELECT
  xs.model_id AS source_model_id,
  COUNT(DISTINCT xt.model_id) AS target_cnt,
  LISTAGG(DISTINCT xt.model_id, ',') WITHIN GROUP (ORDER BY xt.model_id) AS target_models
FROM XREF_N xs
JOIN XREF_N xt
  ON xt.transformation_id = xs.transformation_id
 AND xt.field_type = 'TARGET'
WHERE xs.field_type = 'SOURCE'
GROUP BY xs.model_id
HAVING COUNT(DISTINCT xt.model_id) > 1;
```

**按单个 transformation 验证：**

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
)
SELECT
  xs.transformation_id,
  xs.model_id AS source_model_id,
  COUNT(DISTINCT xt.model_id) AS target_cnt,
  LISTAGG(DISTINCT xt.model_id, ',') WITHIN GROUP (ORDER BY xt.model_id) AS target_models
FROM XREF_N xs
JOIN XREF_N xt
  ON xt.transformation_id = xs.transformation_id
 AND xt.field_type = 'TARGET'
WHERE xs.field_type = 'SOURCE'
GROUP BY xs.transformation_id, xs.model_id
HAVING COUNT(DISTINCT xt.model_id) > 1;
```

> 任一查询有返回行，就证明“**存在一个 SOURCE 对应多个 TARGET**”。

---

## 4) 「是否存在一个 TARGET 对应多个 SOURCE？」

（同样给出全局与按 transformation 的两种）

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
)
SELECT
  xt.model_id AS target_model_id,
  COUNT(DISTINCT xs.model_id) AS source_cnt,
  LISTAGG(DISTINCT xs.model_id, ',') WITHIN GROUP (ORDER BY xs.model_id) AS source_models
FROM XREF_N xs
JOIN XREF_N xt
  ON xt.transformation_id = xs.transformation_id
 AND xt.field_type = 'TARGET'
WHERE xs.field_type = 'SOURCE'
GROUP BY xt.model_id
HAVING COUNT(DISTINCT xs.model_id) > 1;
```

**按 transformation：**

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
)
SELECT
  xt.transformation_id,
  xt.model_id AS target_model_id,
  COUNT(DISTINCT xs.model_id) AS source_cnt,
  LISTAGG(DISTINCT xs.model_id, ',') WITHIN GROUP (ORDER BY xs.model_id) AS source_models
FROM XREF_N xs
JOIN XREF_N xt
  ON xt.transformation_id = xs.transformation_id
 AND xt.field_type = 'TARGET'
WHERE xs.field_type = 'SOURCE'
GROUP BY xt.transformation_id, xt.model_id
HAVING COUNT(DISTINCT xs.model_id) > 1;
```

---

## 5) 「若 1 个 SOURCE 对应 2 个 TARGET，那在 `transformation` 里是否对应两个 `transformation_id`？」

> 我们给出一个**诊断表**：列出“恰好对应2个TARGET”的 SOURCE，并同时数它涉及的 `transformation_id` 个数，方便你判断到底是**一个** transformation 下挂两个 TARGET，还是**两个**不同的 transformation 各挂一个 TARGET。

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
),
PAIRS AS (
  SELECT xs.transformation_id,
         xs.model_id AS source_model_id,
         xt.model_id AS target_model_id
  FROM XREF_N xs
  JOIN XREF_N xt
    ON xt.transformation_id = xs.transformation_id
   AND xs.field_type = 'SOURCE'
   AND xt.field_type = 'TARGET'
),
AGG AS (
  SELECT
    source_model_id,
    COUNT(DISTINCT target_model_id)         AS target_cnt,
    COUNT(DISTINCT transformation_id)       AS transformation_cnt,
    LISTAGG(DISTINCT transformation_id, ',') WITHIN GROUP (ORDER BY transformation_id) AS transformation_ids
  FROM PAIRS
  GROUP BY source_model_id
)
SELECT *
FROM AGG
WHERE target_cnt = 2;
```

* 若 `transformation_cnt = 1`：说明**同一个 transformation_id** 下，这个 SOURCE 对应了两个 TARGET（不需要两个 transformation_id）。
* 若 `transformation_cnt = 2`：说明是**两个不同的 transformation_id** 对应两个 TARGET。
* 你也可以将 `WHERE target_cnt = 2` 改成 `>= 2` 看更一般的情况。

另外，直接找“同一 transformation 内，一个 source 对应多个 target”的明细：

```sql
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
)
SELECT
  xs.transformation_id,
  xs.model_id AS source_model_id,
  COUNT(DISTINCT xt.model_id) AS target_cnt,
  LISTAGG(DISTINCT xt.model_id, ',') WITHIN GROUP (ORDER BY xt.model_id) AS target_models
FROM XREF_N xs
JOIN XREF_N xt
  ON xt.transformation_id = xs.transformation_id
 AND xt.field_type = 'TARGET'
WHERE xs.field_type = 'SOURCE'
GROUP BY xs.transformation_id, xs.model_id
HAVING COUNT(DISTINCT xt.model_id) >= 2;
```

---

## 6) 「一条性能友好的联表，取出所有 transformation 及其 source/target 的模型信息」

给你两个版本：

### 6A) **成对明细版（推荐做分析）**

每一行是一个 `(transformation_id, source_model_id, target_model_id)` 对，同时带出三者的“是否active”状态；用预聚合避免重复，性能稳定。

```sql
WITH
XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
),
PAIRS AS (
  SELECT s.transformation_id,
         s.model_id AS source_model_id,
         t.model_id AS target_model_id
  FROM XREF_N s
  JOIN XREF_N t
    ON t.transformation_id = s.transformation_id
   AND t.field_type = 'TARGET'
  WHERE s.field_type = 'SOURCE'
),
TX AS (
  SELECT transformation_id, MAX(current_indicator) AS transformation_active
  FROM transformation
  GROUP BY transformation_id
),
DM AS (
  SELECT model_id, MAX(current_indicator) AS model_active
  FROM data_model
  GROUP BY model_id
)
SELECT
  p.transformation_id,
  tx.transformation_active,
  p.source_model_id,
  dms.model_active AS source_active,
  p.target_model_id,
  dmt.model_active AS target_active
FROM PAIRS p
LEFT JOIN TX tx   ON tx.transformation_id = p.transformation_id
LEFT JOIN DM dms  ON dms.model_id = p.source_model_id
LEFT JOIN DM dmt  ON dmt.model_id = p.target_model_id;
```

### 6B) **汇总版（每个 transformation 一行，列出源/目的列表）**

适合做总览或导出报表。

```sql
WITH
XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
),
TX AS (
  SELECT transformation_id, MAX(current_indicator) AS transformation_active
  FROM transformation
  GROUP BY transformation_id
),
SRC AS (
  SELECT transformation_id,
         LISTAGG(DISTINCT model_id, ',') WITHIN GROUP (ORDER BY model_id) AS source_models
  FROM XREF_N
  WHERE field_type = 'SOURCE'
  GROUP BY transformation_id
),
TGT AS (
  SELECT transformation_id,
         LISTAGG(DISTINCT model_id, ',') WITHIN GROUP (ORDER BY model_id) AS target_models
  FROM XREF_N
  WHERE field_type = 'TARGET'
  GROUP BY transformation_id
)
SELECT
  x.transformation_id,
  tx.transformation_active,
  src.source_models,
  tgt.target_models
FROM (SELECT DISTINCT transformation_id FROM XREF_N) x
LEFT JOIN TX  tx  ON tx.transformation_id  = x.transformation_id
LEFT JOIN SRC src ON src.transformation_id = x.transformation_id
LEFT JOIN TGT tgt ON tgt.transformation_id = x.transformation_id;
```

---

### 🔧 小小性能建议（可选）

1. 建索引（如果还没有）：

```sql
-- 交叉表最重要：按 (transformation_id, field_type, model_id)
CREATE INDEX IDX_XREF_TF_M ON transformation_model_xref (transformation_id, field_type, model_id);

-- 两侧维表：按 (id, current_indicator)
CREATE INDEX IDX_TX_ID_CI ON transformation (transformation_id, current_indicator);
CREATE INDEX IDX_DM_ID_CI ON data_model (model_id, current_indicator);
```

2. 定期收集统计信息（`DBMS_STATS.GATHER_TABLE_STATS`），让优化器选择 Hash Join 等更优计划。
3. 如果 Oracle 版本较老（不支持 `LISTAGG(DISTINCT ...)`），把 `LISTAGG(DISTINCT ...)` 换成**先去重再 LISTAGG** 的子查询即可。

---

需要我把这些查询封装成视图/物化视图，或者加上过滤条件（比如仅看 active/仅看某批 transformation）也可以直接帮你写。
