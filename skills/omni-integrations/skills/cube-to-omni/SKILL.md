---
name: cube-to-omni
description: "Sync a Cube semantic model into Omni Analytics — read cubes, views, measures, dimensions and joins from Cube (Cube Cloud via the Cube CLI, or Cube Core via its model files and REST /v1/meta), translate them into Omni view, topic and relationship YAML, and write the result to an Omni branch. Use this skill whenever someone wants to bring Cube metrics into Omni, import a Cube view as an Omni topic, build in Omni on top of definitions that live in Cube, migrate off Cube, or keep Omni in sync with a changed Cube model. Triggers on \"import our Cube model into Omni\", \"sync Cube to Omni\", \"turn this Cube view into an Omni topic\", \"we define revenue in Cube, model it in Omni\", \"bring Cube measures into Omni\", and \"Cube changed, update Omni\"."
---

# Cube → Omni

Reads a Cube data model and writes the equivalent Omni semantic model to a
**branch**. Cube stays where it is; nothing is written to Cube by this skill.

This skill owns the **translation and the branch choreography**. It delegates
the platform work to the two native skill sets:

| Work | Delegate to |
|---|---|
| Reading the Cube model, impact analysis | [`cube-explore-model`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-explore-model/SKILL.md) (Cloud) |
| Querying Cube for the parity check | [`cube-run-query`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-run-query/SKILL.md) (Cloud) |
| Writing Omni YAML, branches, validation | [`omni-model-builder`](../../../omni-model-builder/SKILL.md) |
| Inspecting the existing Omni model | [`omni-model-explorer`](../../../omni-model-explorer/SKILL.md) |
| Making the result AI-ready in Omni | [`omni-ai-optimizer`](../../../omni-ai-optimizer/SKILL.md) |
| Building dashboards on the result | [`omni-content-builder`](../../../omni-content-builder/SKILL.md) |

> **Validated end to end** against omni CLI 1.3.1 and a live Cube Core
> instance (`cubejs/cube:latest`) while authoring: a Cube model was translated
> to Omni views + `relationships` + a topic, written to a branch, validated
> clean, and test-queried against Snowflake with correct join resolution.
> **Both platforms were then pointed at the same Snowflake tables and the same
> aggregate compared row for row — 32/32 values agreed numerically** across 8
> groups x 4 measures (sum, filtered sum, count distinct, count) over 848,768
> fact rows. The
> Cube Cloud-only surfaces (`cube data-model`, dev-mode branches, `cube meta`,
> build status) are documented from Cube's published docs and **not** validated
> here — see [EDITIONS.md](../cube-omni-pipeline/references/EDITIONS.md).

Read first:
- [FIELD-MAPPING.md](./references/FIELD-MAPPING.md) — parameter-by-parameter translation + a worked example
- [NOTATION.md](../cube-omni-pipeline/references/NOTATION.md) — **the lineage-comment scheme; every object you write carries one**
- [EDITIONS.md](../cube-omni-pipeline/references/EDITIONS.md) — Cloud vs. Core; **which commands even exist**
- [BRANCHING.md](../cube-omni-pipeline/references/BRANCHING.md) — the branch rules
- [LIMITATIONS.md](../cube-omni-pipeline/references/LIMITATIONS.md) — what will not cross

---

## Prerequisites

```bash
# Omni CLI — if missing, ask the user to install it
# See: https://github.com/exploreomni/cli#readme
command -v omni >/dev/null || echo "ERROR: Omni CLI is not installed."
omni config show
omni config use <profile-name>
omni whoami whoami
```

> **Auth**: a profile authenticates with an **API key** or **OAuth**. If
> `whoami` (or any call) returns **401**, hand off — ask the user to run
> `! omni config login <profile>` (OAuth 2.1 browser flow; blocks ~2 min on the
> browser). Don't run `config login` yourself in a headless/CI session. See the
> [**`omni-api-conventions`**](../../../../rules/omni-api-conventions.mdc) rule
> for profile setup and discovering command shapes with `--schema`.
>
> `omni whoami` requires **omni CLI ≥ 1.0.7**. Probe capability, don't gate on
> `--version`: an `unknown command "whoami"` error means the binary is too old —
> ask the user to update it.

Then, **per edition** — see [EDITIONS.md](../cube-omni-pipeline/references/EDITIONS.md):

```bash
# Cube Cloud
command -v cube >/dev/null || echo "Cube CLI not installed: curl -fsSL https://raw.githubusercontent.com/cube-js/cube/master/install-cli.sh | sh"
cube whoami || echo "Not authenticated. Interactive: cube login. Headless: set CUBE_API_URL + CUBE_API_KEY."
cube context list      # confirm the tenant before reading anything
cube deployments list
```

```bash
# Cube Core
curl -fsS "${CUBE_CORE_URL:-http://localhost:4000}/readyz"
ls "$CUBE_PROJECT_DIR"/model/cubes "$CUBE_PROJECT_DIR"/model/views
```

## Discovering Commands

```bash
omni models --help
omni models yaml-get --schema
omni models yaml-create --schema
omni models create-branch --schema
omni query run --schema

cube data-model --help          # Cloud only
cube meta --help                # Cloud only
```

Use `-o json` for structured Omni output, `-o human` for tables. Cube list
commands print tables; add `--json`. Cube `get` commands always print JSON.

---

## Known Issues & Safe Defaults

- **🛑 Always work on an Omni branch. If the user named no branch: ask in an
  interactive session; create one in auto-mode and report its name and id.**
  Never write semantics to the shared model. Never merge unprompted. Full
  policy: [BRANCHING.md](../cube-omni-pipeline/references/BRANCHING.md).
- **`/v1/meta` and `cube meta` carry NO `sql`.** Verified: a compiled-model
  payload contains zero `sql` expressions. A translation driven from metadata
  alone silently drops every measure and dimension expression. **Always read the
  authored YAML for logic** and use the compiled model only for exposure.
- **`public: false` members are absent from the compiled model** but present in
  the files — and still translatable. Another reason to read the files.
- **Field-reference syntax differs and never copies verbatim.** Cube YAML uses
  `{field}` and `{CUBE}.column`; Omni uses `${field}` / `${view.field}`.
  **There is no `${TABLE}` in Omni** — it fails validation with
  `Column "__omni_scoped" not found`. A plain column dimension in Omni needs no
  `sql:` at all.
- **A Cube measure-level `filter` returns NULL — not 0 — for groups with no
  matching rows** when queried alongside other measures. Verified. Omni's
  `filters:` does not behave that way, so your parity check will legitimately
  disagree on sparse groups. Say so rather than "fixing" it.
- **`prefix: true` on a view's cube renames members** to `<cube>_<member>`
  (`orders_status`, `users_full_name`). Verified. Translate the **prefixed**
  names and check for collisions *after* translation.
- **Segments and hierarchies do not propagate into a Cube view.** Verified: a
  cube with a segment and a hierarchy exposes neither through a view that
  includes its other members. If the user expects them in Omni, they must be
  handled explicitly — see FIELD-MAPPING.
- **`primary_key: true` is not optional in Omni.** A missing primary key is the
  single most common cause of wrong numbers after a sync, because join row
  multiplication stops being deduplicated. Confirm one on **both** sides of
  every join before trusting a number.
- **Never auto-translate `pre_aggregations`.** Report them and let the user
  decide. Same for JavaScript/Jinja dynamic models — translate the compiled
  output and say that is what you did.
- **`count_distinct_approx` → `count_distinct` changes the numbers.** HyperLogLog
  is approximate and additive; exact distinct is neither. Never translate it
  silently.
- **Dimension `case` replaces `sql`.** Declaring both fails the Cube build with
  `(dimensions.<name>.sql …) is not allowed`. Relevant when you round-trip.
- **One Omni model targets one connection.** A multi-`data_source` Cube project
  needs multiple Omni models or warehouse-side federation.
- **Every object you write carries its lineage in a comment** — one per field,
  one header block per view/topic/model, following
  [NOTATION.md](../cube-omni-pipeline/references/NOTATION.md). A reader must be
  able to tell what came from Cube, what Omni added, and what was dropped,
  without access to the Cube project. Comments survive Omni's canonicalization
  (verified) and field-level ones stay attached to their key.
- **Anything that cannot be even partially mapped is dropped from the YAML, not
  approximated** — and then recorded by name in the header block of the parent
  model, topic or view. A dropped member has no YAML left to annotate, so the
  parent block is its only trace. Never leave a dropped member unrecorded, and
  never name a count without naming the members.
- **User-attribute-dependent parameters need an annotation and a post-promotion
  check.** `access_filters`, `required_access_grants`,
  `hidden_unless_access_grants`, `mask_unless_access_grants`, `access_grants`,
  `default_topic_access_filters` and `dynamic_shared_extensions` are all inert
  until the attribute exists **and is assigned** — and a missing attribute
  **does not fail validation**: the model validates, the query runs, and the
  filter silently does not constrain. Annotate the attribute names, link
  [user attributes](https://docs.omni.co/administration/users/attributes.md),
  and tell the user to verify per role with
  [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md).
  Note also that **access grants do not propagate through a measure** that
  references a granted or masked dimension — test the measure too.
- **Both platforms must read the same warehouse tables** for the sync to mean
  anything. Confirm the Omni connection and Cube's `data_source` point at the
  same place *before* translating, not after.

---

## Workflow

### Step 1 — Gather requirements

Ask the user:

1. **Which Cube object(s)?** A view (preferred — it maps to an Omni topic), a
   cube, or the whole project.
2. **Which Cube edition, deployment and branch?** (`cube deployments list` /
   `cube data-model branches`, or the Core project directory + git branch.)
3. **Which Omni model?** (`omni models list`) — or does one need creating? It
   must sit on a connection pointing at the **same warehouse** Cube reads
   (`omni connections list`).
4. **Which Omni branch?** If they don't name one, see the branch policy above.
5. **New or updating?** If an Omni topic/view of that name already exists, does
   this overwrite it, extend it, or land beside it?

> ⚠️ **STOP** — confirm all five before writing anything.

### Step 2 — Read the Cube model (both layers)

Logic from the files, exposure from the compiled model. Never one without the other.

```bash
# Cube Cloud — one request, then search locally. Do NOT loop `get` over paths.
cube data-model list <deployment> --content --json --branch <branch> > /tmp/cube-model.json
cube meta --selectors '[{"type":"cube","deploymentId":<id>,"environment":"<branch>"}]' > /tmp/cube-meta.json
```

```bash
# Cube Core
find "$CUBE_PROJECT_DIR/model" -name '*.yml' -o -name '*.yaml' | sort
curl -fsS "$CUBE_CORE_URL/cubejs-api/v1/meta" > /tmp/cube-meta.json
```

Then inventory what you found, and classify it **before** translating:

```bash
jq -r '.cubes[] | "\(.type)\t\(.name)\tmeasures=\(.measures|length)\tdims=\(.dimensions|length)\tsegments=\(.segments|length)\thierarchies=\(.hierarchies|length)\tfolders=\(.folders|length)"' /tmp/cube-meta.json
```

Produce a three-bucket list and show it to the user:

- **Maps cleanly** (✅ rows in FIELD-MAPPING)
- **Maps with loss** (⚠️ rows — name the loss for each)
- **Does not map** (❌ rows — `pre_aggregations`, `mask`, `rolling_window`,
  `time_shift`, dynamic generators, multi-`data_source`)

### Step 3 — Confirm the Omni target

```bash
omni models list
omni connections list
omni whoami whoami --model-id <modelId>     # QUERY_FULL_MODEL required to branch
```

No `QUERY_FULL_MODEL` → you cannot branch this model. Report it and stop; do not
degrade to an unbranched write.

If the model needs creating, use [`omni-admin`](../../../omni-admin/SKILL.md)
(connection/model provisioning) and refresh its schema layer so the base tables
exist before you write views:

```bash
omni models refresh <modelId>
```

### Step 4 — Create the Omni branch

```bash
omni models create-branch <modelId> --name "cube-omni/c2o-<subject>-$(date +%Y%m%d)"
# The response `model.id` is your branchId (a UUID). Capture it.
```

Record the branch pair as shown in
[BRANCHING.md](../cube-omni-pipeline/references/BRANCHING.md#branch-naming-convention).

### Step 5 — Translate

Work through [FIELD-MAPPING.md](./references/FIELD-MAPPING.md) in this order —
Omni resolves references at validation time, so dependencies must exist first:

1. **Views** (one per Cube cube) — dimensions, then measures.
2. **Relationships** (one per Cube join), resolving `{CUBE}` to the declaring
   view's name explicitly.
3. **Topics** (one per Cube view) — `base_view`, `joins:`, `fields:`.
4. **Lineage comments** — as you write each object, attach its comment per
   [NOTATION.md](../cube-omni-pipeline/references/NOTATION.md): one line per
   field (`# cube: <cube>.<member> · <path>`, plus `· PARTIAL` and an `omni:`
   continuation when only some sub-parameters came from Cube, or
   `# omni-native:` when nothing did), and one header block per view/topic
   recording the source, the project commit, the **named** dropped members, the
   list of `PARTIAL` fields, and anything needing post-promotion validation.
   Do this *while* translating — reconstructing provenance afterwards does not
   work.
5. **AI metadata** — carry `meta.ai_context` → `ai_context`, and `description`
   → `description`. Then consider
   [`omni-ai-optimizer`](../../../omni-ai-optimizer/SKILL.md) for the
   `synonyms` / `sample_queries` / `ai_fields` Cube has no way to express —
   these are a **gain** on the way into Omni.

Quote the user's own words into `description` when they explained a metric. If
they said "revenue excludes refunds", that belongs in the SQL *and* the
description — not only in the chat.

### Step 6 — Write to the branch

```bash
# modelId is a POSITIONAL argument, not a body field. The content key is `yaml`.
# --body also accepts @path/to/file.json, which avoids shell-escaping whole files.
omni models yaml-create <modelId> --body '{
  "branchId": "<branchId>",
  "fileName": "revenue_overview.topic",
  "yaml": "<full file content>"
}'
```

> ⚠️ **Three shapes that all fail with a confusing error.** Verified against
> omni CLI 1.3.1:
> - `modelId` goes in the **path**, not the body — putting it in the body only
>   gives `Error: accepts 1 arg(s), received 0`.
> - The content key is **`yaml`**, not `content` — otherwise
>   `Bad Request: yaml: Invalid input: expected string, received undefined,
>   Unrecognized key: "content"`.
> - `fileName` must be `model`, `relationships`, or end in `.topic`,
>   `.composite_topic` or `.view`. **There is no `.relationship` extension** —
>   relationships live in a single file named exactly `relationships`, and its
>   YAML is a **bare top-level list** with no `relationships:` key. Getting that
>   wrong returns `saveError: "Property must be a list"`.
>
> ⚠️ **`yaml-create` replaces a file's entire authored content** — there is no
> field-by-field merge. When editing an existing file, `yaml-get` it first, edit,
> and write the **complete** file back, or the other authored fields are dropped.
> `fileName` is the file's **exact path** including any folder prefix; a
> non-matching name does not error — it **silently creates a duplicate** and
> returns `success: true`.

### Step 7 — Validate and test-query

```bash
omni models validate <modelId> --branch-id <branchId>

omni query run --body '{"query":{
  "modelId":"<modelId>",
  "table":"<base_view>",
  "fields":["<view>.<dimension>","<view>.<measure>"],
  "sorts":[{"column_name":"<view>.<measure>","descending":true}],
  "limit":10,
  "join_paths_from_topic_name":"<topic>"
},"branchId":"<branchId>"}'
```

> ⚠️ **A sort entry's key is `column_name`, not `field`.** Using `field` fails
> with `Instantiation of … Sort value failed … missing … columnName`.
>
> ⚠️ **The job result is base64-encoded Apache Arrow IPC**, not JSON rows —
> `summary.display_sql` is the readable proof the translation resolved
> correctly, and decoding `result` needs an Arrow reader. Read the SQL first:
> it shows the join path and the exact aggregate that was generated.

Validation passing is not enough — a topic can validate and still fail to
resolve a join at query time. Query it.

### Step 8 — Parity check

Run the **same aggregate on both platforms** and compare. This is the step that
makes the sync trustworthy; do not skip it.

```bash
# Cube Core
curl -fsS "$CUBE_CORE_URL/cubejs-api/v1/load" -H 'Content-Type: application/json' \
  --data '{"query":{"measures":["<view>.<measure>"],"dimensions":["<view>.<dimension>"]}}'
```

For Cube Cloud, hand off to
[`cube-run-query`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-run-query/SKILL.md).

Report both numbers side by side. A mismatch is a finding — check, in this
order: primary keys on both sides of every join (fan-out), a measure filter that
did not survive, the NULL-vs-0 filter behavior above, timezone/fiscal handling,
then `count_distinct_approx`.

### Step 9 — Report, then stop

Report:

- **Branch pair** — Omni branch name + `branchId` + `modelId`; Cube deployment/branch or commit SHA read from.
- **Written** — each file and what it contains.
- **Validation** — `omni models validate` result, and the test query's output.
- **Parity** — the side-by-side numbers.
- **Mapped with loss** — every ⚠️, each with its specific consequence.
- **Not mapped** — every ❌, explicitly. Never let an unmapped `mask`,
  `access_policy` or `pre_aggregation` go unmentioned.
- **Gained** — Omni features now available that Cube could not express
  (extra timeframes, `synonyms`, `sample_queries`), flagged as *will not survive
  a return sync*.
- **User attributes to create or assign** — every attribute name the written
  YAML depends on, with
  [the docs link](https://docs.omni.co/administration/users/attributes.md) and
  the instruction to verify per role with
  [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md).
  Say plainly that a missing attribute **does not fail validation** — the model
  validates and the filter silently does not constrain.

Confirm before you report:

- every written field has exactly **one** lineage comment;
- every **dropped** Cube member is named in a parent header block, grouped by
  cause — a count without names hides the gap someone has to close;
- every `PARTIAL` field names what Omni added and why;
- the header block alone is enough to see *what is missing*, with `LIMITATIONS.md`
  one link away for *why*;
- the header's `partial —` list **matches** the fields actually marked
  `PARTIAL`. Derive it, don't write it from memory — a header that undercounts
  hides work someone has to finish:
  ```bash
  grep -A3 'PARTIAL' <file> | grep -oE '^  [a-z_]+:' | tr -d ' :'
  ```

> 🛑 **Then stop.** `omni models merge-branch`, `omni models commit`, and any
> promotion are **separate, user-initiated steps**. Preparing the branch is the
> whole job for a "sync this" request. Ask only when the user has asked to ship,
> and see [BRANCHING.md](../cube-omni-pipeline/references/BRANCHING.md#step-5--shipping-when-the-user-asks-for-it).

### Step 10 — Record the sync point

So the next sync can diff instead of guess, write what you synced from into the
Omni topic's `description` or `ai_context`:

```
Source of truth: Cube view revenue_overview (model/views/revenue_overview.yml)
Synced from commit a1b2c3d on 2026-09-22
```

---

## When something fails

| Symptom | Cause |
|---|---|
| `Column "__omni_scoped" not found` | A `${TABLE}` reference, or a `sql:` on a plain column dimension. Remove the `sql:` and let it auto-map. |
| Omni validate: field not found | Translated in the wrong order — write views before relationships before topics. |
| Omni validate passes, query fails on a join | The relationship resolves differently inside the topic. Inspect the topic's resolved `join_via_map` with `omni models get-topic`. |
| `400 Unrecognized key: "branchName"` | You passed a branch **name** where Omni wants the server-issued `branchId` UUID. |
| `Error: accepts 1 arg(s), received 0` | `modelId` belongs in the **path** (positional), not the request body. |
| `Unrecognized key: "content"` | The `yaml-create` content key is **`yaml`**, not `content`. |
| `File name must be either 'model', 'relationships', or end with '.topic', '.composite_topic', or '.view'` | There is no `.relationship` extension — the file is named exactly `relationships`. |
| `saveError: "Property must be a list"` | The `relationships` file is a **bare top-level list**; drop the `relationships:` key. |
| `Instantiation of … Sort value failed … columnName` | A sort entry's key is `column_name`, not `field`. |
| A duplicate file appeared in the Omni model | `fileName` did not match the existing file's exact path. `yaml-create` silently creates rather than erroring. |
| Numbers inflated on a `one_to_many` path | Missing `primary_key: true`. Check both sides. |
| Cube filtered measure is NULL where Omni's is 0 | Expected — see Known Issues. Not a bug to fix. |
| Cube `/v1/meta` returns HTTP 500 | A Cube build error. Read it verbatim; it names the file and the member. Fix on the Cube side first. |
| A Cube member exists in the files but not in `/v1/meta` | `public: false`, or it is not exposed in any view. Both are translatable — the files are the source. |

---

## Resources

- Cube: [data modeling reference](https://docs.cube.dev/reference/data-modeling/cube.md) · [measures](https://docs.cube.dev/reference/data-modeling/measures.md) · [dimensions](https://docs.cube.dev/reference/data-modeling/dimensions.md) · [joins](https://docs.cube.dev/reference/data-modeling/joins.md) · [views](https://docs.cube.dev/reference/data-modeling/view.md) · [REST API](https://docs.cube.dev/reference/core-data-apis/rest-api/reference.md) · [CLI](https://docs.cube.dev/reference/cli) · [agent skills](https://github.com/cube-js/cube-agent-skills)
- Omni: [Model YAML API](https://docs.omni.co/api/models.md) · [Views](https://docs.omni.co/modeling/views.md) · [Topics](https://docs.omni.co/modeling/topics/parameters.md) · [Relationships](https://docs.omni.co/modeling/relationships.md) · [Branch mode](https://docs.omni.co/finding-content/drafting-publishing/branch-mode.md)
