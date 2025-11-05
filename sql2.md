明白你新增的规则了：

* **同一个 `model_id` / `transformation_id` 在表里可能有多行**；
* 只要其中**任意一行** `current_indicator = 1`，这个 model/transform 就**视为 Active**；
* 只有当**同一个 ID 的所有行** `current_indicator = 0` 时，才**真的 Inactive**。

基于这个“**按 ID 聚合判断活性**”的语义，我把所有验证 SQL 都改为**先聚合出状态**再参与判断，避免行级误判。

---

# 统一前置（建议每条查询都带上）

```sql
-- 去重后的交叉表
WITH XREF_N AS (
  SELECT DISTINCT transformation_id, model_id, field_type
  FROM transformation_model_xref
),

-- 聚合后的 transformation 状态：1=该ID有任一active行；0=该ID所有行都inactive
TX AS (
  SELECT transformation_id,
         MAX(current_indicator) AS transformation_active
  FROM transformation
  GROUP BY transformation_id
),

-- 聚合后的 data_model 状态：1=该ID有任一active行；0=该ID所有行都inactive
DM AS (
  SELECT model_id,
         MAX(current_indicator) AS model_active
  FROM data_model
  GROUP BY model_id
)
```

> 下面每条 SQL 在最上面加上这个 `WITH ...` 即可直接跑。

---

## 1) Inactive 的 transformation 里是否可能包含至少一个 Active 的 data model？

```sql
WITH XREF_N AS (...), TX AS (...), DM AS (...)
SELECT DISTINCT tx.transformation_id
FROM TX tx
JOIN XREF_N x  ON x.transformation_id = tx.transformation_id
JOIN DM m      ON m.model_id          = x.model_id
WHERE tx.transformation_active = 0    -- 该 transform 的所有行都为0 => 真Inactive
  AND m.model_active = 1;             -- 至少关联到一个真Active的 model
```

> 有返回行就说明“**确实存在**这种情况”。

---

## 2) 若某 transformation 关联的 data model 里**至少有一个真 Inactive**，则该 transformation 也应当是 Inactive

— 查找**违反**该规则的 transformation

```sql
WITH XREF_N AS (...), TX AS (...), DM AS (...)
SELECT DISTINCT tx.transformation_id
FROM TX tx
JOIN XREF_N x  ON x.transformation_id = tx.transformation_id
JOIN DM m      ON m.model_id          = x.model_id
WHERE tx.transformation_active = 1    -- transform 被视为 Active（存在任一行=1）
  AND m.model_active = 0;             -- 却关联到至少一个真Inactive的 model（违规）
```

> **返回0行**则该规则在现有数据中成立；若有行，每条即为**违规**的 `transformation_id`。

---

## 3) 验证「一个 SOURCE data model 能对应多个 TARGET data model」

**按单个 transformation 粒度：**（更符合“在一个 transform 内，一个 source 连多个 target”的直觉）

```sql
WITH XREF_N AS (...)
SELECT
  s.transformation_id,
  s.model_id AS source_model_id,
  COUNT(DISTINCT t.model_id) AS target_cnt,
  LISTAGG(DISTINCT t.model_id, ',') WITHIN GROUP (ORDER BY t.model_id) AS target_models
FROM XREF_N s
JOIN XREF_N t
  ON t.transformation_id = s.transformation_id
 AND t.field_type = 'TARGET'
WHERE s.field_type = 'SOURCE'
GROUP BY s.transformation_id, s.model_id
HAVING COUNT(DISTINCT t.model_id) > 1;
```

> 有返回行即证明**存在**“一个 SOURCE 对应多个 TARGET”。

（如需**全局**不分 transformation 的统计，把 `GROUP BY` 改为仅 `s.model_id` 即可。）

---

## 4) 验证「一个 TARGET 是否会对应多个 SOURCE」

```sql
WITH XREF_N AS (...)
SELECT
  t.transformation_id,
  t.model_id AS target_model_id,
  COUNT(DISTINCT s.model_id) AS source_cnt,
  LISTAGG(DISTINCT s.model_id, ',') WITHIN GROUP (ORDER BY s.model_id) AS source_models
FROM XREF_N s
JOIN XREF_N t
  ON t.transformation_id = s.transformation_id
 AND t.field_type = 'TARGET'
WHERE s.field_type = 'SOURCE'
GROUP BY t.transformation_id, t.model_id
HAVING COUNT(DISTINCT s.model_id) > 1;
```

---

## 5) 如果 1 个 SOURCE 对应 2 个 TARGET，那是否会对应两个 `transformation_id`？

> 输出“恰好2个 TARGET”的 SOURCE，并统计它涉及的 `transformation_id` 个数，判断是**一个 transform 内**的多 TARGET，还是**多个 transform 各连1个 TARGET**。

```sql
WITH XREF_N AS (...),
PAIRS AS (
  SELECT s.transformation_id,
         s.model_id AS source_model_id,
         t.model_id AS target_model_id
  FROM XREF_N s
  JOIN XREF_N t
    ON t.transformation_id = s.transformation_id
   AND s.field_type = 'SOURCE'
   AND t.field_type = 'TARGET'
),
AGG AS (
  SELECT
    source_model_id,
    COUNT(DISTINCT target_model_id) AS target_cnt,
    COUNT(DISTINCT transformation_id) AS transformation_cnt,
    LISTAGG(DISTINCT transformation_id, ',') WITHIN GROUP (ORDER BY transformation_id) AS transformation_ids
  FROM PAIRS
  GROUP BY source_model_id
)
SELECT *
FROM AGG
WHERE target_cnt = 2;
```

* `transformation_cnt = 1` → 同一个 transformation 下的 2 个 TARGET（**不是**两个 transformation）。
* `transformation_cnt = 2` → 分属两个不同的 transformation。

> 若想看“≥2 个 TARGET”的一般情形，把 `WHERE target_cnt = 2` 改为 `>= 2`。

---

## 6) 一次性优雅高效地取出：每个 transformation 及其 Source/Target 与各自活性

**成对明细版（推荐分析/核对）：**

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
LEFT JOIN TX  tx  ON tx.transformation_id  = p.transformation_id
LEFT JOIN DM  dms ON dms.model_id         = p.source_model_id
LEFT JOIN DM  dmt ON dmt.model_id         = p.target_model_id;
```

**汇总版（每个 transformation 一行，列出源/目的列表 + 活性）：**

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
         LISTAGG(model_id, ',') WITHIN GROUP (ORDER BY model_id) AS source_models
  FROM (SELECT DISTINCT transformation_id, model_id FROM XREF_N WHERE field_type = 'SOURCE')
  GROUP BY transformation_id
),
TGT AS (
  SELECT transformation_id,
         LISTAGG(model_id, ',') WITHIN GROUP (ORDER BY model_id) AS target_models
  FROM (SELECT DISTINCT transformation_id, model_id FROM XREF_N WHERE field_type = 'TARGET')
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

### 补充的小建议

* 你可以把 `TX` / `DM` 两个聚合状态做成**视图**（或物化视图）方便复用：

  ```sql
  CREATE OR REPLACE VIEW v_transformation_status AS
  SELECT transformation_id, MAX(current_indicator) AS transformation_active
  FROM transformation GROUP BY transformation_id;

  CREATE OR REPLACE VIEW v_data_model_status AS
  SELECT model_id, MAX(current_indicator) AS model_active
  FROM data_model GROUP BY model_id;
  ```

  之后把上述 SQL 里的 `TX` / `DM` 分别替换为视图即可，语义与性能都更稳定。
* 若数据库版本不支持 `LISTAGG(DISTINCT ...)`，我已在汇总版里演示了“先 `SELECT DISTINCT` 再 `LISTAGG`”的写法。
* 给 `transformation_model_xref (transformation_id, field_type, model_id)`、
  `transformation (transformation_id, current_indicator)`、
  `data_model (model_id, current_indicator)`
  建索引，并保持统计信息新鲜，可让优化器倾向 Hash Join / 半连接等更优执行计划。

需要我按你的库名/Schema 前缀、或者只筛 Active/Inactive 的不同视图再细化一版，也可以直接改好给你。
