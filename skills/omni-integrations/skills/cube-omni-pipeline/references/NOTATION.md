# Lineage notation — marking what came from Cube and what was added in Omni

A synced model is read by people who did not run the sync. Six months on, the
only question that matters about any given field is: **did this come from Cube,
was it added here, or is it half of each?** This file defines how to answer that
in the YAML itself.

Two rules govern the whole scheme:

1. **One comment per object, at the most granular level that object owns.** A
   field gets one comment above it — never a comment per sub-parameter. A view,
   topic or model gets one header block.
2. **Be explicit inside that one comment.** The budget is a single comment, so
   spend it on specifics — which Cube member, which file, what changed — not on
   restating what the mapping docs already say.

The goal is a file whose *definitions* stay readable. A reader skimming for the
`sql` of a measure must not have to wade through three lines of provenance on
every key.

> **Comments survive.** Verified against a live Omni model: comments written
> through `yaml-create` come back intact from `yaml-get`, and field-level
> comments **stay attached to their key** through Omni's canonicalization.
> One caveat — Omni re-serializes the file, hoisting `label` / `description`
> above a header comment block and prepending its own
> `# Reference this view as …`. A header block survives but will not stay on
> line one, so never write a comment whose meaning depends on its position.

---

## Field-level: one line, three forms

Every dimension and measure carries exactly one comment, and it opens with a
greppable token.

### Fully from Cube — nothing changed but syntax

```yaml
  # cube: conversions.total_net_revenue · model/cubes/conversions.yml
  total_net_revenue:
    sql: ${net_revenue}
    aggregate_type: sum
    description: Net revenue after discounts
```

`{…}` → `${…}` rewriting and `type:` → `aggregate_type:` do **not** count as
changes. They are the translation, not an addition.

### Partially from Cube — some sub-parameters are Omni's

Name what Omni added or altered, and why, on one continuation line at most.

```yaml
  # cube: conversions.margin_pct · model/cubes/conversions.yml · PARTIAL
  #   omni: format percent_2 (Cube `percent` carries no precision); timeframes added
  margin_pct:
    sql: ${gross_margin} / NULLIF(${net_revenue}, 0)
    format: percent_2
```

### Added in Omni — no Cube ancestor

```yaml
  # omni-native: added for AI field curation; no Cube equivalent
  customer_tier:
    sql: ${tier}
    ai_context: Segment used in all pipeline reporting.
```

### When a value needs checking after promotion

Append `· VERIFY: <what>` to the same comment rather than adding a second one.

```yaml
  # cube: conversions.new_customer_revenue · model/cubes/conversions.yml · VERIFY: NULL-vs-0 on sparse groups
  new_customer_revenue:
    sql: ${net_revenue}
    aggregate_type: sum
    filters:
      is_new_customer:
        is: true
```

### The grammar

```
# cube: <cube>.<member> · <path/in/cube/project.yml>[ · PARTIAL][ · VERIFY: <what>]
#   omni: <what Omni added or changed, and why>        ← only when PARTIAL
# omni-native: <why it exists>                          ← no Cube ancestor
```

Three tokens make the model auditable with `grep`:

| Question | Command |
|---|---|
| What came from Cube? | `grep -rn '# cube:'` |
| What is only half-ported? | `grep -rn 'PARTIAL'` |
| What still needs checking? | `grep -rn 'VERIFY:'` |
| What is ours, not Cube's? | `grep -rn '# omni-native:'` |

---

## Model / topic / view level: one header block

The header block carries what no individual field can: the sync's provenance,
and **the record of everything that was dropped**. A dropped member has no YAML
left to annotate, so its only trace is here.

Keep it scannable. Group drops by cause, give a count and the member names, and
**point at the reference doc instead of re-explaining the limitation**.

```yaml
# ── from Cube ─────────────────────────────────────────────────────────────
# source:  view revenue_overview · model/views/revenue_overview.yml
# project: github.com/acme/cube-models @ a1b2c3d (branch main)
# synced:  2026-09-28 by cube-to-omni
# ported:  3 cubes → 3 views · 2 joins → relationships · 1 view → this topic
#
# dropped — not representable in Omni (see LIMITATIONS.md):
#   pre_aggregations (2): conversions_rollup, conversions_daily
#   rolling_window (1):   conversions.revenue_28d
#
# partial — 3 fields, each marked PARTIAL at its definition:
#   margin_pct, created_at, avg_order_value
#
# verify after promotion:
#   access_policy.row_level → access_filters · user attributes: region, tier
#     user attributes:  https://docs.omni.co/administration/users/attributes.md
#     validate with:    https://docs.omni.co/visualize-present/dashboards/view-as.md
# ──────────────────────────────────────────────────────────────────────────
base_view: cube_conversions
```

### What belongs in the block, and what does not

| Include | Leave out |
|---|---|
| The upstream object, its **file path**, and the project + commit/branch | A restatement of why a limitation exists — link `LIMITATIONS.md` |
| Every **dropped** member, grouped by cause, with counts and names | Per-field detail that already lives at the field |
| A **name list** of the `PARTIAL` fields, so a reader knows the count without grepping | Prose explaining each partial — that is the field's own comment |
| Anything needing **post-promotion validation**, with doc links | Commentary on unchanged, cleanly-mapped fields |

The balance to hold: a reader must be able to see **what is missing** from the
header alone, and find **why** by following one link or grepping one token.
Every dropped member is named — a summary that says "2 pre-aggregations dropped"
without naming them hides exactly the gap someone needs to close.

---

## Check the header against the fields before you report

A header block that disagrees with the field markers is **worse than no header**
— it tells a reader the gap is smaller than it is. This happened while authoring
this scheme: a header claimed 2 `PARTIAL` fields where the file carried 6.

So derive the summary from the markers rather than writing it by hand, and
verify before reporting:

```bash
# the PARTIAL fields the file actually declares
grep -A3 'PARTIAL' <file> | grep -oE '^  [a-z_]+:' | tr -d ' :'

# counts (subtract 1 from the PARTIAL count for the header's own mention)
grep -c '# cube:'  <file>
grep -c 'PARTIAL'  <file>
grep -c 'VERIFY:'  <file>
```

Three invariants to hold on every file you write:

| Invariant | Why |
|---|---|
| The header's `partial —` list equals the fields actually marked `PARTIAL` | A short list hides work someone must finish |
| Every dropped member is named, never only counted | "2 pre-aggregations dropped" hides exactly which |
| Every field has **exactly one** lineage comment — no field with two, none with zero | Zero means unknown provenance; two means the definition is buried |

---

## User attributes: always annotate, always validate

Several Omni parameters are inert until a **user attribute** exists and is
assigned. A sync writes the parameter; it cannot create the attribute. So the
YAML must say which attributes it depends on, and the sync report must say they
need checking.

Annotate every one of these with the attribute names it reads:

| Parameter | Level |
|---|---|
| [`access_filters`](https://docs.omni.co/modeling/topics/parameters/access-filters.md) | topic |
| [`required_access_grants`](https://docs.omni.co/modeling/topics/parameters/required-access-grants.md) | topic · view · dimension · measure · composite topic |
| [`hidden_unless_access_grants`](https://docs.omni.co/modeling/topics/parameters/hidden-unless-access-grants.md) | topic · composite topic |
| [`mask_unless_access_grants`](https://docs.omni.co/modeling/dimensions/parameters/mask-unless-access-grants.md) | dimension |
| [`access_grants`](https://docs.omni.co/modeling/models/access-grants.md) | model |
| [`default_topic_access_filters`](https://docs.omni.co/modeling/models/default-topic-access-filters.md) | model |
| [`dynamic_shared_extensions`](https://docs.omni.co/modeling/models/dynamic-shared-extensions.md) | model |

```yaml
# cube: access_policy.row_level on `conversions` · model/cubes/conversions.yml
#   user attributes required: region, tier — must exist and be assigned in Omni
#   docs:     https://docs.omni.co/administration/users/attributes.md
#   VERIFY after promotion, per role, with View as:
#             https://docs.omni.co/visualize-present/dashboards/view-as.md
access_filters:
  region:
    field: cube_channel.region
    user_attribute: region
```

Two facts to carry into every such annotation:

- **A missing or unassigned attribute does not fail validation.** The model
  validates, the query runs, and the filter silently does not constrain. A
  broken access filter is the worst failure mode in this pipeline precisely
  because nothing announces it.
- **Access grants apply only to direct field access.** They do **not** propagate
  through a measure that references a granted or masked dimension. Test the
  measure, not just the dimension.

[View as](https://docs.omni.co/visualize-present/dashboards/view-as.md) exists
for this — it previews as another user specifically to verify attribute-based
grants and filters. Name it in the sync report so "validate after promotion"
becomes a step someone can actually perform.

---

## Sources

- [Omni view `template`](https://docs.omni.co/modeling/views/parameters/template.md) · [user attributes](https://docs.omni.co/administration/users/attributes.md) · [access grants](https://docs.omni.co/modeling/models/access-grants.md) · [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md)
- [LIMITATIONS.md](./LIMITATIONS.md) · [BRANCHING.md](./BRANCHING.md) · [PIPELINE.md](./PIPELINE.md)
