# Branch choreography — Cube ↔ Omni

Both platforms support branch-based development. This integration treats
branching as **mandatory, not optional**: a semantic sync rewrites the
definitions that every downstream dashboard, workbook and AI answer depends on,
and it does so on two systems at once. An unbranched sync has no review point
and no way back.

---

## 🛑 Policy: branch by default, always

**1. Never write semantics to a production surface.** That means never to the
Omni **shared model** without a branch, and never to a Cube **production /
default branch** (Cloud) or the project's **default git branch** (Core).

**2. If the user has not named a branch, do not pick one silently.**

| Mode | Behavior when no branch is specified |
|---|---|
| **Interactive** | **Stop and ask.** Propose a name (below) and wait for confirmation or a different name. Do not begin writing. |
| **Auto / headless / unattended** | **Create one** using the naming convention below, then **report the exact names and ids you created on both sides** in your first output. Never fall back to writing unbranched. |

Detect auto-mode from the session: no interactive user to answer (CI, a
scheduled run, a subagent with no channel back to the user), or the user has
explicitly said to proceed without further questions. When in doubt, ask —
an unnecessary question costs one turn, a silent production write costs a
rollback.

**3. Merging is a separate, user-initiated step.** This is the same HARD STOP
that governs [`omni-model-builder`](../../../../omni-model-builder/SKILL.md):

> Do not merge, promote, publish, ship, or "make live" on either platform unless
> the user has told you to **in this conversation**. Preparing the branches —
> create → write → validate → test → report — is the whole job for a "sync this"
> request.

That applies symmetrically:

| Platform | Commands you must not run unprompted |
|---|---|
| Omni | `omni models merge-branch` |
| Cube Cloud | merging the PR opened by `cube data-model commit`; `cube deploy` against a production deployment |
| Cube Core | `git merge` / `git push` to the default branch; `docker compose` restart of a production instance |

If a permission layer blocks one of these, that is the guardrail working.
Surface it and ask — do not route around it.

---

## Branch naming convention

A sync spans two systems, so the branches must be traceable to each other.
Use the same slug on both sides:

```
cube-omni/<direction>-<subject>-<YYYYMMDD>
```

| Direction | Slug prefix | Example |
|---|---|---|
| Cube → Omni | `c2o` | `cube-omni/c2o-revenue-overview-20260922` |
| Omni → Cube | `o2c` | `cube-omni/o2c-margin-measures-20260922` |

Cube Cloud does not let you choose the dev-mode branch name — `cube data-model
dev-mode` forks a personal `dev-…` branch and **prints the name it chose**.
Capture that name and record the pairing explicitly instead:

```
Paired branches for this sync
  Cube  (cloud) : dev-jonathon-4f21a9        deployment 412
  Omni          : cube-omni/c2o-revenue-overview-20260922
                  branchId 7c1e…  modelId 09ac…
```

Report that block every time. It is the only thing that lets a reviewer line up
two PRs a week later.

---

## Identifier cheat sheet

The two platforms name branches differently, and mixing them up is the single
most common failure in this pipeline.

| Concept | Omni | Cube Cloud | Cube Core |
|---|---|---|---|
| What you pass to write commands | `--branch-id <UUID>` | `--branch <name>` | current git branch |
| Where the identifier comes from | `omni models create-branch` → response `model.id` | `cube data-model dev-mode` → printed name | `git checkout -b` |
| Human-readable name | `--name` you chose | auto-assigned `dev-…` | you chose |
| Listing branches | `omni models list --include activeBranches` | `cube data-model branches <deployment>` | `git branch -a` |
| Reading the branch's model | `omni models yaml-get <modelId> --branch-id <id> --mode extension` | `cube data-model list <deployment> --branch <name> --content --json` | read files in the worktree |
| Compiled/queryable model on the branch | query with `"branchId"` in the body | `cube meta … "environment":"<branch>"` | `/v1/meta` after reload |

> ⚠️ **Omni's `branchId` is a server-issued UUID, not the name.** Passing the
> name returns `400 Bad Request: Unrecognized key: "branchName"`.
>
> ⚠️ **Cube Cloud's `--branch` defaults to your active dev-mode branch, and the
> default is per-user, not per-command.** Pass it explicitly whenever more than
> one deployment is in play, or you will write to the wrong place with a
> success response.

---

## The paired-branch workflow

### Step 0 — Preflight both platforms

```bash
# Omni: confirm you can branch at all. QUERY_FULL_MODEL must be present.
omni whoami whoami --model-id <modelId>
```

No `QUERY_FULL_MODEL` → you cannot branch this Omni model, and the sync cannot
proceed as designed. Do not degrade to an unbranched write; report it and stop.
(Permission → capability map: `omni-admin` → *Model Roles & Caller Access*.)

```bash
# Cube Cloud
cube whoami && cube context list && cube deployments list
```

```bash
# Cube Core
curl -fsS "${CUBE_CORE_URL:-http://localhost:4000}/readyz"
git -C "$CUBE_PROJECT_DIR" status --porcelain   # must be clean before branching
```

### Step 1 — Create the branch on the side you are writing to

Create it on **both** sides only when the sync writes to both. Most syncs write
to one side and read from the other; branch the write target, and pin the read
source to a named branch so the translation is reproducible.

```bash
# Omni
omni models create-branch <modelId> --name "cube-omni/c2o-revenue-overview-20260922"
# → response model.id is the branchId; capture it
```

```bash
# Cube Cloud — forks a personal dev branch and prints its name
cube data-model dev-mode <deployment> main
```

```bash
# Cube Core
git -C "$CUBE_PROJECT_DIR" checkout -b cube-omni/o2c-margin-measures-20260922
```

### Step 2 — Write, then validate on the branch

Validation is **not** symmetric, and this trips people up:

| | Omni | Cube Cloud | Cube Core |
|---|---|---|---|
| Offline/static validation | ✅ `omni models validate <modelId> --branch-id <id>` | ❌ none exists | ❌ none exists |
| Real validation | the above, plus a test query | **a build**: `cube deployments build-status <deployment> --branch <name>` | **a reload**: restart the container, then `/readyz` + `/v1/meta` |

> Cube has no linter. As `cube-build-model` puts it: *"Validation is a build,
> not a linter… A failing build is the real error message."* Report Cube build
> errors **verbatim** — they name the file and the member, and a paraphrase
> loses both.

Then prove it is queryable, not merely compilable:

```bash
# Omni — query the branch
omni query run --body '{"query":{"modelId":"<modelId>","table":"<view>","fields":["<view>.<field>"],"limit":10,"join_paths_from_topic_name":"<topic>"},"branchId":"<branchId>"}'
```

```bash
# Cube Cloud — confirm exposure, then run a query
cube meta --selectors '[{"type":"cube","deploymentId":<id>,"environment":"<branch>"}]'
```

```bash
# Cube Core — confirm exposure, then run a query
curl -fsS "$CUBE_CORE_URL/cubejs-api/v1/meta" | jq '.cubes[].name'
curl -fsS "$CUBE_CORE_URL/cubejs-api/v1/load" -H 'Content-Type: application/json' \
  --data '{"query":{"measures":["<view>.<measure>"],"limit":10}}'
```

A member can compile, and still be exposed in no view. A measure can be
exposed, and still be wrong. Do both checks before reporting success.

### Step 3 — Cross-check the numbers

This is the step that makes a semantic sync trustworthy, and it is the one most
often skipped. Run **the same aggregate on both platforms against the same
warehouse rows** and compare:

```
Parity check — revenue, FY2026 Q1, grouped by brand
  Cube  (dev-jonathon-4f21a9) : 4,182,930.44
  Omni  (branch 7c1e…)        : 4,182,930.44   ✅ match
```

> ⚠️ **Compare numerically, never as strings.** Cube serializes numbers as JSON
> floats and Omni as Arrow doubles, so the *same* value prints as `2211783` on
> one side and `2211783.0` on the other, and `2155538.57` as
> `2155538.5700000003`. A `diff` of the two outputs reports differences that do
> not exist. Parse both sides as numbers and compare with a small relative
> tolerance (`1e-9` is ample — it absorbs IEEE-754 representation and nothing a
> human would call a different number).
>
> ⚠️ **Only one Cube Core instance can hold port 4000.** If you keep more than
> one local project, `docker compose down` the other one first. Otherwise the
> new container fails to bind, the *old* one keeps answering `/readyz` and
> `/v1/meta`, and you will happily parity-check **the wrong model** — with a
> plausible-looking result. Confirm the model you expect is the one that
> answers: `curl -fsS $CUBE_CORE_URL/cubejs-api/v1/meta | jq -r '.cubes[].name'`.

A mismatch is a finding, not a rounding problem. The usual causes, in order of
frequency: a fan-out from a `one_to_many` join that only one side deduplicates,
a measure filter that did not survive translation, a timezone or fiscal-calendar
difference, and an unmapped `count_distinct_approx`. See
[LIMITATIONS.md](./LIMITATIONS.md).

### Step 4 — Hand back, do not ship

Report: both branch identifiers, what was written, validation status, the parity
check, and what is **not** mapped. Then stop. See the HARD STOP above.

### Step 5 — Shipping, when the user asks for it

```bash
# Omni, git-connected model → open/update a PR
omni models git-get <modelId>          # sshUrl/baseBranch present → git-connected
omni models commit --body '{"model_id":"<modelId>","branch_id":"<branchId>"}'
# surface the returned pr_url

# Omni, not git-connected → promote into the shared model
omni models merge-branch <modelId> <branchName>
```

```bash
# Cube Cloud
cube data-model commit <deployment> -m "Sync margin measures from Omni" --branch <branch>
cube deployments build-status <deployment> --branch <branch>
cube data-model exit-dev-mode <deployment>
```

```bash
# Cube Core
git -C "$CUBE_PROJECT_DIR" commit -am "Sync margin measures from Omni"
git -C "$CUBE_PROJECT_DIR" push -u origin <branch>   # then open a PR in the host
```

> ⚠️ **Omni git-connected models: never hand-edit model YAML in the git repo.**
> The repo is a *projection* of Omni's model for governance, not the source of
> truth. Omni regenerates the default branch from its own state and **deletes
> git-only model files**. Author through the Omni API on a branch →
> `omni models commit` → review in your git provider. Details:
> [`omni-model-builder`](../../../../omni-model-builder/SKILL.md).
>
> ⚠️ **Cube Cloud: `cube deploy` and `cube github connect` do not mix.** One
> pushes your disk, the other makes git the source of truth; against one
> deployment, whichever ran last wins, silently. Ask which the project uses
> before deploying.

---

## Sources

- [Omni branch mode](https://docs.omni.co/finding-content/drafting-publishing/branch-mode.md) · [Omni model API](https://docs.omni.co/api/models.md) · [`omni-model-builder`](../../../../omni-model-builder/SKILL.md)
- [Cube dev mode](https://docs.cube.dev/docs/data-modeling/dev-mode.md) · [Develop in IDE](https://docs.cube.dev/docs/getting-started/develop-in-ide.md) · [Continuous deployment](https://docs.cube.dev/admin/deployment/continuous-deployment.md)
- [`cube-build-model`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-build-model/SKILL.md) · [`cube-deploy`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-deploy/SKILL.md)
