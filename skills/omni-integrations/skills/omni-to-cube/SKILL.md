---
name: omni-to-cube
description: "Sync Omni Analytics semantic model logic back into Cube — take Omni views, topics, dimensions, measures and relationships and write them as Cube cubes, views and joins in YAML, on a Cube dev-mode branch (Cube Cloud) or a git branch (Cube Core), then build and verify. Use this skill whenever someone wants to push Omni metrics down into Cube, make Cube the source of truth for a measure modeled in Omni, export an Omni topic as a Cube view, hand Omni logic to Cube's semantic layer, or return a Cube model that was edited in Omni. Triggers on \"push this Omni measure to Cube\", \"sync Omni back to Cube\", \"export this topic to Cube\", \"make Cube the source of truth for revenue\", \"we modeled it in Omni, put it in Cube\", and \"send my Omni changes back to Cube\"."
---

# Omni → Cube

Reads an Omni semantic model and writes the equivalent Cube data model to a
**branch**. Omni is not modified by this skill.

This direction loses **more** than Cube → Omni — almost entirely because Omni's
AI-modeling and query-layer parameters have no Cube counterparts. Read
[LIMITATIONS.md](../cube-omni-pipeline/references/LIMITATIONS.md) before
promising a faithful export.

| Work | Delegate to |
|---|---|
| Reading the Omni model | [`omni-model-explorer`](../../../omni-model-explorer/SKILL.md) |
| Omni branches, YAML shapes, validation | [`omni-model-builder`](../../../omni-model-builder/SKILL.md) |
| Reading the target Cube project, impact analysis | [`cube-explore-model`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-explore-model/SKILL.md) (Cloud) |
| Writing Cube YAML on a dev branch | [`cube-build-model`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-build-model/SKILL.md) (Cloud) |
| Builds, deployments, environments | [`cube-deploy`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-deploy/SKILL.md) (Cloud) |
| Verifying the numbers in Cube | [`cube-run-query`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-run-query/SKILL.md) (Cloud) |

Read first:
- [FIELD-MAPPING.md](./references/FIELD-MAPPING.md) — parameter-by-parameter translation + a worked example
- [NOTATION.md](../cube-omni-pipeline/references/NOTATION.md) — **the lineage-comment scheme, applied in reverse: mark what came from Omni**
- [EDITIONS.md](../cube-omni-pipeline/references/EDITIONS.md) — **Cube Core has no `cube data-model` commands at all**
- [BRANCHING.md](../cube-omni-pipeline/references/BRANCHING.md) — the branch rules
- [LIMITATIONS.md](../cube-omni-pipeline/references/LIMITATIONS.md) — what will not cross

---

## Prerequisites

```bash
command -v omni >/dev/null || echo "ERROR: Omni CLI is not installed."
omni config show
omni config use <profile-name>
omni whoami whoami
```

> **Auth**: API key or OAuth. On **401**, ask the user to run
> `! omni config login <profile>`; never run the browser flow in a headless
> session. `omni whoami` needs **omni CLI ≥ 1.0.7** — probe capability rather
> than checking `--version`. See
> [**`omni-api-conventions`**](../../../../rules/omni-api-conventions.mdc).

```bash
# Cube Cloud
command -v cube >/dev/null || echo "Cube CLI not installed: curl -fsSL https://raw.githubusercontent.com/cube-js/cube/master/install-cli.sh | sh"
cube whoami || echo "Not authenticated. Interactive: cube login. Headless: set CUBE_API_URL + CUBE_API_KEY."
cube context list      # confirm the tenant BEFORE writing anything
cube deployments list
```

```bash
# Cube Core
curl -fsS "${CUBE_CORE_URL:-http://localhost:4000}/readyz"
git -C "$CUBE_PROJECT_DIR" status --porcelain    # must be clean before branching
```

## Discovering Commands

```bash
omni models yaml-get --schema
omni models get-topic --schema
omni query run --schema

cube data-model --help          # Cloud only
cube data-model put --help      # Cloud only
cube deployments build-status --help
```

---

## Known Issues & Safe Defaults

- **🛑 Always work on a Cube branch. If the user named no branch: ask in an
  interactive session; create one in auto-mode and report its name.** On Cube
  Cloud, **the API rejects file writes on any branch that is not a dev-mode
  branch** — the branch step is not ceremony, it is the difference between
  working and failing. Never merge or deploy unprompted. Full policy:
  [BRANCHING.md](../cube-omni-pipeline/references/BRANCHING.md).
- **`cube data-model dev-mode` picks the branch name, not you.** It forks a
  personal `dev-…` branch and **prints the name**. Capture it; writes target it.
- **`--branch` defaults to your active dev-mode branch, and that default is
  per-user, not per-command.** Pass it explicitly whenever more than one
  deployment is in play, or you will write to the wrong place and get a success
  response.
- **There is no offline validation on either Cube edition. Validation is a
  build.** Cloud: `cube deployments build-status`. Core: reload the container,
  then `/readyz` and `/v1/meta`. **Report build errors verbatim** — Cube names
  the file and the member, and a paraphrase loses both.
- **Read the target project before authoring.** Match its file layout, naming,
  whether measures live on cubes or views, and how joins are declared. A correct
  cube that looks nothing like its neighbours is a bad contribution.
- **Cube requires `sql` *and* `type` on every dimension.** Omni infers both from
  the column; Cube infers neither. Look the type up from Omni's schema layer or
  the warehouse — do not guess.
- **`case` replaces `sql` on a dimension.** Declaring both fails the build with
  `(dimensions.<name>.sql …) is not allowed`. Verified.
- **A dimension may not be named `day`, `week`, `month`, `quarter`, `year`,
  `hour`, `minute`, or `second`** — reserved granularity keywords. Rename
  (`month_dt` with `sql: month`).
- **Prefer `CASE WHEN … THEN 1 END` inside `sql` over a measure-level `filter`.**
  A Cube measure `filter` returns **NULL, not 0**, for groups with no matching
  rows when queried alongside other measures. Verified. Omni's `filters:` does
  not behave that way, so a literal translation changes the result on sparse
  groupings.
- **`prefix: true` renames members** to `<cube>_<member>`. Verified. Decide
  deliberately and check for collisions afterwards.
- **Segments and hierarchies do not propagate into a Cube view.** Verified.
  Anything the Omni side expects to be user-visible must be included explicitly.
- **`ai_context` goes on the view or the member, never the cube.** Cube ignores
  cube-level `ai_context`. Each value is capped at **2,000 characters and
  silently truncated** beyond that.
- **`synonyms`, `sample_queries`, `ai_fields`, `all_values`, `sample_values`
  have no Cube parameters.** Fold what fits into `meta.ai_context` prose and
  **report what was dropped** — this is the largest single loss in this
  direction.
- **`median`, `percentile` and any `number_agg` are Tesseract-only.** Confirm the
  target deployment runs Tesseract, or the cube will not build.
- **`*_distinct_on` aggregates do not translate.** They need a pre-deduplicated
  cube — a modeling change, not a translation. Stop and say so.
- **Omni's user-attribute-dependent parameters do not survive as-is.**
  `access_filters`, `required_access_grants`, `hidden_unless_access_grants` and
  `mask_unless_access_grants` all resolve against
  [Omni user attributes](https://docs.omni.co/administration/users/attributes.md);
  Cube resolves a **security context per request** instead. The shapes map
  (`access_policy.row_level` / `member_level`) but the plumbing does not — the
  attribute → security-context wiring is a separate task, not part of the
  translation. Before exporting, confirm the Omni side actually works with
  [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md), or
  you may faithfully port a policy that was already inert. After exporting,
  re-test on the Cube side with a scoped token, per role.
- **Omni grants apply only to direct field access.** They do **not** propagate
  through a measure that references a granted or masked dimension. When porting,
  check whether the Cube `access_policy` needs to cover the measure as well —
  otherwise a field that was protected in Omni is readable through its measure
  in Cube.
- **`always_where_sql` on an Omni topic becomes a predicate on the underlying
  cube, which changes every Cube view built on it.** That blast radius is larger
  than the Omni original. Flag it before writing.
- **If an Omni measure references a dimension with a model-layer `sql`
  override, resolve the override first.** A Cube measure `sql` reads physical
  columns; it does not see Omni's dimension overrides. Never inline silently —
  show the user the override and say what you did with it.
- **Never auto-translate `materialized_query` views into `pre_aggregations`.**
  Report them.
- **Composite topics do not translate.** Stop, do not approximate.
- **Templated filters half-translate.** A filter-only field that exists to offer
  a fixed set of choices maps to a `switch` dimension with `values`
  (Tesseract-only). A filter-only field used for **Mustache value injection**
  — `{{filters.v.f.value}}` spliced into another field's SQL, `bind_to`, a
  metric switcher — has **no** Cube analog. Split the two cases and report the
  second as dropped.
- **Every object you write carries its lineage in a comment**, following
  [NOTATION.md](../cube-omni-pipeline/references/NOTATION.md) with the tokens
  reversed — `# omni: <view>.<field> · <file>` for a ported member,
  `· PARTIAL` plus a `cube:` continuation when Cube needed additions the Omni
  source did not have (an explicit `type:`, a derived timeframe dimension,
  dialect-specific aggregate SQL), and `# cube-native:` for anything with no
  Omni ancestor. One comment per member; one header block per cube/view naming
  every **dropped** member.
- **Anything that cannot be even partially mapped is dropped, not
  approximated**, and recorded by name in the parent cube's or view's header
  block. Never emit a plain `sum` in place of a `sum_distinct_on` — that
  silently returns a wrong number.

---

## Workflow

### Step 1 — Gather requirements

Ask the user:

1. **What is moving?** A field list, a view, or a topic. (A topic → a Cube view
   is the cleanest unit.)
2. **Which Omni model and branch** to read from? (`omni models list`)
3. **Which Cube edition, deployment/project, and base branch?**
4. **Where should the members live** — on the cube (mirrors Omni's layout) or
   lifted to the view (mirrors Cube's convention)? Default to **whatever the
   target project already does**.
5. **New files, or editing existing ones?**
6. **Does Cube become the source of truth for these measures?** If yes, the Omni
   side should eventually stop defining them — that is a separate, explicit
   decision; record it, don't perform it.

> ⚠️ **STOP** — confirm all six before writing anything.

### Step 2 — Read the Omni source

```bash
omni models yaml-get <modelId> --mode combined                     # full composed model
omni models yaml-get <modelId> --file-name <view>.view             # one file
omni models yaml-get <modelId> --file-name <topic>.topic
omni models get-topic <modelId> --topic-name <topic>               # resolved joins
```

`--mode combined` gives the full composed model (schema + shared + branch);
`--mode extension` gives only a branch's changed files. Use `get-topic` to see
how joins actually resolve — the relationships file alone will mislead you.

### Step 3 — Read the target Cube project

```bash
# Cube Cloud — one request, then search locally
cube data-model list <deployment> --content --json > /tmp/cube-model.json
```

```bash
# Cube Core
find "$CUBE_PROJECT_DIR/model" -name '*.yml' | sort
```

Check **every name you are about to write** against what exists. A Cube member
name collides within its cube or view, and the failure shows up as a build
error, not a write error.

### Step 4 — Classify before translating

Produce the three buckets from
[FIELD-MAPPING.md](./references/FIELD-MAPPING.md) and show them to the user:

- **Maps cleanly** (✅)
- **Maps with loss** (⚠️ — name the loss: lost format precision, a timeframe
  that became a derived dimension, AI metadata folded into prose)
- **Does not map** (❌ — `synonyms`/`sample_queries`/`ai_fields`,
  `*_distinct_on`, composite topics, templated filters, `drill_queries`,
  `always_having_*`, `bin_boundaries`, `duration`, `dynamic_top_n`)

This list is the deliverable as much as the YAML is. An export that quietly
drops the AI layer looks successful and is not.

### Step 5 — Create the Cube branch

```bash
# Cube Cloud — captures the printed dev-… name
cube data-model branches <deployment>
cube data-model dev-mode <deployment> main
```

```bash
# Cube Core
git -C "$CUBE_PROJECT_DIR" checkout -b "cube-omni/o2c-<subject>-$(date +%Y%m%d)"
```

### Step 6 — Translate and write

Order matters — write cubes before the views that include them:

1. **Cubes** (one per Omni view): `sql_table`, dimensions (`sql` **and**
   `type`), measures, then `joins`.
2. **Views** (one per Omni topic): `cubes[].join_path`, `includes`/`excludes`,
   `folders`, `meta.ai_context`.
3. **Lineage comments**, written as you go, per
   [NOTATION.md](../cube-omni-pipeline/references/NOTATION.md) — one per member,
   one header block per file, with every dropped Omni member named. State in the
   block which Omni model and branch the content came from, so the Cube side can
   be traced back without the Omni UI.

```bash
# Cube Cloud
cube data-model put <deployment> model/cubes/order_items.yml --file ./order_items.yml --branch <dev-branch>
cube data-model put <deployment> model/views/revenue.yml --content - --branch <dev-branch>   # stdin
```

```bash
# Cube Core — ordinary file writes
cp ./order_items.yml "$CUBE_PROJECT_DIR/model/cubes/order_items.yml"
```

Quote the user's own definition into `description` when you write a `sql`
expression. If they said "revenue excludes refunds", that belongs in the SQL and
in the `description`.

### Step 7 — Commit and build (this *is* validation)

```bash
# Cube Cloud
cube data-model commit <deployment> -m "Sync <subject> from Omni" --branch <dev-branch>
cube deployments build-status <deployment> --branch <dev-branch>
```

```bash
# Cube Core
docker compose -f "$CUBE_PROJECT_DIR/docker-compose.yml" restart
curl -fsS "$CUBE_CORE_URL/readyz"
curl -sS -w '\nHTTP %{http_code}\n' "$CUBE_CORE_URL/cubejs-api/v1/meta" | head -c 800
```

A **500** from `/v1/meta` is a compile error and the body is the real message.
Read it; it names the file and the member.

### Step 8 — Confirm exposure, then query

Compiling is not the same as being queryable — a measure can build perfectly and
be exposed in no view.

```bash
# Cube Cloud
cube meta --selectors '[{"type":"cube","deploymentId":<id>,"environment":"<dev-branch>"}]'
```

```bash
# Cube Core
curl -fsS "$CUBE_CORE_URL/cubejs-api/v1/meta" \
  | jq -r '.cubes[] | "\(.type) \(.name): \([.measures[].name] | join(", "))"'

curl -fsS "$CUBE_CORE_URL/cubejs-api/v1/load" -H 'Content-Type: application/json' \
  --data '{"query":{"measures":["<view>.<measure>"],"dimensions":["<view>.<dimension>"],"limit":10}}'
```

### Step 9 — Parity check against Omni

Same aggregate, both platforms, same warehouse rows:

```bash
omni query run --body '{"query":{"modelId":"<modelId>","table":"<view>","fields":["<view>.<dim>","<view>.<measure>"],"limit":10,"join_paths_from_topic_name":"<topic>"}}'
```

Report the numbers side by side. Expected, legitimate differences: the
NULL-vs-0 filter behavior, a `median`/`percentile` implemented through
`number_agg`, and any timeframe you re-derived with dialect SQL. Everything else
is a defect — check primary keys on both sides of every join first.

### Step 10 — Report, then stop

Report the branch pair, the files written, the **verbatim** build result, the
exposure check, the parity numbers, and the full ⚠️/❌ list from Step 4.

Confirm before you report:

- every written member has exactly **one** lineage comment;
- every **dropped** Omni member is named in a parent header block;
- every `PARTIAL` member says what Cube needed that Omni did not supply;
- the header's `partial —` list matches the members actually marked `PARTIAL`
  (derive it with `grep`, don't write it from memory);
- anything user-attribute-dependent on the Omni side (`access_filters`,
  `required_access_grants`, `mask_unless_access_grants`) is called out as
  needing a re-test on the Cube side against its security context, since the
  enforcement model differs and a silently-inert policy is the worst outcome.

```bash
cube data-model exit-dev-mode <deployment>    # Cloud, when done inspecting
```

> 🛑 **Then stop.** Merging the PR, `cube deploy`, and any promotion to a
> production deployment are **separate, user-initiated steps**. And `cube deploy`
> and `cube github connect` do not mix — against one deployment whichever ran
> last wins, silently. Ask which the project uses before deploying anything.

---

## When something fails

| Symptom | Cause |
|---|---|
| Write rejected | Not on a dev-mode branch. Run `cube data-model dev-mode` and use the branch it prints. |
| `not logged in` | Rerun the preflight; do not retry the write. |
| `session expired — run cube login` | Refresh token is dead; the user must re-authenticate. |
| Build fails after commit | A real model error. Read `build-status` verbatim and fix the named file. |
| `"dimensions.<name>" does not match any of the allowed types` | Usually `case` together with `sql`, or a missing `type`. |
| `Cube <name> doesn't exist` | A join references a cube that failed to compile. Fix the upstream cube first — this error cascades. |
| Change builds but is not queryable | Not exposed in a view. Check `cube meta` / `/v1/meta`. |
| `type: number_agg` rejected | Deployment is not running Tesseract. |
| 403 on a deployment | The account lacks access to that deployment, not a bad id. |
| Empty file list from `data-model list` | Real — usually an unbuilt or newly created deployment. Say so rather than retrying. |
| Numbers differ from Omni on a fan-out join | Missing `primary_key: true` on one side. |

---

## Resources

- Omni: [modelParameters](../../../omni-model-builder/references/modelParameters.md) · [Model YAML API](https://docs.omni.co/api/models.md) · [Measures](https://docs.omni.co/modeling/measures.md) · [Topics](https://docs.omni.co/modeling/topics/parameters.md)
- Cube: [cube](https://docs.cube.dev/reference/data-modeling/cube.md) · [dimensions](https://docs.cube.dev/reference/data-modeling/dimensions.md) · [measures](https://docs.cube.dev/reference/data-modeling/measures.md) · [joins](https://docs.cube.dev/reference/data-modeling/joins.md) · [view](https://docs.cube.dev/reference/data-modeling/view.md) · [AI context](https://docs.cube.dev/docs/data-modeling/ai-context.md) · [CLI](https://docs.cube.dev/reference/cli) · [agent skills](https://github.com/cube-js/cube-agent-skills)
