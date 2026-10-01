---
name: cube-omni-pipeline
description: "The orchestration and help guide for the bidirectional Cube ↔ Omni Analytics semantic pipeline — set up the integration, understand which Cube edition supports what, learn the ordered steps to make a change in Cube and have it appear in Omni (and back), choreograph branches on both platforms, and see exactly what can and cannot be mapped between the two semantic layers. Use this skill whenever someone asks how Cube and Omni fit together, wants the setup or prerequisites for the integration, asks which direction to sync or which platform should own a metric, needs the end-to-end pipeline of steps from development through deployment, asks what is lost in translation between Cube and Omni, or asks about Cube Cloud versus Cube Core differences. Triggers on \"how do I get my Cube model into Omni\", \"set up the Cube integration\", \"what's the workflow for Cube and Omni\", \"what can't be mapped between Cube and Omni\", \"which should own this metric\", \"Cube Core vs Cube Cloud\", \"show me the pipeline\", and \"how do I keep Cube and Omni in sync\"."
---

# Cube ↔ Omni — pipeline, setup, and orchestration

The hub for the bidirectional Cube ↔ Omni semantic integration. This skill does
not translate anything itself — it decides **which direction, which edition,
which branch**, and hands off.

```
                    ┌──────────────── the warehouse (one source) ─────────────────┐
                    │                                                              │
   ┌────────────────┴─────────────────┐                  ┌─────────────────────────┴───┐
   │            CUBE                  │   cube-to-omni   │            OMNI             │
   │  model/cubes/*.yml   (physical)  │  ─────────────▶  │  *.view          (physical) │
   │  model/views/*.yml   (curated)   │                  │  *.topic         (curated)  │
   │  joins on the left cube          │  ◀─────────────  │  relationships (one file)   │
   │  dev branch → build → production │   omni-to-cube   │  branch → validate → shared │
   └──────────────────────────────────┘                  └─────────────────────────────┘
```

Both platforms read the **same warehouse tables**. Neither queries the other.
The sync moves *definitions*; the data never moves.

## The skills

| Skill | Direction |
|---|---|
| [`cube-to-omni`](../cube-to-omni/SKILL.md) | Cube model → Omni views, topics, relationships |
| [`omni-to-cube`](../omni-to-cube/SKILL.md) | Omni model → Cube cubes, views, joins |
| **`cube-omni-pipeline`** (this one) | Setup, edition detection, branch choreography, the help guide |

## The references

| Read this | For |
|---|---|
| [SETUP.md](./references/SETUP.md) | **Start here.** Prerequisites, install, auth, the local Cube Core instance, required env vars |
| [EDITIONS.md](./references/EDITIONS.md) | Cube Cloud vs. Cube Core — the capability matrix and which commands exist |
| [PIPELINE.md](./references/PIPELINE.md) | The ordered steps, dev and deployed, both directions, plus a side-by-side command reference |
| [BRANCHING.md](./references/BRANCHING.md) | The branch policy, identifier cheat sheet, and the paired-branch workflow |
| [LIMITATIONS.md](./references/LIMITATIONS.md) | Everything that does not survive a sync, in both directions |
| [NOTATION.md](./references/NOTATION.md) | The lineage-comment scheme — marking what came from Cube, what was added, what was dropped, and which user attributes need validating |

---

## Prerequisites

```bash
command -v omni  >/dev/null || echo "Omni CLI missing: curl -fsSL https://raw.githubusercontent.com/exploreomni/cli/main/install.sh | sh"
command -v cube  >/dev/null || echo "Cube CLI missing: curl -fsSL https://raw.githubusercontent.com/cube-js/cube/master/install-cli.sh | sh"
omni whoami whoami
```

Full setup, including the Cube Core Docker instance and the credential
requirements, is in [SETUP.md](./references/SETUP.md).

## Discovering Commands

```bash
omni models --help && omni models yaml-get --schema
cube --help && cube data-model --help
```

---

## Known Issues & Safe Defaults

- **🛑 Branch by default on both platforms.** If the user named no branch: ask
  in an interactive session, create one in auto-mode and report it. Never merge,
  promote, deploy or "make live" unless the user asked **in this conversation**.
  [BRANCHING.md](./references/BRANCHING.md)
- **Detect the Cube edition before choosing any command.** The `cube` CLI and
  the `cube-agent-skills` plugin are **Cube Cloud clients**. Against Cube Core
  there are no deployments, no dev-mode branches, and no `cube meta` — only the
  YAML files and REST `/v1/meta`. [EDITIONS.md](./references/EDITIONS.md)
- **Never translate from `/v1/meta` alone — it carries no `sql`.** Verified.
  Read the authored YAML for logic, the compiled model for exposure.
- **A round trip is not idempotent.** Cube → Omni → Cube does not return the
  original files. Diff before believing a return sync is a no-op.
- **Pick one owner per object and write it down** in the model's `description`
  or `ai_context`. Two-way sync of the same measure with no recorded owner is a
  silent merge conflict; neither platform warns you.
- **Confirm both platforms read the same warehouse** before translating. An Omni
  model targets one connection; a Cube project can target several.
- **A compiling model is not a correct model.** Always run the parity check.
- **Every synced object carries its lineage in a comment**, and everything
  unmappable is **dropped and then named** in the parent object's header block.
  One comment per field, one block per model/topic/view. See
  [NOTATION.md](./references/NOTATION.md).
- **User-attribute-dependent parameters validate clean while doing nothing.**
  `access_filters`, `*_access_grants`, `mask_unless_access_grants`,
  `default_topic_access_filters` and `dynamic_shared_extensions` all need the
  [user attribute](https://docs.omni.co/administration/users/attributes.md) to
  exist *and* be assigned; a missing one **does not fail validation** — the
  model validates, the query runs, and the filter silently does not constrain.
  Always report the attribute names, link the docs in the YAML itself, and point
  at [View as](https://docs.omni.co/visualize-present/dashboards/view-as.md) for
  a per-role check. Going the other way, Cube resolves a security context per
  request rather than per-user attributes, so the policy needs re-testing there
  too.
- **Only semantics cross.** Dashboards, workbooks, reports, charts, schedules
  and content folders do not, in either direction.
- **`CUBEJS_DEV_MODE=true` is an authentication bypass**, not a convenience
  flag — local machine only. See [EDITIONS.md](./references/EDITIONS.md).

---

## Workflow — routing a request

### Step 1 — Establish the ground truth

```bash
# Omni: which model, which connection, can I branch?
omni models list
omni connections list
omni whoami whoami --model-id <modelId>       # QUERY_FULL_MODEL required

# Cube: which edition?
cube whoami >/dev/null 2>&1 && cube deployments list && echo "EDITION=cloud"
curl -fsS "${CUBE_CORE_URL:-http://localhost:4000}/readyz" >/dev/null 2>&1 && echo "EDITION=core"
```

If both Cube paths answer, the user has both — **ask which one this task
targets.** A Core project also imported into Cloud is common, and writing to the
wrong side silently diverges them.

### Step 2 — Confirm the two platforms share a warehouse

The sync is meaningless otherwise. Compare the Omni connection's
`host`/`database` against Cube's `data_source` configuration. State the match
(or the mismatch) explicitly before going further.

### Step 3 — Pick the direction

| The user wants | Direction | Skill |
|---|---|---|
| Cube metrics available in Omni for analysis, dashboards, AI | Cube → Omni | [`cube-to-omni`](../cube-to-omni/SKILL.md) |
| To build in Omni on top of Cube definitions | Cube → Omni, then `omni-model-builder` | [`cube-to-omni`](../cube-to-omni/SKILL.md) |
| Cube to become the source of truth for something modeled in Omni | Omni → Cube | [`omni-to-cube`](../omni-to-cube/SKILL.md) |
| Cube's pre-aggregations to accelerate Omni queries | **neither** — the SQL API path | [PIPELINE.md](./references/PIPELINE.md#alternate-architecture-omni-on-cubes-sql-api) |
| To leave Cube and consolidate on Omni | Cube → Omni, once, then stop syncing | [`cube-to-omni`](../cube-to-omni/SKILL.md) |
| "Keep them in sync" | **Push back first** — see Step 4 | — |

### Step 4 — When the user asks for continuous two-way sync

There is no continuous sync, and symmetrical two-way sync of the same object is
not a thing either platform supports. Do not pretend otherwise. Offer the
pattern that works — **split ownership, recorded in the model**:

| Object | Recommended owner | Why |
|---|---|---|
| Physical cubes, `sql_table`, joins | Cube | Cube's engine depends on them; Omni re-derives them cleanly |
| `pre_aggregations` | Cube | No faithful Omni equivalent |
| Topics, field curation | Omni | Richer curation; Cube views are a subset |
| AI metadata (`synonyms`, `sample_queries`, `ai_fields`) | Omni | Cube has no parameters for these at all |
| Business measures | **pick one per measure and record it** | The only genuinely ambiguous class |

Then a one-directional re-sync per change, with the sync point recorded so the
next run can diff instead of guess:

```bash
# Cube Cloud — server-side content hashes; cheap divergence check
cube data-model file-hashes <deployment> --branch <branch>

# Cube Core — git
git -C "$CUBE_PROJECT_DIR" diff --stat <last-synced-sha>..HEAD -- model/
```

### Step 5 — Set up the branch pair, then hand off

Create the branch on the **write** target and pin the **read** source to a named
branch so the translation is reproducible. Report the pair:

```
Paired branches for this sync
  Cube  (core)  : cube-omni/c2o-revenue-overview-20260922   commit a1b2c3d
  Omni          : cube-omni/c2o-revenue-overview-20260922
                  branchId 7c1e…  modelId 09ac…
```

Then invoke the directional skill. Do not translate here.

### Step 6 — Close the loop

After the directional skill reports back, confirm it covered all four:

1. **Validation** on the write side (Omni `validate`, or a Cube build).
2. **Exposure** — the member is actually queryable, not merely compiled.
3. **Parity** — the same aggregate on both platforms, side by side.
4. **The ⚠️/❌ list** — every lossy and unmapped item named explicitly.

Missing any of the four means the sync is not done, whatever the file writes say.

---

## The five questions users actually ask

**"I changed a measure in Cube — how does it get to Omni?"**
[PIPELINE.md → Pipeline A](./references/PIPELINE.md#pipeline-a--cube--omni). Read
the Cube branch, create an Omni branch, translate, validate, test-query, parity
check, report. Merging is your call, not the agent's.

**"It's already deployed in Cube. Same thing?"**
Yes, with two changes: read from Cube's `production` environment instead of a dev
branch, and after the Omni merge re-resolve against production with no
`--branch-id`.
[PIPELINE.md → A2](./references/PIPELINE.md#a2-deployed-a-change-already-live-in-cube-reaching-omnis-shared-model).

**"Can I go the other way?"**
Yes — [`omni-to-cube`](../omni-to-cube/SKILL.md) — but it loses more, chiefly
Omni's AI layer (`synonyms`, `sample_queries`, `ai_fields` have no Cube
parameters at all). [LIMITATIONS.md](./references/LIMITATIONS.md).

**"Which platform should own our metrics?"**
Step 4 above. The short answer: Cube owns physical structure and
pre-aggregations, Omni owns curation and the AI layer, and business measures
need an explicit per-measure decision recorded in the model.

**"Do I need Cube Cloud?"**
No for the semantics — data models port unchanged between editions. Yes for the
`cube` CLI, dev-mode branches, build status, workbooks and the MCP connector.
[EDITIONS.md](./references/EDITIONS.md).

---

## Resources

- [Cube docs](https://docs.cube.dev/docs/introduction) · [data modeling](https://docs.cube.dev/reference/data-modeling/cube.md) · [Cube Core](https://docs.cube.dev/cube-core/index) · [CLI](https://docs.cube.dev/reference/cli) · [REST API](https://docs.cube.dev/reference/core-data-apis/rest-api/reference.md) · [SQL API](https://docs.cube.dev/reference/core-data-apis/sql-api/index.md) · [MCP server](https://docs.cube.dev/docs/integrations/mcp-server.md)
- [`cube-agent-skills`](https://github.com/cube-js/cube-agent-skills) · [Cube connector & skills for Claude](https://cube.dev/blog/cube-connector-and-skills-for-claude)
- [Omni modeling](https://docs.omni.co/modeling/views.md) · [Model API](https://docs.omni.co/api/models.md) · [Branch mode](https://docs.omni.co/finding-content/drafting-publishing/branch-mode.md)
