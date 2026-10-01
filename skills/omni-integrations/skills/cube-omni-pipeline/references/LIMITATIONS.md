# Limitations — what does not survive a Cube ↔ Omni sync

Read this before promising a round trip. Neither direction is lossless, and the
losses are not symmetric. Everything below is a **structural** limitation of the
two semantic layers, not a gap in the skills — no amount of translation effort
closes these.

Per-parameter mapping tables live in each direction's own reference:
- [cube-to-omni/references/FIELD-MAPPING.md](../../cube-to-omni/references/FIELD-MAPPING.md)
- [omni-to-cube/references/FIELD-MAPPING.md](../../omni-to-cube/references/FIELD-MAPPING.md)

---

## The six rules that govern everything else

**1. A round trip is not idempotent.** Cube → Omni → Cube does not return the
original files. Names normalize, unmappable parameters drop, and re-derived SQL
is semantically equivalent but textually different. **Never** run a return sync
over a hand-authored model and call it a no-op — diff it first and expect real
changes.

**2. `/v1/meta` has no SQL.** The compiled model omits every `sql` expression, so
a translation driven from metadata alone drops all business logic. Read authored
YAML for logic; read `/v1/meta` for exposure. (See [EDITIONS.md](./EDITIONS.md).)

**3. Both platforms are the source of truth for something.** Pick one side per
object and write it down. Two-way sync of the *same* measure is a merge conflict
waiting to happen, and neither platform will warn you.

**4. Anything unmappable is dropped and then recorded — never approximated.**
An approximation that returns a plausible wrong number is worse than a gap.
Drop it from the YAML and name it in the parent model/topic/view header block,
per [NOTATION.md](./NOTATION.md) — that block is a dropped member's only trace.

**5. Translation preserves definitions, never content.** Dashboards, workbooks,
reports, saved queries, charts, schedules and folders of content do **not**
cross in either direction. Only semantics do.

**6. A compiling model is not a correct model.** Both platforms will happily
build a measure whose SQL is wrong. The parity check in
[BRANCHING.md](./BRANCHING.md#step-3--cross-check-the-numbers) is not optional.

---

## Cube → Omni: what is lost

### Hard blockers — stop and tell the user

| Cube feature | Why it cannot cross |
|---|---|
| **`pre_aggregations`** | Cube Store rollups are an engine feature with partitioning, refresh keys, lambda/streaming tiers and index definitions. Omni's nearest analog is a `materialized_query` view (aggregate awareness), which is a *different mechanism with different matching rules*. Never auto-translate; surface the pre-aggregations you found and let the user decide. [Cube](https://docs.cube.dev/docs/pre-aggregations/index.md) · [Omni](https://docs.omni.co/analyze-explore/performance/aggregate-awareness) |
| **JavaScript / Jinja dynamic models** | Cube models can be generated at compile time by JS or Jinja, including `cube_dbt` and `lkml2cube` loops. Omni YAML is static. Translate the **compiled output**, not the generator, and say so — the generator's intent is gone. [Dynamic modeling](https://docs.cube.dev/docs/data-modeling/dynamic/index.md) |
| **Multi-`data_source` cubes** | A Cube project can join across data sources. An Omni model targets **one connection**. A multi-source Cube view needs either separate Omni models or warehouse-side federation. [Multiple data sources](https://docs.cube.dev/admin/connect-to-data/multiple-data-sources.md) |
| **`rolling_window` measures** | Cube computes rolling windows in the query engine with `trailing`/`leading`/`offset`. Omni has no measure-level rolling window; it expresses these as table calculations or window-function dimensions **at query time**, which means they are not a reusable model field. |

### Degraded — translate, but tell the user what changed

| Cube feature | Omni landing | What degrades |
|---|---|---|
| `segments` | a boolean dimension carrying the segment SQL — defined once in a [`template: true`](https://docs.omni.co/modeling/views/parameters/template.md) view and inherited via `extends` when several views share it; or topic `always_where_filters` / `default_filters` for the enforced and default variants | Omni has no object *named* segment, but the properties that matter — a named, reusable, one-click-filterable condition — all map. A boolean dimension is directly filterable in the UI. ⚠️ A template view is consumable **only through `extends`**: it cannot itself be joined or named in `always_where`. Define the dimension in the template, extend it into the concrete view, then reference the **inherited** dimension. |
| `hierarchies` | `drill_fields` | Cube hierarchies are a named, ordered, reusable structure; Omni `drill_fields` is a per-field drill list. Nesting and the hierarchy's own `title` are lost. |
| `view.folders` (nested) | `group_label` | Omni's `group_label` is **single-level**. Nested folders flatten — encode the path in the label (`Revenue / Margin`) and say it was flattened. [Cube folders](https://docs.cube.dev/docs/organize-content/folders.md) |
| `multi_stage` measures (`group_by` / `reduce_by` / `add_group_by`) | `level_of_detail` (`fixed` / `always_include` / `always_exclude`) | Genuinely the closest analog and worth using, but the semantics differ in edge cases — especially with filters applied between stages. Validate every one numerically. [Omni LOD](https://docs.omni.co/modeling/level-of-detail) |
| `measure.time_shift` / `dimension.time_shift` | none at model level | Omni does period-over-period in the query layer. The model field disappears. |
| `count_distinct_approx` | `count_distinct` | **Changes the numbers.** HyperLogLog is approximate and additive; `count_distinct` is exact and non-additive. The value will differ and rollups behave differently. Never translate silently. |
| `number_agg` (e.g. `PERCENTILE_CONT`) | `percentile` + `percentile:` param, or raw `sql` | Only maps when the aggregate is one Omni names. Tesseract-only on the Cube side. |
| `type: string` / `time` / `boolean` measures | a measure with `sql` and **no** `aggregate_type` | ✅ **Drop the type declaration entirely.** Omni infers datatype from the warehouse column, so the Cube `type:` is superfluous. Cube already requires the aggregate inside the `sql` for these types, so the expression ports directly: `sql: BOOL_OR(${is_active})`, `sql: MAX(${shipped_at})`. Omni's docs endorse this — *"Don't see an aggregate you want? Use SQL to define the aggregate instead."* Only the `{…}` → `${…}` rewrite is needed. |
| `dimension.type: geo` | two dimensions (`latitude`, `longitude`) | Omni has no single geo type; map visualization is configured on the chart, not the field. |
| `dimension.type: switch` | a [filter-only field](https://docs.omni.co/modeling/templated-filters/index.md) in the view's `filters:` block with `suggestion_list` + `filter_single_select_only` | The enum constraint **is** enforceable — `suggestion_list` fixes the allowed values and `filter_single_select_only` forces a single choice, which is what a `switch` dimension is for. `groups` remains the alternative when the values bucket an existing column rather than parameterize a query. ⚠️ `display_order` is **not** a substitute: it orders the field in the picker and constrains nothing. Tesseract-only on the Cube side. |
| `dimension.granularities` (custom, e.g. `two_weeks`) | a derived dimension holding the bucketing SQL, given the base timestamp's `group_label` so it sits **alongside** the default timeframes in the picker | Omni's `timeframes` is a closed list, but a custom grain is just a dimension: `sql: DATE_TRUNC('week', ${created_at}) - (EXTRACT(week FROM ${created_at})::int % 2) * INTERVAL '1 week'` with `group_label: Created At`. Calendar-based grains map through [`custom_calendars`](https://docs.omni.co/modeling/models/custom-calendars.md) `mappings` (`week_of_year`, `day_of_quarter`, …) instead. ⚠️ [`duration`](https://docs.omni.co/modeling/dimensions/parameters/duration.md) is **not** the tool here — it measures elapsed time between `sql_start` and `sql_end`, not buckets of one timestamp. |
| `sub_query` dimensions | `level_of_detail: fixed`, or a [query view](https://docs.omni.co/modeling/views/parameters/query.md) | Cube's `sub_query` references a measure inside a dimension; `level_of_detail: fixed` is the direct analog. **`propagate_filters_to_sub_query` does have an equivalent** — a query view's [`bind_all_filters`](https://docs.omni.co/modeling/query-views/parameters/index.md) (*"passes all chosen filters from the outer query into the query view"*), with `bind` for specific fields and `bind_outer_context` for context operators. ⚠️ If Cube is propagating predicates into a **pre-aggregation**, that is caching behavior Omni has no equivalent for — Omni pushes the predicate into the subquery at query time instead. Validate numerically either way. |
| `meta.member_masking` / dimension `mask` (default `NULL`) | [`mask_unless_access_grants`](https://docs.omni.co/modeling/dimensions/parameters/mask-unless-access-grants.md) | Masks the **value** rather than hiding the field, so aggregation and grouping still work — Cube's actual use case. Omni's built-in mask is an **MD5 hash**, not a value you choose. |
| `mask: -1` / `mask: {sql: …}` (a chosen replacement value) | one companion dimension per masking variant, each gated by complementary [`access_grants`](https://docs.omni.co/modeling/models/access-grants.md) | ✅ **Arbitrary mask shapes are achievable**, and with more tiers than Cube allows: define `code_full`, `code_partial` (`sql: CONCAT('***', RIGHT(${code}, 3))`), `code_hidden`, and gate each with a grant keyed to a different `allowed_values` of one user attribute. Two grants on the same attribute (`allowed_values: ["true"]` / `["false"]`) give the inverse condition. ⚠️ Costs: one field per variant; the grants must be mutually exclusive and exhaustive or a user sees both or neither; and **access grants apply only to direct field access — they do not propagate through a measure that references the masked dimension.** Re-test per role. |
| `calendar` cubes | [`custom_calendars`](https://docs.omni.co/modeling/models/custom-calendars.md) at model level, or `fiscal_month_offset` + `fiscal_*` timeframes | Retail **4-5-4 is a documented Omni example**; the `mappings` block points each custom period (`year`, `quarter`, `month`, `week`, `week_of_year`, `day_of_quarter`, …) at a column in your calendar table, and Omni joins it on the date without fan-out. ⚠️ **A custom calendar *replaces* the fiscal calendar on the dimensions that use it** — `fiscal_month_offset` and every `fiscal_*` timeframe become unavailable **on those dimensions**. The two coexist in one model across *different* dimensions, never on the same one. [Setup guide](https://docs.omni.co/guides/modeling/custom-calendars.md) |
| `access_policy.row_level` | [`access_filters`](https://docs.omni.co/modeling/topics/parameters/access-filters.md) | Both are user-attribute-driven RLS, but the condition grammar and the attribute plumbing differ. The sync writes the parameter; it **cannot create or assign the [user attribute](https://docs.omni.co/administration/users/attributes.md)** it reads — and a missing one **does not fail validation**: the model validates, the query runs, and the filter silently does not constrain. **Re-test per role after every sync** with [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md). A silently broken access filter is the worst failure mode in this pipeline. |
| `access_policy.member_level` | [`required_access_grants`](https://docs.omni.co/modeling/topics/parameters/required-access-grants.md) / `hidden_unless_access_grants` | Grant model differs. Same [user-attribute](https://docs.omni.co/administration/users/attributes.md) dependency and same silent-failure mode as the row above. ⚠️ Omni grants apply only to **direct field access** — they do not propagate through a measure that references the granted field, so test the measure too. Verify per role with [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md). |
| `refresh_key` | topic `cache_policy` | Different mechanism (Cube invalidates a cache key; Omni applies a cache policy). Not a faithful translation. |

---

## Omni → Cube: what is lost

### Hard blockers

| Omni feature | Why it cannot cross |
|---|---|
| **AI model metadata beyond text** | Omni's `synonyms`, `sample_queries`, `ai_fields`, `all_values`, `sample_values` have **no Cube parameters**. The only carrier is `meta.ai_context` free text, capped at **2,000 characters per value, silently truncated** beyond that. Fold what fits into prose and report what was dropped. [Cube AI context](https://docs.cube.dev/docs/data-modeling/ai-context.md) |
| **Composite topics** | An Omni `.composite_topic` unions multiple topics with `shared_dimensions` mappings and `unrelated_dimension_handling`. Cube's nearest construct is a [multi-fact view](https://docs.cube.dev/docs/data-modeling/multi-fact-views.md), which does not reproduce the mapping-per-topic behavior. Do not attempt automatically. |
| **Workbook-model customizations** | Fields defined in a workbook's own model layer are document-scoped and are not part of the shared model. They are out of scope — promote them to the shared model first if they should cross. |
| **`*_distinct_on` aggregates** | `sum_distinct_on` / `average_distinct_on` / `median_distinct_on` / `percentile_distinct_on` dedupe via `custom_primary_key_sql`. Cube has no dedup-aware aggregate; this needs a pre-deduplicated cube (a `sql:` derived cube with `DISTINCT`/`ROW_NUMBER`) — a modeling change, not a translation. |
| **`dynamic_top_n`, `duration`, `colors`, `display_order` for values** | Query- and presentation-layer conveniences with no Cube counterpart. They drop. (`groups`/`else` and `bin_boundaries` are **not** in this list — both map to Cube's dimension `case`; see the Degraded table.) |

### Degraded

| Omni feature | Cube landing | What degrades |
|---|---|---|
| Topic | View | Good structural match — but a topic's `always_where_sql` / `always_having_*` become a `sql` predicate on the underlying cube or a `default_filters` entry, and `always_having_*` has no clean home at all (Cube filters pre-aggregation, not post-aggregation, in the same place). |
| `fields: [view.*, -view.x]` | `includes` / `excludes` | Close analog. Omni's wildcard-plus-exclusion ordering does not always have a one-line Cube equivalent; enumerate when in doubt. |
| `default_filters` | `meta.default_ui_filters` on the view | Cube's version drives UI defaults; operator vocabularies differ. |
| `timeframes` | `granularities` + a `time` dimension | Omni's non-date-part timeframes (`day_of_week_name`, `month_name`, `fiscal_quarter`, …) are **not** Cube granularities. Each becomes its own derived dimension with explicit SQL. This is the single most verbose part of the translation. |
| `median` aggregate | `number_agg` with `MEDIAN(…)` | **Tesseract-only** on the Cube side; unavailable on older Core versions. |
| `list` aggregate | `type: string` with `LISTAGG`/`STRING_AGG` | Dialect-specific SQL; not portable across Cube data sources. |
| Measure `filters` | Measure `filters` | Both exist, but Omni's filter grammar is structured and Cube's is `sql:` predicates using `{CUBE}`. Hand-translate each. ⚠️ **Verified numeric difference:** a Cube measure-level `filter` returns **NULL** for groups with no matching rows when queried alongside other measures, while Omni compiles to `COALESCE(SUM(CASE WHEN … END), 0)` and returns **0**. Prefer `CASE WHEN … THEN 1 END` inside a Cube `sql` to preserve Omni's behavior. |
| `groups:` + `else:` | dimension `case:` (`when[].sql` / `when[].label` + `else.label`) | Direct structural analog — Cube's `case` is CASE-like bucketing with an else fallback, same as Omni's `groups`. ⚠️ On the Cube side `case` **replaces** `sql` on that dimension; declaring both fails the build. |
| `bin_boundaries` | dimension `case:` with one `when` per bin | Maps, but the boundaries become explicit SQL predicates rather than a list of numbers, so the intent is less legible and the bins must be maintained by hand. |
| Filter-only field used as a **constrained enum picker** | `dimension.type: switch` + `values` | Maps when the filter-only field exists to offer a fixed set of choices (`suggestion_list` + `filter_single_select_only`). **Tesseract-only** on the Cube side. |
| Filter-only field used for **Mustache value injection** (`{{filters.v.f.value}}` into arbitrary SQL, `bind_to`, metric switchers) | — | ❌ No Cube analog. A `switch` dimension constrains values; it does not splice a chosen value into another field's SQL. A security-context variable is the nearest thing and is not equivalent. This half of templated filters does not cross. |
| `drill_fields` / `drill_queries` | `drill_members` | `drill_members` is a flat member list; `drill_queries` (multiple named drills with full query control) has no equivalent. |
| `links` | `meta` | Cube dimension `links` exists but with a different templating contract; verify per link. |
| `level_of_detail` | `multi_stage` measures | The reverse of the mapping above, with the same caveat: validate numerically. |
| `access_filters` | `access_policy.row_level` | Cube resolves a **security context per request**; Omni resolves [user attributes](https://docs.omni.co/administration/users/attributes.md) per user. The shape maps, the plumbing does not — wiring attributes to a security context is a separate task. Confirm the Omni side is not already inert with [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md) **before** exporting, then re-test on the Cube side with a scoped token. |
| `materialized_query` views | `pre_aggregations` | Do not auto-translate. Report and let the user decide. |
| View `extends` | `extends` | Both support inheritance, but resolution order and override semantics differ; check the compiled output on both sides rather than trusting the file. |

---

## Naming and identifier hazards

| Hazard | Detail |
|---|---|
| **Field-reference syntax** | Cube uses `{field}` and `{CUBE}.column` in YAML (`${…}` in JavaScript). Omni uses `${field}` / `${view.field}`. A mechanical copy of a `sql` expression between them is always wrong. |
| **`${TABLE}` does not exist in Omni** | A LookML-ism. It fails `omni models validate` with `Column "__omni_scoped" not found`. In Omni, a plain column dimension needs no `sql:` at all. |
| **Cube's `{CUBE}` is the declaring cube** | In a join `sql`, `{CUBE}` is the cube that declares the join (the *left* side) and the other cube is named directly. Resolve it to an explicit Omni `${view.field}` before writing. |
| **Reserved granularity names** | A Cube dimension may not be named `day`, `week`, `month`, `quarter`, `year`, `hour`, `minute`, or `second`. An Omni dimension with one of those names must be renamed on the way in (e.g. `month_dt` with `sql: month`). |
| **Project-wide uniqueness** | Cube member names are unique within a cube/view; Omni field names are unique within a view. Prefixing (`view.cubes[].prefix`) changes the resulting names — check for collisions after translation, not before. |
| **Case conventions** | Both prefer `snake_case`. Cube `/v1/meta` returns some legacy camelCase keys (`aliasName`, `shortTitle`, `drillMembers`, `connectedComponent`); do not carry those spellings into YAML. |

---

## Sources

- [Cube data modeling reference](https://docs.cube.dev/reference/data-modeling/cube.md) · [measures](https://docs.cube.dev/reference/data-modeling/measures.md) · [dimensions](https://docs.cube.dev/reference/data-modeling/dimensions.md) · [joins](https://docs.cube.dev/reference/data-modeling/joins.md) · [views](https://docs.cube.dev/reference/data-modeling/view.md) · [segments](https://docs.cube.dev/reference/data-modeling/segments.md) · [hierarchies](https://docs.cube.dev/reference/data-modeling/hierarchies.md) · [pre-aggregations](https://docs.cube.dev/reference/data-modeling/pre-aggregations.md) · [data access policies](https://docs.cube.dev/reference/data-modeling/data-access-policies.md)
- Omni: [`omni-model-builder` modelParameters](../../../../omni-model-builder/references/modelParameters.md) · [level of detail](../../../../omni-model-builder/references/level-of-detail.md) · [aggregate awareness](../../../../omni-model-builder/references/aggregate-awareness.md) · [templated filters](../../../../omni-model-builder/references/templated-filters.md) · [composite topics](../../../../omni-model-builder/references/composite-topics.md)
