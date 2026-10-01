# Cube → Omni field mapping

Parameter-by-parameter translation from a Cube data model to Omni model YAML.
Legend: ✅ direct · ⚠️ lossy or requires judgment · ❌ no mapping (report it).

Structural losses and the reasoning behind the ⚠️/❌ rows are in
[LIMITATIONS.md](../../cube-omni-pipeline/references/LIMITATIONS.md). Omni
parameter definitions are in
[`omni-model-builder` → modelParameters](../../../../omni-model-builder/references/modelParameters.md).

---

## Object-level mapping

| Cube | Omni | | Notes |
|---|---|---|---|
| `cubes[]` entry with `sql_table` | a `.view` file | ✅ | The physical layer on both sides. One cube per file, one view per file. |
| `cubes[]` entry with `sql:` | a `.view` file with `sql:` (query view) | ✅ | Cube's derived cube ≈ Omni's [query view](https://docs.omni.co/modeling/query-views.md). |
| `views[]` entry | a `.topic` file | ✅ | **The key mapping.** A Cube view is the curated layer users query; so is an Omni topic. |
| `joins[]` on a cube | an entry in the model's `relationships` file, or topic-level `relationships` | ⚠️ | Cube declares joins on the left cube; Omni relationships are global or topic-scoped. See [Joins](#joins) below. |
| `segments[]` | a boolean dimension (define once in a [`template: true`](https://docs.omni.co/modeling/views/parameters/template.md) view + `extends` when shared), or topic `always_where_filters` / `default_filters` | ✅ | No object *named* segment, but the properties map: named, reusable, one-click filterable in the UI. ⚠️ A template view is reachable **only via `extends`** — never join it or name it in `always_where`; reference the *inherited* dimension. |
| `hierarchies[]` | `drill_fields` | ⚠️ | Loses nesting and the hierarchy's own name. |
| `pre_aggregations[]` | `materialized_query` view | ❌ | **Do not auto-translate.** Report and let the user decide. |
| `access_policy[]` | topic `access_filters` + `required_access_grants` / `hidden_unless_access_grants` | ⚠️ | **Depends on [user attributes](https://docs.omni.co/administration/users/attributes.md) the sync cannot create.** A missing or unassigned attribute **does not fail validation** — the model validates, the query runs, and the filter silently does not constrain. Annotate the attribute names and verify per role with [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md). |
| A `.js` / Jinja model file | — | ❌ | Translate the compiled output, not the generator. |

> **Where Cube puts a measure matters.** Cube convention is that business
> metrics live on a **view** (or on a cube and are then exposed via a view). Omni
> requires every dimension and measure to live in a **view file**; a topic only
> curates and joins them. So a measure defined directly on a Cube *view* must be
> written into the Omni *base view file*, then exposed through the topic's
> `fields:`. Getting this backwards produces a topic that references fields that
> do not exist.

---

## Cube (physical layer) → Omni view

| Cube parameter | Omni | | Notes |
|---|---|---|---|
| `name` | view file name | ✅ | `snake_case` on both sides. |
| `sql_table: schema.table` | `schema:` + `table_name:` | ✅ | Split the qualified name. |
| `sql: SELECT …` | `sql:` on the view | ✅ | Becomes a query view. |
| `title` | `label` | ✅ | |
| `description` | `description` | ✅ | Consumed by the Omni Agent too. |
| `public: false` | `hidden: true` | ⚠️ | Omni `hidden` keeps the field queryable in SQL; `ignored: true` is the harder form. Cube's `public: false` also removes it from `/v1/meta`. |
| `extends` | view `extends` | ⚠️ | Both support inheritance; resolution/override order differs. Verify the *compiled* result on both sides. |
| `data_source` | the Omni connection | ⚠️ | Model-level in Omni, not per-view. Multiple Cube data sources → multiple Omni models. |
| `sql_alias` | — | ❌ | Omni scopes view names itself (`alwaysScopeViewNames`). |
| `refresh_key` | topic `cache_policy` | ⚠️ | Different mechanism. Not faithful. |
| `meta.ai_context` | `ai_context` | ✅ | **The AI round-trip carrier.** Note Cube ignores cube-level `ai_context` — only view- and member-level reach its agent. |
| `meta.*` (other keys) | `tags`, or fold into `ai_context` | ⚠️ | Free-form on both sides; no schema to preserve. |
| `calendar: true` | model-level [`custom_calendars`](https://docs.omni.co/modeling/models/custom-calendars.md), or `fiscal_month_offset` + `fiscal_*` timeframes | ✅ | Retail 4-5-4 is a documented Omni example; `mappings` points each period at a column in your calendar table and Omni joins it on the date without fan-out. ⚠️ A custom calendar **replaces** the fiscal calendar on the dimensions that use it — `fiscal_month_offset` and every `fiscal_*` timeframe become unavailable **on those dimensions**. Both can coexist in one model on *different* dimensions. |

---

## Dimensions

| Cube | Omni | | Notes |
|---|---|---|---|
| `name` | dimension key | ✅ | ⚠️ Rename if it collides with a reserved granularity (`day`, `week`, `month`, `quarter`, `year`, `hour`, `minute`, `second`). |
| `sql: column` | *omit `sql:` entirely* | ✅ | **Do not write `sql: column` in Omni.** A plain column dimension auto-maps by name. Only add `sql:` for derived expressions. |
| `sql: "{CUBE}.a || {CUBE}.b"` | `sql: ${a} \|\| ${b}` | ⚠️ | Rewrite references: Cube `{field}`/`{CUBE}.col` → Omni `${field}`/`${view.field}`. **There is no `${TABLE}` in Omni.** |
| `type: string` | (inferred) | ✅ | Omni infers type from the column. |
| `type: number` | (inferred) | ✅ | |
| `type: boolean` | (inferred) | ✅ | |
| `type: time` | dimension + `timeframes:` | ✅ | See [Time and granularity](#time-and-granularity). |
| `type: geo` (`latitude`/`longitude`) | two dimensions | ⚠️ | No single geo type; map viz is chart-side. |
| `type: switch` (`values`) | a [filter-only field](https://docs.omni.co/modeling/templated-filters/index.md) in the view's `filters:` block with `suggestion_list` + `filter_single_select_only`; or `groups` when the values bucket a real column | ✅ | The enum **is** enforced: `suggestion_list` fixes the allowed values, `filter_single_select_only` forces a single choice. ⚠️ `display_order` is *not* a substitute — it orders the field in the picker and constrains nothing. Tesseract-only on the Cube side. |
| `primary_key: true` | `primary_key: true` | ✅ | **Required.** Omni needs it for correct aggregation and joins; a missing primary key is the most common cause of wrong numbers after a sync. |
| `title` | `label` | ✅ | |
| `description` | `description` | ✅ | |
| `public: false` | `hidden: true` | ⚠️ | |
| `format: currency` + `currency: USD` | `format: currency_2` | ⚠️ | Omni formats are a named set — see *Format Values* in modelParameters. |
| `format: percent` | `format: percent_2` | ⚠️ | Pick the precision variant deliberately. |
| `format: imageUrl` / `link` / `id` | `format` / `links` | ⚠️ | Partial; `links` carries templated URLs. |
| `order` | `display_order`, or `order_by_field` | ⚠️ | Two different Omni parameters depending on intent. |
| `granularities` (custom) | a derived dimension + `group_label` | ✅ | Omni's `timeframes` list is closed, but a custom grain is just a dimension. See [Time and granularity](#time-and-granularity). |
| `sub_query: true` | `level_of_detail: fixed`, or a [query view](https://docs.omni.co/modeling/views/parameters/query.md) | ✅ | Cube's `sub_query` puts a measure inside a dimension; `level_of_detail: fixed` is the direct analog. |
| `propagate_filters_to_sub_query` | query view [`bind_all_filters`](https://docs.omni.co/modeling/query-views/parameters/index.md) (or `bind` for named fields, `bind_outer_context` for context operators) | ⚠️ | *"Passes all chosen filters from the outer query into the query view."* With `level_of_detail`, outer filters already apply unless excluded. ⚠️ If Cube is pushing predicates into a **pre-aggregation**, that is caching behavior Omni has no analog for — Omni pushes into the subquery at query time. Validate numerically. |
| `mask` (default `NULL`) | [`mask_unless_access_grants`](https://docs.omni.co/modeling/dimensions/parameters/mask-unless-access-grants.md) | ⚠️ | Masks the **value**, not the field, so aggregation and grouping still work — Cube's actual use case. Omni's built-in mask is an **MD5 hash**, not a value you choose. |
| `mask: -1` / `mask: {sql: …}` (a chosen replacement) | one companion dimension per masking variant, each gated by complementary [`access_grants`](https://docs.omni.co/modeling/models/access-grants.md) | ✅ | Arbitrary mask shapes, with more tiers than Cube allows: `code_full`, `code_partial` (`sql: CONCAT('***', RIGHT(${code}, 3))`), each behind a grant keyed to a different `allowed_values` of one user attribute — two grants on one attribute give the inverse condition. ⚠️ One field per variant; grants must be mutually exclusive **and** exhaustive; **grants do not propagate through a measure** referencing the masked dimension. Annotate the attributes, re-test per role — [NOTATION.md](../../cube-omni-pipeline/references/NOTATION.md). |
| `links` | `links` | ⚠️ | Different templating contract; verify each. |
| `time_shift` | — | ❌ | Omni does period-over-period at query time. |
| `meta.ai_context` | `ai_context` | ✅ | |
| `case` (on a dimension) | `groups:` + `else:` | ✅ | Good match — Omni `groups` is CASE-like bucketing with an `else` fallback. String fields only. |
| `synthetic` | — | ❌ | |

### Time and granularity

A Cube `type: time` dimension becomes an Omni dimension with `timeframes:`.
Omni's `timeframes` is a **closed list**, and Cube's `granularities` is open:

| Cube granularity | Omni timeframe | |
|---|---|---|
| `second`, `minute`, `hour`, `day`, `week`, `month`, `quarter`, `year` | `second`, `minute`, `hour`, `date`, `week`, `month`, `quarter`, `year` | ✅ (`day` → `date`) |
| a custom `granularities:` entry (e.g. `interval: 2 weeks`) | a derived dimension holding the bucketing SQL, given the base timestamp's `group_label` so it sits **beside** the default timeframes in the picker | ✅ A custom grain is just a dimension. ⚠️ [`duration`](https://docs.omni.co/modeling/dimensions/parameters/duration.md) is *not* this — it measures elapsed time between `sql_start` and `sql_end`, not buckets of one timestamp. |
| a fiscal calendar via a `calendar` cube | `fiscal_quarter` / `fiscal_year` + `fiscal_month_offset`, **or** `custom_calendars` for a real calendar table | ✅ See the `calendar: true` row above for the replacement caveat. |

Omni additionally offers date-*part* timeframes with no Cube equivalent
(`day_of_week_name`, `month_name`, `quarter_of_year`, `day_of_year`, …). These
are a **free gain** on the way into Omni — add the defaults
(`raw, date, week, month, quarter, year`) plus any part the source model
clearly needed, and note that they will not survive a return sync.

---

## Measures

### Aggregate type

| Cube `type` | Omni `aggregate_type` | | Notes |
|---|---|---|---|
| `count` | `count` | ✅ | |
| `count_distinct` | `count_distinct` | ✅ | |
| `count_distinct_approx` | `count_distinct` | ⚠️ | **Changes the numbers.** HLL is approximate and additive; exact distinct is neither. Never translate silently. |
| `sum` | `sum` | ✅ | |
| `avg` | `average` | ✅ | Note the spelling difference. |
| `min` | `min` | ✅ | |
| `max` | `max` | ✅ | |
| `number` (arithmetic on other measures) | measure with `sql:` and **no** `aggregate_type` | ✅ | Omni's calculated measure. Rewrite `{other_measure}` → `${other_measure}`. |
| `number_agg: PERCENTILE_CONT(…)` | `percentile` + `percentile: <n>` | ⚠️ | Only when the aggregate is one Omni names. Tesseract-only in Cube. |
| `number_agg: MEDIAN(…)` | `median` | ⚠️ | |
| `number_agg` (anything else) | raw `sql` | ⚠️ | |
| `string` / `time` / `boolean` | a measure with `sql` and **no** `aggregate_type` | ✅ | **Drop the Cube `type:` — it is superfluous.** Omni infers datatype from the warehouse column, and Cube already requires the aggregate inside `sql` for these types, so the expression ports directly: `sql: BOOL_OR(${is_active})`, `sql: MAX(${shipped_at})`, `sql: LISTAGG(${name}, ', ')`. Omni's docs endorse this — *"Don't see an aggregate you want? Use SQL to define the aggregate instead."* Only `{…}` → `${…}` changes. |

### Other measure parameters

| Cube | Omni | | Notes |
|---|---|---|---|
| `sql` | `sql` | ⚠️ | Rewrite `{}` → `${}`. A Cube measure `sql` reads **physical columns**, not other dimensions' overrides — resolve before translating. |
| `title` / `description` | `label` / `description` | ✅ | |
| `public: false` | `hidden: true` | ⚠️ | |
| `format` / `currency` | `format` | ⚠️ | Named-format set; pick precision deliberately. |
| `filters: [{ sql: "{CUBE}.status = 'x'" }]` | `filters:` (Omni filter syntax) | ⚠️ | Grammar differs — Omni is structured, Cube is a SQL predicate. Hand-translate; see [yaml-filter-syntax](../../../../omni-model-builder/references/yaml-filter-syntax.md). |
| `drill_members` | `drill_fields` | ✅ | |
| `rolling_window` | — | ❌ | Query-layer in Omni; not a model field. |
| `multi_stage` + `group_by` / `reduce_by` / `add_group_by` | `level_of_detail` (`fixed` / `always_include` / `always_exclude`) | ⚠️ | Closest analog. **Validate numerically** — filter interaction between stages differs. [level-of-detail](../../../../omni-model-builder/references/level-of-detail.md) |
| `time_shift` | — | ❌ | |
| `grain` | — | ❌ | |
| `case` (on a measure) | `filters:`, or a `CASE` in `sql` | ⚠️ | |
| `mask` | `mask_unless_access_grants`, or companion measures gated by `access_grants` | ⚠️ | As the dimension row above. |
| `meta.ai_context` | `ai_context` | ✅ | |

> ⚠️ **Cube measure `filters` vs. Omni measure `filters` — a real numeric trap,
> verified on both platforms.**
>
> Cube returns **NULL** for groups with no matching rows when the filtered
> measure is queried alongside others. Observed on a live Cube instance:
>
> | status | total_revenue | completed_revenue |
> |---|---|---|
> | processing | 5,540,815 | *(null)* |
> | completed | 5,540,571 | 5,540,571 |
> | shipped | 5,476,747 | *(null)* |
>
> Omni compiles the same measure to a `COALESCE`-wrapped conditional sum, so it
> returns **0**:
>
> ```sql
> COALESCE(SUM(CASE WHEN "IS_NEW_CUSTOMER" THEN "NET_REVENUE" ELSE NULL END), 0)
> ```
>
> So the two platforms legitimately disagree on sparse groupings: NULL on the
> Cube side, 0 on the Omni side. Omni's answer is the more useful one, but say so
> in the parity report rather than treating it as a defect to fix.

---

## Joins

Cube declares a join **on the cube that owns it**; the declaring cube is the
*left* side and the joined cube is the *right*. Omni puts relationships in a
single model-level file named exactly **`relationships`** (global) or in a
topic's `relationships:` block (topic-scoped).

> ⚠️ **There is no `.relationship` file extension in Omni.** Valid model file
> names are `model`, `relationships`, `*.view`, `*.topic` and
> `*.composite_topic`. The `relationships` file's YAML is a **bare top-level
> list** — no `relationships:` key — and writing it **replaces the whole file**,
> so `yaml-get` it first and preserve the existing entries.

| Cube | Omni | | Notes |
|---|---|---|---|
| `joins[].name` | the joined view in `relationships` | ✅ | |
| `relationship: many_to_one` (`belongs_to`) | `relationship_type: many_to_one` | ✅ | |
| `relationship: one_to_many` (`has_many`) | `relationship_type: one_to_many` | ✅ | |
| `relationship: one_to_one` (`has_one`) | `relationship_type: one_to_one` | ✅ | |
| `sql: "{CUBE}.user_id = {users}.id"` | `on_sql: ${orders.user_id} = ${users.id}` | ⚠️ | **Resolve `{CUBE}` to the declaring view's name explicitly.** |
| left-join semantics (implicit) | `join_type` | ⚠️ | Cube's declaring side is always the left of a left join. State the Omni `join_type` rather than leaving it implicit. |
| transitive joins / `join_path` | topic `joins:` nesting | ✅ | Omni's nested `joins:` expresses the same chain. |

> ⚠️ **Fan-out and chasm traps are where numbers diverge.** Cube documents these
> explicitly ([Chasm and fan traps](https://docs.cube.dev/reference/data-modeling/joins.md)),
> and both platforms handle join row multiplication *only* when primary keys are
> correctly declared. Before trusting any post-sync number on a `one_to_many`
> path, confirm `primary_key: true` exists on both sides of every join.

---

## Views → topics

| Cube view parameter | Omni topic | | Notes |
|---|---|---|---|
| `name` | topic file name | ✅ | |
| `cubes[].join_path` | `base_view` + nested `joins:` | ✅ | The first `join_path` root becomes `base_view`. |
| `cubes[].includes` | `fields: [view.a, view.b]` | ✅ | |
| `cubes[].includes: "*"` + `excludes` | `fields: [view.*, -view.x]` | ✅ | Direct analog. |
| `cubes[].prefix: true` | `view_label`, or renamed fields | ⚠️ | Changes resulting names — check collisions *after* translation. |
| `cubes[].alias` / `title` / `description` / `format` | corresponding Omni field params | ⚠️ | Applied per-field in Omni. |
| `measures` / `dimensions` (view-level) | fields in the **base view file**, exposed via `fields:` | ⚠️ | See the callout above — Omni has no view-level field definition. |
| `folders` (flat) | `group_label` | ✅ | |
| `folders` (nested) | `group_label` with a flattened path | ⚠️ | Omni `group_label` is single-level. Encode as `Revenue / Margin` and say it flattened. |
| `default_filters` | `default_filters` | ✅ | Operator vocabularies differ; check each. |
| `meta.default_ui_filters` | `default_filters` | ⚠️ | |
| `meta.auto_run` | `auto_run` | ✅ | |
| `meta.ai_context` | `ai_context` | ✅ | The main AI carrier. |
| `title` / `description` | `label` / `description` | ✅ | |
| `public: false` | `hidden: true` | ⚠️ | |
| `extends` | `extends` | ⚠️ | Verify compiled output. |
| `access_policy` | `access_filters` / `required_access_grants` / `hidden_unless_access_grants` | ⚠️ | User-attribute-dependent; annotate the attributes and verify per role. See the `access_policy[]` row at the top. |

---

## Worked example

### Source — Cube

```yaml
# model/cubes/orders.yml
cubes:
  - name: orders
    sql_table: ANALYTICS_DEV.DBT_JGIRVETZ.FCT_ORDERS
    description: All orders including pending, shipped, and completed

    joins:
      - name: users
        relationship: many_to_one
        sql: "{CUBE}.user_id = {users}.id"

    dimensions:
      - name: id
        sql: id
        type: number
        primary_key: true

      - name: status
        sql: status
        type: string
        description: "Current order status: pending, shipped, or completed"

      - name: created_at
        sql: created_at
        type: time

      - name: net_amount
        sql: "{CUBE}.amount - {CUBE}.refund_amount"
        type: number
        format: currency
        currency: USD

    measures:
      - name: count
        type: count

      - name: total_revenue
        sql: amount
        type: sum
        description: Total revenue from completed orders only
        filters:
          - sql: "{CUBE}.status = 'completed'"
        meta:
          ai_context: Use for all revenue questions. Excludes pending and cancelled orders.

      - name: unique_customers
        sql: user_id
        type: count_distinct

      - name: avg_order_value
        sql: "{total_revenue} / {count}"
        type: number
        format: currency
```

```yaml
# model/views/revenue_overview.yml
views:
  - name: revenue_overview
    description: Revenue metrics and breakdowns
    meta:
      ai_context: >
        Primary view for revenue analysis. Use when users ask about
        sales, revenue, or customer counts.
    cubes:
      - join_path: orders
        includes:
          - count
          - total_revenue
          - unique_customers
          - avg_order_value
          - status
          - created_at
      - join_path: orders.users
        includes:
          - name
          - country
```

### Result — Omni

```yaml
# orders.view
schema: DBT_JGIRVETZ
table_name: FCT_ORDERS
description: All orders including pending, shipped, and completed

dimensions:
  id:
    primary_key: true          # plain column — no sql: needed

  status:
    description: "Current order status: pending, shipped, or completed"

  created_at:
    timeframes: [raw, date, week, month, quarter, year]

  net_amount:
    sql: ${amount} - ${refund_amount}    # {CUBE}.x → ${x}
    format: currency_2

measures:
  count:
    aggregate_type: count

  total_revenue:
    sql: ${amount}
    aggregate_type: sum
    description: Total revenue from completed orders only
    ai_context: Use for all revenue questions. Excludes pending and cancelled orders.
    filters:
      status:
        is: completed

  unique_customers:
    sql: ${user_id}
    aggregate_type: count_distinct

  avg_order_value:
    sql: ${total_revenue} / ${count}     # calculated: no aggregate_type
    format: currency_2
```

```yaml
# relationships   (exact file name; bare top-level list, no `relationships:` key)
- join_from_view: orders
  join_to_view: users
  relationship_type: many_to_one
  join_type: always_left
  on_sql: ${orders.user_id} = ${users.id}
```

```yaml
# revenue_overview.topic
base_view: orders
label: Revenue Overview
description: Revenue metrics and breakdowns
ai_context: >
  Primary view for revenue analysis. Use when users ask about
  sales, revenue, or customer counts.
joins:
  users: {}
fields:
  - orders.count
  - orders.total_revenue
  - orders.unique_customers
  - orders.avg_order_value
  - orders.status
  - orders.created_at
  - users.name
  - users.country
```

### What to report back

```
Translated cube `orders` + view `revenue_overview` → 1 view, 1 relationship, 1 topic.

Gained on the way in (will not survive a return sync):
  • created_at timeframes beyond Cube's granularities (month_name, fiscal_quarter, …)

Verify:
  • total_revenue measure filter — Cube's metric-level filter returns NULL for
    empty groups; Omni's does not. Numbers will differ on sparse groupings.
  • primary_key confirmed on orders.id and users.id (required for the
    many_to_one fan-out to aggregate correctly).

Not mapped: none in this cube.
```

---

## Sources

- Cube: [cube](https://docs.cube.dev/reference/data-modeling/cube.md) · [dimensions](https://docs.cube.dev/reference/data-modeling/dimensions.md) · [measures](https://docs.cube.dev/reference/data-modeling/measures.md) · [joins](https://docs.cube.dev/reference/data-modeling/joins.md) · [view](https://docs.cube.dev/reference/data-modeling/view.md) · [segments](https://docs.cube.dev/reference/data-modeling/segments.md) · [hierarchies](https://docs.cube.dev/reference/data-modeling/hierarchies.md) · [AI context](https://docs.cube.dev/docs/data-modeling/ai-context.md)
- Omni: [modelParameters](../../../../omni-model-builder/references/modelParameters.md) · [dimensions](https://docs.omni.co/modeling/dimensions.md) · [measures](https://docs.omni.co/modeling/measures.md) · [relationships](https://docs.omni.co/modeling/relationships.md) · [topics](https://docs.omni.co/modeling/topics/parameters.md)
