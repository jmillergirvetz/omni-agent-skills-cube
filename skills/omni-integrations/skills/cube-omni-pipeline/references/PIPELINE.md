# The pipeline — how a change travels between Cube and Omni

This is the help guide: the ordered steps to make a change in one platform and
have it exist in the other, **both while developing and once deployed**.

Read [EDITIONS.md](./EDITIONS.md) first to know which edition path applies, and
[BRANCHING.md](./BRANCHING.md) for the branch rules that govern every step here.

---

## The shape of it

```
                    ┌──────────────── the warehouse (one Snowflake) ───────────────┐
                    │                                                              │
   ┌────────────────┴─────────────────┐                  ┌─────────────────────────┴───┐
   │            CUBE                  │                  │            OMNI             │
   │                                  │                  │                             │
   │  model/cubes/*.yml   (physical)  │  ──  c2o  ──▶    │  *.view          (physical) │
   │  model/views/*.yml   (curated)   │                  │  *.topic         (curated)  │
   │  joins: on the left cube         │  ◀──  o2c  ──    │  relationships (one file)   │
   │                                  │                  │                             │
   │  dev branch → build → production │                  │  branch → validate → shared │
   └──────────────────────────────────┘                  └─────────────────────────────┘
```

Both platforms read the **same warehouse tables**. Neither queries the other.
The sync moves *definitions*; the data never moves. That is what makes the
parity check in [BRANCHING.md](./BRANCHING.md#step-3--cross-check-the-numbers)
meaningful — the same rows, aggregated two ways, must agree.

---

## Pipeline A — Cube → Omni

> Use the [`cube-to-omni`](../../cube-to-omni/SKILL.md) skill.

### A1. Development (a change on a Cube dev branch reaching an Omni branch)

| # | Step | Cube Cloud | Cube Core |
|---|---|---|---|
| 1 | Preflight both platforms | `cube whoami` · `cube context list` · `cube deployments list` | `curl $CUBE_CORE_URL/readyz` · `git status` |
| 2 | Identify the branch to read **from** | `cube data-model branches <dep>` — pin one with `--branch` | `git checkout <branch>` |
| 3 | Read the **authored YAML** (logic lives here) | `cube data-model list <dep> --content --json --branch <b>` | read `model/cubes/*.yml`, `model/views/*.yml` |
| 4 | Read the **compiled model** (exposure lives here) | `cube meta --selectors '[{"type":"cube","deploymentId":<id>,"environment":"<b>"}]'` | `curl $CUBE_CORE_URL/cubejs-api/v1/meta` |
| 5 | Confirm the Omni target model + connection | `omni models list` · `omni connections list` — the connection must point at the **same warehouse** Cube reads | ← same |
| 6 | **Create the Omni branch** | `omni models create-branch <modelId> --name cube-omni/c2o-<subject>-<date>` → capture `model.id` as `branchId` | ← same |
| 7 | Translate | [cube-to-omni FIELD-MAPPING](../../cube-to-omni/references/FIELD-MAPPING.md) | ← same |
| 8 | Write to the Omni branch | `omni models yaml-create` with `branchId`, one **whole file** per call | ← same |
| 9 | Validate | `omni models validate <modelId> --branch-id <branchId>` | ← same |
| 10 | Test-query the branch | `omni query run --body '{…,"branchId":"<branchId>"}'` | ← same |
| 11 | **Parity check** — same aggregate, both platforms | `cube meta` + a Cube query vs. the Omni query | `/v1/load` vs. the Omni query |
| 12 | Report: branch pair, what mapped, what did not, parity result | — | — |
| 13 | 🛑 **Stop.** Merging is the user's call. | — | — |

### A2. Deployed (a change already live in Cube reaching Omni's shared model)

The steps are identical except for what you read from and write to:

- **Step 2** reads from Cube's **production** environment, not a dev branch:
  `cube meta --selectors '[{…,"environment":"production"}]'`, or for Core, the
  default git branch with the container running that commit.
- **Step 13** becomes, *when the user asks for it*:
  - git-connected Omni model → `omni models commit` → review the PR → merge in
    your git host → changes reach `baseBranch` on the next sync.
  - non-git Omni model → `omni models merge-branch <modelId> <branchName>`.
- **After merge**, re-resolve against production with **no** `--branch-id`:
  `omni models yaml-get <modelId> --file-name <file>` and one more test query.
  Net-new topics and views especially — a branch that validated can still fail
  to resolve in the shared model.

### A3. Keeping it in sync afterwards

There is no continuous sync. Re-run Pipeline A when the Cube model changes. To
know **whether** it changed without pulling everything:

```bash
# Cube Cloud — server-side content hashes, cheap
cube data-model file-hashes <dep> --branch <branch>
```

```bash
# Cube Core — git
git -C "$CUBE_PROJECT_DIR" diff --stat <last-synced-sha>..HEAD -- model/
```

Record the Cube commit SHA (Core) or file-hash set (Cloud) you last synced from
in the Omni topic's `description` or `ai_context`. It is the only durable link
between the two models, and it turns "is Omni current?" from a guess into a
diff.

---

## Pipeline B — Omni → Cube

> Use the [`omni-to-cube`](../../omni-to-cube/SKILL.md) skill.

### B1. Development (a change on an Omni branch reaching a Cube dev branch)

| # | Step | Cube Cloud | Cube Core |
|---|---|---|---|
| 1 | Preflight both platforms | as above | as above |
| 2 | Confirm you can branch in Omni | `omni whoami whoami --model-id <modelId>` → needs `QUERY_FULL_MODEL` | ← same |
| 3 | Read the Omni source | `omni models yaml-get <modelId> [--branch-id <id>] --mode combined` | ← same |
| 4 | **Read the target Cube project first** — match its conventions | `cube data-model list <dep> --content --json` | read `model/` |
| 5 | **Create the Cube branch** | `cube data-model dev-mode <dep> main` → **capture the printed `dev-…` name** | `git checkout -b cube-omni/o2c-<subject>-<date>` |
| 6 | Translate | [omni-to-cube FIELD-MAPPING](../../omni-to-cube/references/FIELD-MAPPING.md) | ← same |
| 7 | Write | `cube data-model put <dep> model/cubes/x.yml --file ./x.yml --branch <b>` | write the file, then `git add` |
| 8 | Commit (Cloud: required before a build) | `cube data-model commit <dep> -m "…" --branch <b>` | `git commit` |
| 9 | **Validate = build.** There is no linter. | `cube deployments build-status <dep> --branch <b>` | restart the container, then `curl $CUBE_CORE_URL/readyz` |
| 10 | Confirm exposure — compiling ≠ queryable | `cube meta --selectors '[{…,"environment":"<b>"}]'` | `curl $CUBE_CORE_URL/cubejs-api/v1/meta` |
| 11 | Test-query | hand off to `cube-run-query` | `POST $CUBE_CORE_URL/cubejs-api/v1/load` |
| 12 | **Parity check** against Omni | — | — |
| 13 | Report; then 🛑 **stop** | `cube data-model exit-dev-mode <dep>` when done inspecting | — |

> ⚠️ **A failing build is the real error message.** Cube names the file and the
> member. Report it **verbatim** — paraphrasing loses both. Do not guess at the
> cause before reading it.

### B2. Deployed (reaching Cube's production deployment)

*Only when the user explicitly asks to ship.* Which route depends on how the
deployment takes code, and the two do not mix:

| The project uses | Ship with | Caveat |
|---|---|---|
| Git as source of truth (`cube github connect`) | merge the PR in your git host; Cube builds from git | Do **not** also run `cube deploy` — whichever ran last wins, silently |
| Local-directory deploys | `cube deploy <dep>` | Replaces the project from local files |
| Cube Core | `git push` + merge, then redeploy the container on the new commit | `docker compose up -d --force-recreate` |

Then, always:

```bash
cube deployments build-status <dep>      # a successful deploy ≠ a successful build
```

A deploy command returning successfully means the **upload** succeeded. Never
tell anyone it shipped before build status is green.

---

## Pipeline C — the full round trip

Only do this deliberately, and only with the rule from
[LIMITATIONS.md](./LIMITATIONS.md) in front of you: **a round trip is not
idempotent.**

```
Cube (source of truth for physical cubes + pre-aggregations)
  └─ A ─▶ Omni (source of truth for topics, AI metadata, curation)
            └─ B ─▶ Cube   ← only for measures Omni owns
```

The workable pattern is **split ownership, written down**:

| Object | Owner | Why |
|---|---|---|
| Physical cubes / `sql_table` / joins | Cube | Cube's engine depends on them; Omni re-derives cleanly |
| `pre_aggregations` | Cube | No faithful Omni equivalent |
| Topics / field curation | Omni | Richer curation; Cube views are a subset |
| AI metadata (`synonyms`, `sample_queries`, `ai_fields`) | Omni | No Cube parameters at all |
| Business measures | **pick one per measure and record it** | This is the only genuinely ambiguous class |

Record ownership in the model itself — an `ai_context` or `description` line
such as `Source of truth: Cube (model/views/revenue_overview.yml)` — so the next
sync does not have to guess. Two-way sync of the same measure with no recorded
owner is a silent merge conflict, and neither platform will warn you.

---

## Alternate architecture — Omni on Cube's SQL API

Everything above translates semantics so Omni queries the warehouse directly.
The alternative is to let **Cube stay the query engine**: Cube exposes a
Postgres-wire [SQL API](https://docs.cube.dev/reference/core-data-apis/sql-api/index.md),
and Omni connects to it as a Postgres connection.

| | Translation (the default above) | Omni on Cube's SQL API |
|---|---|---|
| Where metrics execute | the warehouse, via Omni | Cube |
| Single source of truth | split, recorded per object | Cube, genuinely |
| Omni features available | **all** — topics, AI, aggregate awareness, branching | reduced; Omni sees flattened tables |
| Cube pre-aggregation acceleration | not used by Omni | **used** — a real advantage |
| Drift risk | real; needs re-syncing | none |
| Setup cost | per-object translation | one connection |
| Omni modeling | full semantic model | thin model over Cube views |

Choose the SQL API path when Cube must remain the single governed engine and
Cube's pre-aggregations are load-bearing. Choose translation when Omni's
curation, AI layer and workbook experience are the point.

⚠️ **Known frictions on the SQL API path**, all of which should be raised before
anyone commits to it:

- Omni re-aggregating an already-aggregated Cube measure is easy to do by
  accident and produces wrong numbers. Model Cube measures as non-aggregatable
  in Omni, or expose pre-shaped views.
- Cube's SQL API dialect is Postgres-compatible but not Postgres; some Omni-generated
  SQL (window functions, certain casts, `LATERAL`) may not be supported. Test the
  specific query shapes Omni generates before promising it works.
- Row-level security moves to Cube's security context, which Omni must supply —
  plan the user-attribute → security-context plumbing explicitly.
- `CUBEJS_SQL_PASSWORD` **must** be set. With dev mode on and no password, the
  SQL API accepts any credentials (see [EDITIONS.md](./EDITIONS.md)).

This integration's skills implement the **translation** path. The SQL API path
is a connection-configuration task, not a semantic-translation one — use
[`omni-admin`](../../../../omni-admin/SKILL.md) to create the connection.

---

## Quick reference — the commands, side by side

| Intent | Omni | Cube Cloud | Cube Core |
|---|---|---|---|
| Who am I | `omni whoami whoami` | `cube whoami` | n/a |
| List models/deployments | `omni models list` | `cube deployments list` | n/a |
| List branches | `omni models list --include activeBranches` | `cube data-model branches <dep>` | `git branch -a` |
| Create a branch | `omni models create-branch <id> --name <n>` | `cube data-model dev-mode <dep> <base>` | `git checkout -b <n>` |
| Read all model files | `omni models yaml-get <id>` | `cube data-model list <dep> --content --json` | read `model/` |
| Read one file | `omni models yaml-get <id> --file-name <f>` | `cube data-model get <dep> <path>` | read the file |
| Write one file | `omni models yaml-create` (whole file) | `cube data-model put <dep> <path> --file <f>` | write the file |
| Compiled model | query with `branchId` | `cube meta --selectors …` | `GET /cubejs-api/v1/meta` |
| Validate | `omni models validate <id> --branch-id <b>` | `cube deployments build-status <dep>` | container reload + `/readyz` |
| Run a query | `omni query run --body …` | `cube-run-query` skill | `POST /cubejs-api/v1/load` |
| Open a PR | `omni models commit` | `cube data-model commit` | `git push` + PR |
| Promote | `omni models merge-branch` 🛑 | merge the PR 🛑 | merge + redeploy 🛑 |
| Discover a command's shape | `--schema` | `--help` | n/a |

🛑 = never without the user asking in this conversation.

---

## Sources

- [Cube CLI reference](https://docs.cube.dev/reference/cli) · [REST API reference](https://docs.cube.dev/reference/core-data-apis/rest-api/reference.md) · [SQL API](https://docs.cube.dev/reference/core-data-apis/sql-api/index.md) · [SQL API security](https://docs.cube.dev/reference/core-data-apis/sql-api/security.md) · [Continuous deployment](https://docs.cube.dev/admin/deployment/continuous-deployment.md)
- [Omni model API](https://docs.omni.co/api/models.md) · [Branch mode](https://docs.omni.co/finding-content/drafting-publishing/branch-mode.md) · [`omni-model-builder`](../../../../omni-model-builder/SKILL.md)
- [`cube-agent-skills`](https://github.com/cube-js/cube-agent-skills)
