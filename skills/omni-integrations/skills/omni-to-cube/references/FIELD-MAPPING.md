# Omni → Cube field mapping

Parameter-by-parameter translation from Omni model YAML to a Cube data model.
Legend: ✅ direct · ⚠️ lossy or requires judgment · ❌ no mapping (report it).

This direction loses **more** than Cube → Omni, almost entirely because Omni's
AI-modeling and query-layer parameters have no Cube counterparts. Read
[LIMITATIONS.md](../../cube-omni-pipeline/references/LIMITATIONS.md) before
promising a faithful export, and
[NOTATION.md](../../cube-omni-pipeline/references/NOTATION.md) for the lineage
comment every written member carries — applied in reverse here (`# omni:` for a
ported member, `# cube-native:` for one with no Omni ancestor), with every
**dropped** Omni member named in the parent cube's or view's header block.

---

## Object-level mapping

| Omni | Cube | | Notes |
|---|---|---|---|
| `.view` file (with `schema`/`table_name`) | `cubes[]` entry with `sql_table` | ✅ | |
| `.view` file with `sql:` (query view) | `cubes[]` entry with `sql:` | ✅ | |
| `.topic` file | `views[]` entry | ✅ | **The key mapping.** |
| the model's `relationships` file / topic `relationships` | `joins[]` on the left cube | ⚠️ | Omni relationships are standalone (a bare top-level list in a file named exactly `relationships`); Cube joins are declared **on** a cube. Pick the owning cube deliberately — see [Relationships](#relationships). |
| `.composite_topic` file | multi-fact view | ❌ | Do not attempt automatically. |
| `materialized_query` view | `pre_aggregations[]` | ❌ | Report; let the user decide. |
| Workbook-model fields | — | ❌ | Document-scoped; promote to the shared model first. |
| `access_filters` | `access_policy[].row_level` | ⚠️ | **Depends on [user attributes](https://docs.omni.co/administration/users/attributes.md).** The enforcement models differ — Omni resolves attributes per user, Cube resolves a security context per request — so the policy must be **re-tested per role on the Cube side**. A silently-inert policy is the worst outcome of this whole pipeline. Validate the Omni side with [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md) before syncing, and the Cube side with a scoped token afterwards. |
| `required_access_grants` | `access_policy[].member_level` | ⚠️ | User-attribute-dependent; re-test per role. ⚠️ Omni grants apply only to **direct** field access and do not propagate through a measure referencing the field — confirm the Cube equivalent covers the measure too. |
| `mask_unless_access_grants` | `mask` + a data-masking `access_policy` | ✅ | Cube is *more* expressive here: `mask` takes a chosen value or SQL expression (`mask: -1`, `mask: {sql: "CONCAT('***', …)"}`), where Omni always MD5s. An Omni companion-dimension masking scheme (several variants gated by complementary grants) **collapses into one Cube dimension** with a `mask` — simplify rather than porting the variants one by one. |
| View-level `filters:` block — constrained enum picker | `dimension.type: switch` + `values` | ⚠️ | Maps when the filter-only field exists to offer fixed choices. **Tesseract-only.** |
| View-level `filters:` block — Mustache value injection (`{{filters.v.f.value}}`, `bind_to`, metric switchers) | — | ❌ | No Cube analog. A `switch` constrains values; it does not splice a chosen value into another field's SQL. Report as dropped. |

> **Where Omni puts a measure matters.** Every Omni dimension and measure lives
> in a **view file**; the topic only curates. Cube allows measures on either a
> cube or a view, and its convention is that business metrics belong on the
> **view**. So you have a choice on export: keep the measure on the cube
> (mirrors Omni's file layout) or lift it to the view (mirrors Cube's
> convention). **Match the target project's existing layout** — as
> `cube-build-model` puts it, *"a correct cube that looks nothing like its
> neighbours is a bad contribution."* Read the project with
> `cube-explore-model` first.

---

## View → cube

| Omni | Cube | | Notes |
|---|---|---|---|
| view file name | `name` | ✅ | `snake_case` both. |
| `schema:` + `table_name:` | `sql_table: schema.table` | ✅ | Recombine into a qualified name. |
| `sql:` on the view | `sql:` | ✅ | |
| `label` | `title` | ✅ | |
| `description` | `description` | ✅ | |
| `hidden: true` | `public: false` | ⚠️ | Cube's `public: false` also removes the member from `/v1/meta`; Omni's `hidden` does not hide it from SQL. |
| `ignored: true` | omit the member entirely | ⚠️ | |
| `extends` | `extends` | ⚠️ | Verify compiled output on both sides. |
| `ai_context` | `meta.ai_context` | ⚠️ | **2,000 chars per value, silently truncated.** Cube ignores cube-level `ai_context` — put it on the **view** or the member, never the cube. |
| `tags` | `meta` | ⚠️ | Free-form; no schema preserved. |
| `cache_policy` (topic) | `refresh_key` | ⚠️ | Different mechanism. |

---

## Dimensions

| Omni | Cube | | Notes |
|---|---|---|---|
| dimension key | `name` | ⚠️ | **Rename if it is `day`, `week`, `month`, `quarter`, `year`, `hour`, `minute`, or `second`** — those are reserved granularity keywords and `dbt parse`-equivalent validation fails. Use `month_dt` with `sql: month`. |
| no `sql:` (plain column) | `sql: <column>` + `type:` | ✅ | **Cube requires both.** Omni infers; Cube does not. Look up the column type from the warehouse or Omni's schema layer. |
| `sql: ${a} + ${b}` | `sql: "{a} + {b}"` | ⚠️ | Rewrite `${field}` → `{field}`, `${view.field}` → `{view.field}`, and a bare column → `{CUBE}.column`. |
| (string column) | `type: string` | ✅ | |
| (numeric column) | `type: number` | ✅ | |
| (boolean column) | `type: boolean` | ✅ | |
| `timeframes: [...]` | `type: time` + `granularities` | ⚠️ | See [Timeframes](#timeframes) — the most verbose part of this direction. |
| `primary_key: true` | `primary_key: true` | ✅ | **Required** for Cube to handle join row multiplication. |
| `label` | `title` | ✅ | |
| `description` | `description` | ✅ | |
| `hidden: true` | `public: false` | ⚠️ | |
| `format: currency_2` | `format: currency` + `currency: USD` | ⚠️ | Omni encodes precision in the name; Cube splits format from currency. Precision is lost. |
| `format: percent_2` | `format: percent` | ⚠️ | Precision lost. |
| `group_label` | view `folders` | ⚠️ | Only expressible on the Cube **view**, not the cube. |
| `display_order` | `order` | ⚠️ | |
| `order_by_field` | — | ❌ | |
| `groups:` + `else:` | dimension `case:` (`when[].sql` / `when[].label`, `else.label`) | ✅ | Direct structural analog. ⚠️ On the Cube side `case` **replaces** `sql` on that dimension — declaring both fails the build with `(dimensions.<name>.sql …) is not allowed`. |
| `bin_boundaries` | dimension `case:` with one `when` per bin | ⚠️ | Maps, but a list of numbers becomes explicit SQL predicates, so the intent is less legible and the bins are maintained by hand. |
| `duration` (`sql_start` / `sql_end` / `intervals`) | one dimension per interval, each with dialect datediff SQL | ⚠️ | Omni generates `${dim[hours]}`, `${dim[days]}`, … from one declaration; Cube needs a separate dimension per interval with explicit SQL (`DATEDIFF('day', …)`), which is dialect-specific. Emit only the intervals the source declared. |
| `level_of_detail: fixed` | `sub_query: true`, or a `multi_stage` measure | ⚠️ | Depends on intent; validate numerically. |
| `drill_fields` | measure `drill_members` | ⚠️ | Cube puts drills on **measures**, not dimensions. |
| `links` | `links` | ⚠️ | Different templating contract. |
| `synonyms` | fold into `meta.ai_context` | ❌ | No Cube parameter. |
| `all_values` / `sample_values` | fold into `meta.ai_context` | ❌ | No Cube parameter. |
| `ai_context` | `meta.ai_context` | ⚠️ | 2,000-char cap. |
| `colors` | — | ❌ | Presentation-layer. |
| `dynamic_top_n` | — | ❌ | Query-layer. |
| `suggestion_list` | `type: switch` + `values` | ⚠️ | Tesseract-only in Cube. |
| `suggest_from_field` / `suggest_from_topic` | — | ❌ | |
| `filter_single_select_only` | — | ❌ | |
| `convert_tz: false` | — | ❌ | Handle in `sql` or via Cube's timezone config. |
| `required_access_grants` | `access_policy[].member_level` | ⚠️ | User-attribute-dependent; re-test per role. ⚠️ Omni grants apply only to **direct** field access and do not propagate through a measure referencing the field — confirm the Cube equivalent covers the measure too. |
| `aliases` | — | ❌ | Omni's content-preservation mechanism; no Cube analog. |

### Timeframes

This is where Omni → Cube gets verbose. Omni's `timeframes` mixes true
granularities with date *parts*; Cube only has granularities.

| Omni timeframe | Cube | |
|---|---|---|
| `raw` | the `type: time` dimension itself | ✅ |
| `date` | granularity `day` | ✅ |
| `week`, `month`, `quarter`, `year`, `hour`, `minute`, `second` | same-named granularity | ✅ |
| `millisecond` | — | ❌ |
| `day_of_week_name`, `day_of_week_num`, `day_of_month`, `day_of_quarter`, `day_of_year`, `hour_of_day`, `month_name`, `month_num`, `quarter_of_year` | **a separate derived dimension each**, with explicit dialect SQL | ⚠️ |
| `fiscal_quarter`, `fiscal_year` | a [calendar cube](https://docs.cube.dev/docs/data-modeling/concepts/calendar-cubes.md), or derived dimensions using the `fiscal_month_offset` | ⚠️ |

So an Omni dimension with the default timeframes plus `month_name` becomes one
Cube `type: time` dimension *plus* one extra `type: string` dimension:

```yaml
- name: created_at
  sql: created_at
  type: time

- name: created_at_month_name
  sql: "TO_CHAR({CUBE}.created_at, 'Month')"   # dialect-specific
  type: string
```

Only emit the extra dimensions the source model actually declared. Emitting all
nine per date field produces an unusable model.

---

## Measures

### Aggregate type

| Omni `aggregate_type` | Cube `type` | | Notes |
|---|---|---|---|
| `sum` | `sum` | ✅ | |
| `count` | `count` | ✅ | |
| `count_distinct` | `count_distinct` | ✅ | |
| `average` | `avg` | ✅ | Note the spelling difference. |
| `min` | `min` | ✅ | |
| `max` | `max` | ✅ | |
| `median` | `number_agg` with `MEDIAN(…)` | ⚠️ | **Tesseract-only.** Unavailable on older Cube Core. |
| `percentile` (+ `percentile: 75`) | `number_agg` with `PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY …)` | ⚠️ | **Tesseract-only**, and the SQL is dialect-specific. |
| `list` | `type: string` with `LISTAGG`/`STRING_AGG` | ⚠️ | Dialect-specific; not portable across Cube data sources. |
| no `aggregate_type` (calculated, has `sql`) | `type: number` | ✅ | Rewrite `${m}` → `{m}`. |
| `sum_distinct_on` | — | ❌ | Needs a pre-deduplicated cube. A modeling change, not a translation. |
| `average_distinct_on` | — | ❌ | Same. |
| `median_distinct_on` | — | ❌ | Same. |
| `percentile_distinct_on` | — | ❌ | Same. |

### Other measure parameters

| Omni | Cube | | Notes |
|---|---|---|---|
| `sql` | `sql` | ⚠️ | Rewrite `${}` → `{}`. A Cube measure `sql` reads **physical columns** — if the Omni measure referenced a dimension with a model-layer `sql` override, inline the override or the measure is wrong. |
| `label` / `description` | `title` / `description` | ✅ | |
| `hidden: true` | `public: false` | ⚠️ | |
| `format` | `format` + `currency` | ⚠️ | Precision lost. |
| `filters:` | `filters: [{ sql: … }]` | ⚠️ | Omni's grammar is structured; Cube's is a SQL predicate with `{CUBE}`. Hand-translate each. |
| `drill_fields` | `drill_members` | ✅ | |
| `drill_queries` | — | ❌ | Multiple named drills with query control; no analog. |
| `level_of_detail` | `multi_stage` + `group_by` / `reduce_by` | ⚠️ | Validate numerically. |
| `custom_primary_key_sql` | — | ❌ | Tied to the `*_distinct_on` family. |
| `ai_context` | `meta.ai_context` | ⚠️ | 2,000-char cap. |
| `synonyms` / `all_values` / `sample_values` | fold into `meta.ai_context` | ❌ | |
| `colors` / `display_order` / `view_label` | — / `order` / — | ❌/⚠️ | Presentation-layer. |
| `required_access_grants` | `access_policy[].member_level` | ⚠️ | User-attribute-dependent; re-test per role. ⚠️ Omni grants apply only to **direct** field access and do not propagate through a measure referencing the field — confirm the Cube equivalent covers the measure too. |

> ⚠️ **The filtered-count trap, in this direction.** A Cube measure-level
> `filter` returns NULL for groups with no matching rows when queried alongside
> other measures. An Omni measure with `filters:` does not behave that way. To
> preserve Omni's behavior in Cube, build the filtered aggregate as
> `CASE WHEN … THEN 1 END` inside `sql` rather than using `filters:`. This is the
> same guidance the `omni-to-dbt-metricflow` skill gives for MetricFlow, and for
> the same reason.

---

## Relationships

An Omni relationship is a standalone object; a Cube join lives **on a cube**.
You must choose which cube owns it, and that choice sets the join direction:
the declaring cube is the **left** side of a left join.

| Omni | Cube | | Notes |
|---|---|---|---|
| `join_from_view` | the cube that declares `joins[]` | ✅ | The left side. |
| `join_to_view` | `joins[].name` | ✅ | The right side. |
| `relationship_type: many_to_one` | `relationship: many_to_one` | ✅ | |
| `relationship_type: one_to_many` | `relationship: one_to_many` | ✅ | |
| `relationship_type: one_to_one` | `relationship: one_to_one` | ✅ | |
| `on_sql: ${orders.user_id} = ${users.id}` | `sql: "{CUBE}.user_id = {users}.id"` | ⚠️ | The declaring view's references become `{CUBE}.`; the other view stays named. |
| `join_type: always_left` / `inner` / `full_outer` | — | ⚠️ | Cube joins are left joins from the declaring cube. A non-left join type does **not** translate — report it. |
| topic-scoped `relationships` | per-view `joins` + `join_path` | ⚠️ | A relationship that exists only inside one Omni topic may need a distinct Cube view rather than a cube-level join. |

---

## Topic → view

| Omni | Cube | | Notes |
|---|---|---|---|
| topic file name | `name` | ✅ | |
| `base_view` | first `cubes[].join_path` | ✅ | |
| `joins: { users: {} }` | additional `cubes[].join_path: orders.users` | ✅ | Omni's nesting becomes dotted join paths. |
| `fields: [v.a, v.b]` | `cubes[].includes: [a, b]` | ✅ | |
| `fields: [v.*, -v.x]` | `includes: "*"` + `excludes: [x]` | ✅ | |
| `label` / `description` | `title` / `description` | ✅ | |
| `hidden: true` | `public: false` | ⚠️ | |
| `ai_context` | `meta.ai_context` | ⚠️ | 2,000-char cap. **This is the one place Cube's agent reads it** (view- or member-level only). |
| `sample_queries` | fold into `meta.ai_context` prose | ❌ | No Cube parameter. |
| `ai_fields` | `includes` / `excludes`, or `public: false` | ⚠️ | Approximate: Cube has no AI-specific field curation, so narrowing for AI narrows for everyone. |
| `default_filters` | `meta.default_ui_filters` | ⚠️ | Operator vocabularies differ. |
| `always_where_sql` | a `sql:` predicate on the underlying cube | ⚠️ | Changes the cube for *all* views that use it — a real blast-radius difference. |
| `always_where_filters` | same | ⚠️ | Same caveat. |
| `always_having_sql` / `always_having_filters` | — | ❌ | Cube has no post-aggregation always-filter. |
| `access_filters` | `access_policy[].row_level` | ⚠️ | User-attribute-dependent; see the object-level row. |
| `auto_run` | `meta.auto_run` | ✅ | |
| `group_label` | view `folders`, or a [view group](https://docs.cube.dev/docs/data-modeling/view-groups.md) | ⚠️ | |
| `default_row_limit` | — | ❌ | |
| `owners` | `meta` | ⚠️ | |
| `extends` | `extends` | ⚠️ | |
| `template: true` | a cube/view intended only for `extends` | ⚠️ | No explicit Cube flag; use `public: false`. |
| `warehouse_override` | `data_source` | ⚠️ | Different granularity. |

---

## Worked example

### Source — Omni

```yaml
# order_items.view
schema: DBT_JGIRVETZ
table_name: FCT_ORDER_ITEMS
dimensions:
  id:
    primary_key: true
  created_at:
    timeframes: [raw, date, month, quarter, year, month_name]
  margin_pct:
    sql: (${sale_price} - ${cost}) / NULLIF(${sale_price}, 0)
    format: percent_2
measures:
  total_margin:
    sql: ${sale_price} - ${cost}
    aggregate_type: sum
    ai_context: Gross margin in dollars. Excludes returns.
    synonyms: [gross margin, GM]
  median_order_value:
    sql: ${sale_price}
    aggregate_type: median
```

### Result — Cube

```yaml
# model/cubes/order_items.yml
cubes:
  - name: order_items
    sql_table: ANALYTICS_DEV.DBT_JGIRVETZ.FCT_ORDER_ITEMS

    dimensions:
      - name: id
        sql: id
        type: number              # Cube requires explicit sql + type
        primary_key: true

      - name: created_at
        sql: created_at
        type: time                # covers raw/date/month/quarter/year

      - name: created_at_month_name
        sql: "TO_CHAR({CUBE}.created_at, 'Month')"   # month_name has no granularity
        type: string

      - name: margin_pct
        sql: "({CUBE}.sale_price - {CUBE}.cost) / NULLIF({CUBE}.sale_price, 0)"
        type: number
        format: percent           # precision (_2) lost

    measures:
      - name: total_margin
        sql: "{CUBE}.sale_price - {CUBE}.cost"
        type: sum
        meta:
          ai_context: >
            Gross margin in dollars. Excludes returns.
            Also known as: gross margin, GM.

      - name: median_order_value
        sql: "MEDIAN({CUBE}.sale_price)"
        type: number_agg          # ⚠️ Tesseract only
```

### What to report back

```
Exported view `order_items` → cube `order_items` (3 dimensions + 1 derived, 2 measures).

Degraded:
  • margin_pct — format percent_2 → percent; decimal precision lost.
  • created_at — month_name timeframe became a separate derived dimension
    (created_at_month_name) with Snowflake-specific TO_CHAR SQL.
  • total_margin — `synonyms: [gross margin, GM]` has no Cube parameter; folded
    into meta.ai_context prose.

Requires attention:
  • median_order_value uses type: number_agg, which is Tesseract-only. Confirm
    the target deployment runs Tesseract or this cube will not build.

Not mapped: none blocking.
```

---

## Sources

- Omni: [modelParameters](../../../../omni-model-builder/references/modelParameters.md) · [yaml-filter-syntax](../../../../omni-model-builder/references/yaml-filter-syntax.md) · [level-of-detail](../../../../omni-model-builder/references/level-of-detail.md) · [dimensions](https://docs.omni.co/modeling/dimensions.md) · [measures](https://docs.omni.co/modeling/measures.md) · [topics](https://docs.omni.co/modeling/topics/parameters.md)
- Cube: [cube](https://docs.cube.dev/reference/data-modeling/cube.md) · [dimensions](https://docs.cube.dev/reference/data-modeling/dimensions.md) · [measures](https://docs.cube.dev/reference/data-modeling/measures.md) · [joins](https://docs.cube.dev/reference/data-modeling/joins.md) · [view](https://docs.cube.dev/reference/data-modeling/view.md) · [view groups](https://docs.cube.dev/docs/data-modeling/view-groups.md) · [AI context](https://docs.cube.dev/docs/data-modeling/ai-context.md)
