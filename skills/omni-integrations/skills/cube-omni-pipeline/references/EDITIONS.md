# Cube Cloud vs. Cube Core — what each edition can do in this pipeline

Read this **before** choosing a workflow. The two editions expose completely
different control surfaces, and the branch choreography in
[BRANCHING.md](./BRANCHING.md) has a separate path for each. Picking the wrong
one produces commands that fail with no useful error.

> **Terminology.** Cube's docs call the open-source, self-hosted product
> **Cube Core** ([docs](https://docs.cube.dev/cube-core/index)) and the managed
> product **Cube Cloud**. The `cube` CLI and the `cube-agent-skills` plugin
> that drive this pipeline are **Cube Cloud clients** — see the table below.
> `data models port unchanged between the two` ([Introduction](https://docs.cube.dev/docs/introduction)),
> which is why the *mapping* half of this integration is edition-independent
> even though the *plumbing* is not.

---

## Detect the edition before anything else

```bash
# A Cube Cloud CLI that authenticates and lists deployments → Cube Cloud.
cube whoami   >/dev/null 2>&1 && cube deployments list && echo "EDITION=cloud"

# A local/self-hosted container answering /readyz on the REST API → Cube Core.
curl -fsS "${CUBE_CORE_URL:-http://localhost:4000}/readyz" >/dev/null 2>&1 \
  && echo "EDITION=core"
```

If both succeed the user has both; **ask which one this task targets** rather
than guessing. A Core project that is *also* imported into Cloud
([import a GitHub repository](https://docs.cube.dev/docs/getting-started/migrate-from-core/import-github-repository.md))
is a common setup, and writing to the wrong side silently diverges the two.

---

## Capability matrix

| Capability | Cube Cloud | Cube Core | Consequence for this pipeline |
|---|---|---|---|
| `cube` CLI (`login`, `context`, `deployments`, `data-model`) | ✅ | ❌ | On Core, every `cube data-model …` command is unavailable. Read and write the YAML **files on disk** instead. |
| `cube-agent-skills` plugin | ✅ full | ⚠️ authoring conventions only | On Core, borrow `cube-build-model`'s YAML conventions but not its dev-mode commands. See [BRANCHING.md](./BRANCHING.md). |
| Dev-mode branches (`cube data-model dev-mode`) | ✅ | ❌ | Core branching = **plain git branches** in the project repo. |
| Server-side model API (`put` / `commit` / `rename` / `delete`) | ✅ | ❌ | Core writes are ordinary file writes + `git commit`. |
| Build as validation (`cube deployments build-status`) | ✅ | ❌ | On Core, validation is **container reload + `/readyz` + `/v1/meta`**. There is no offline validate command on either edition. |
| Compiled model via `cube meta --selectors …` | ✅ | ❌ | On Core, read `GET /cubejs-api/v1/meta` ([reference](https://docs.cube.dev/reference/core-data-apis/rest-api/reference.md)). |
| Compiled model via REST `/v1/meta` | ✅ | ✅ | **The one read path that works on both.** Prefer it for portable logic. |
| SQL API (Postgres wire) | ✅ | ✅ | Both. Used for the alternate architecture in [PIPELINE.md](./PIPELINE.md#alternate-architecture-omni-on-cubes-sql-api). |
| MCP server / Claude connector | ✅ Premium & Enterprise only | ❌ | See "Reading Cube semantics as an LLM" below. |
| Git integration (`cube github connect`) | ✅ | n/a (git *is* the project) | Cloud makes git the source of truth; on Core it already is. |
| Environments & variables (`cube environments`, `cube variables`) | ✅ | ❌ | Core uses `.env` / `docker-compose.yml`. |
| Workbooks, dashboards, Analytics Chat, in-product agents | ✅ | ❌ | `cube-build-content`, `cube-explore-content`, `cube-configure-agent` are Cloud-only and out of scope for a Core round trip. |
| Access policies (`access_policy`) | ✅ | ✅ | Modeled in YAML, so portable. Enforcement depends on a security context being supplied. |
| Tesseract-only members (`number_agg`, `switch` dimensions) | ✅ | ⚠️ version-dependent | Flag these as unmappable-or-risky in both directions. See the ⚠️ rows in each FIELD-MAPPING. |

---

## Reading Cube semantics as an LLM

The requirement "the agent must be able to read and handle Cube's semantics in
full" is satisfied differently per edition.

**Cube Cloud** — three layers, use all three:
1. **Authored YAML** (`cube data-model list <deployment> --content --json`) — the
   only place `sql` expressions, `joins`, `pre_aggregations` and `access_policy`
   live. This is what you translate *from*.
2. **Compiled model** (`cube meta --selectors '[{"type":"cube","deploymentId":<id>,"environment":"<branch>"}]'`)
   — what is actually queryable after `extends`, joins and view exposure resolve.
   This is what you verify *against*.
3. **MCP connector** (Premium/Enterprise) — natural-language access, useful for
   sanity-checking numbers, not for translation. [Docs](https://docs.cube.dev/docs/integrations/mcp-server.md).

**Cube Core** — two layers:
1. **Authored YAML** — the files in `model/cubes/` and `model/views/` on disk.
2. **Compiled model** — `GET /cubejs-api/v1/meta`.
   There is **no MCP server on Core**; do not offer it.

> ⚠️ **`/v1/meta` is not a substitute for the files.** It returns compiled
> metadata — names, types, titles, `meta`, `drillMembers`, `connectedComponent` —
> and **omits every `sql` expression**. A translation driven only by `/v1/meta`
> silently drops all business logic. Always read the YAML for logic, and
> `/v1/meta` for exposure. This mirrors the rule in
> [`cube-explore-model`](https://github.com/cube-js/cube-agent-skills/blob/main/skills/cube-explore-model/SKILL.md):
> *"Use the files when the question is about authoring. Use `cube meta` when the
> question is about availability."*
>
> `/v1/meta` also omits anything with `public: false`, so a private member is
> invisible there but still present — and still translatable — in the files.

---

## Cube Core local setup

Cube Core runs in Docker ([create a project](https://docs.cube.dev/cube-core/getting-started/create-a-project.md)):

```yaml
# docker-compose.yml
services:
  cube:
    image: cubejs/cube:latest
    ports:
      - 4000:4000      # REST + GraphQL + Playground
      - 15432:15432    # SQL API (Postgres wire)
    environment:
      - CUBEJS_DEV_MODE=true
    volumes:
      - .:/cube/conf
```

```bash
docker compose up -d
curl -fsS http://localhost:4000/readyz
```

> 🛑 **`CUBEJS_DEV_MODE=true` is an authentication bypass, not a convenience
> flag.** Cube's own docs are explicit: it switches off JWT verification on the
> REST and GraphQL APIs, mounts Playground with no auth at all, hands any caller
> a ready-to-use API token that can carry **any** security context, lets the SQL
> API accept **any** credentials when `CUBEJS_SQL_PASSWORD` is unset, and exposes
> endpoints that **read your data model and every connected data source's table
> schema, and overwrite your data model and your `.env`**. Member-level access
> control and row-level security are both bypassed.
>
> Use it only on a local machine, never on a shared host, never exposed to the
> internet, never in production, and never against a warehouse you would not
> hand over wholesale. Full warning:
> [Create a project](https://docs.cube.dev/cube-core/getting-started/create-a-project.md)
> · [Running in production](https://docs.cube.dev/cube-core/running-in-production.md).
>
> This matters specifically for this pipeline because a round trip points Cube at
> the **same warehouse Omni reads**. On a shared warehouse, prefer a
> read-only role scoped to the schemas under test.

Linux hosts additionally need `network_mode: 'host'` in the compose file.

---

## Which edition should a given task use?

| Situation | Edition path |
|---|---|
| User says "our Cube deployment", mentions dev mode, workbooks, or Analytics Chat | Cloud |
| User has a `docker-compose.yml` / `model/` directory / `.env` locally | Core |
| User wants the round trip validated end to end with no Cloud account | Core (this is the path validated in this repo — see [SETUP.md](./SETUP.md)) |
| User wants PR review of model changes on the Cube side | Either — Cloud via `cube github connect`, Core via the project repo directly |

## Sources

- [Cube Core](https://docs.cube.dev/cube-core/index) · [Create a project](https://docs.cube.dev/cube-core/getting-started/create-a-project.md) · [Deployment](https://docs.cube.dev/cube-core/deployment.md) · [Running in production](https://docs.cube.dev/cube-core/running-in-production.md)
- [Cube CLI reference](https://docs.cube.dev/reference/cli) · [Cube agent skills](https://github.com/cube-js/cube-agent-skills)
- [REST API reference](https://docs.cube.dev/reference/core-data-apis/rest-api/reference.md) · [SQL API](https://docs.cube.dev/reference/core-data-apis/sql-api/index.md)
- [MCP server](https://docs.cube.dev/docs/integrations/mcp-server.md) · [Dev mode](https://docs.cube.dev/docs/data-modeling/dev-mode.md)
