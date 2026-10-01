# Setup — Cube ↔ Omni semantic pipeline

Everything required before the first sync. Work through it in order; each
section ends with a check you can run.

---

## 1. Omni CLI

```bash
curl -fsSL https://raw.githubusercontent.com/exploreomni/cli/main/install.sh | sh
```

Binaries for macOS (amd64/arm64), Linux (amd64/arm64) and Windows (amd64) are on
the [releases page](https://github.com/exploreomni/cli/releases). The installer
puts `omni` in `~/.local/bin` — make sure that is on your `PATH`.

> **Version matters.** `omni whoami`, the `--schema` body-discovery flag, and
> OAuth login all require **omni CLI ≥ 1.0.7**, and every skill in this
> integration preflights with `omni whoami`. On an older build (e.g. 0.1.2) you
> get `unknown command "whoami"`. **Probe capability rather than checking
> `--version`** — custom and `install.sh`-from-`main` builds carry the features
> without a release version, so a semver check would reject a capable CLI.

```bash
omni config init --name <profile> --endpoint https://<org>.omniapp.co --auth oauth
# or --auth api-key (the key is read from a hidden prompt, never a flag)
omni config use <profile>
omni whoami whoami
```

**Check:** `omni whoami whoami` returns JSON containing `orgRole` and
`rolesByModel`.

**Permission needed:** branching requires `QUERY_FULL_MODEL` on the target
model; merging/promoting additionally requires `UPDATE`. Verify per model:

```bash
omni whoami whoami --model-id <modelId>
```

Without `QUERY_FULL_MODEL` this integration cannot run as designed — it branches
by default and there is no safe unbranched fallback. See
[`omni-admin`](../../../../omni-admin/SKILL.md) → *Model Roles & Caller Access*.

---

## 2. Cube CLI

```bash
curl -fsSL https://raw.githubusercontent.com/cube-js/cube/master/install-cli.sh | sh
```

> ⚠️ **This is a Cube *Cloud* client.** `cube login`, `cube context`,
> `cube deployments`, `cube data-model` and `cube meta` all talk to Cube Cloud's
> public API. **None of them work against a self-hosted Cube Core instance** —
> see [EDITIONS.md](./EDITIONS.md). Install it anyway if there is any chance of a
> Cloud deployment; skip it for a Core-only setup.

```bash
cube login                                  # interactive, opens a browser
export CUBE_API_URL=... CUBE_API_KEY=...    # headless / CI / agent loops
```

**Check:** `cube whoami` succeeds and `cube context list` shows the expected
tenant. Confirm the tenant before any write — a deployment id is only unique
within a tenant.

---

## 3. Cube agent skills (recommended)

The directional skills delegate Cube-side work to Cube's official skills rather
than reimplementing their CLI surface.

```
/plugin marketplace add cube-js/cube-agent-skills
/plugin install cube@cube
```

For Codex, Copilot, Gemini CLI and other skills.sh-compatible agents:

```bash
npx skills add cube-js/cube-agent-skills
```

Nine skills ship there: `cube-explore-model`, `cube-build-model`,
`cube-explore-content`, `cube-build-content`, `cube-run-query`,
`cube-configure-agent`, `cube-admin`, `cube-embed`, `cube-deploy`.
This integration uses the first two, plus `cube-run-query` and `cube-deploy`.

> **Three different things are called "skills" in Cube's ecosystem.** Keep them
> straight: the **Cube connector** (MCP, natural-language questions in Claude),
> **Cube agent skills** (this plugin — operates Cube over the CLI, what we
> delegate to), and **Agent Skills in Cube** (saved workflows your data team
> authors for Analytics Chat). Only the middle one is a dependency here.

**Check:** ask the agent to *"use cube-explore-model to list the deployments"*
and confirm it routes.

**Cube Core note:** these skills are Cloud clients. On Core, borrow
`cube-build-model`'s **YAML authoring conventions** but not its `dev-mode` /
`put` / `commit` commands — the directional skills give Core-native equivalents.

---

## 4. Cube Cloud path

```bash
cube deployments list
cube data-model branches <deployment>
cube deployments build-status <deployment>
```

**Check:** `cube data-model list <deployment>` prints a file tree.
An empty list is real, and usually means an unbuilt or newly created
deployment — say so rather than retrying.

**MCP connector (optional, Premium/Enterprise only):** available in the
[Claude Connectors Directory](https://claude.ai/customize/connectors) — search
for Cube, connect, complete the OAuth flow. Endpoint is
`https://cubecloud.dev/mcp` for every account and region; nothing to configure
by hand. It is useful for asking questions of your data, **not** for
translation — the connector cannot read `sql` expressions any more than
`/v1/meta` can. [Docs](https://docs.cube.dev/docs/integrations/mcp-server.md).

---

## 5. Cube Core path (self-hosted)

### Docker

Cube Core runs in Docker. On macOS:

```bash
brew install --cask docker-desktop     # needs an admin password
open -a Docker                         # first launch requires accepting the terms
docker info                            # must succeed before continuing
```

### Project layout

```
cube-local/
  docker-compose.yml
  .env                 # gitignored — credentials
  .env.example
  model/
    cubes/             one file per cube — the physical layer
    views/             one file per view — the curated layer users query
```

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
    env_file:
      - .env
    volumes:
      - .:/cube/conf
    # Linux hosts also need: network_mode: 'host'
```

```bash
docker compose up -d
curl -fsS http://localhost:4000/readyz          # {"health":"HEALTH"}
```

> 🛑 **`CUBEJS_DEV_MODE=true` is an authentication bypass, not a convenience
> flag.** It disables JWT verification on the REST and GraphQL APIs, mounts
> Playground with no auth, hands any caller a token that can carry **any**
> security context, lets the SQL API accept **any** credentials when
> `CUBEJS_SQL_PASSWORD` is unset, and exposes endpoints that read your data model
> and every connected data source's table schema — and **overwrite your data
> model and your `.env`**. Member-level access control and row-level security are
> both bypassed.
>
> Local machine only. Never on a shared host, never exposed to the internet,
> never in production. This matters especially here because the pipeline points
> Cube at the **same warehouse Omni reads** — prefer a read-only role scoped to
> the schemas under test.
> [Create a project](https://docs.cube.dev/cube-core/getting-started/create-a-project.md) ·
> [Running in production](https://docs.cube.dev/cube-core/running-in-production.md)

### Environment variables

Point Cube at the **same warehouse** the Omni connection reads — otherwise the
parity check is meaningless. For Snowflake:

```bash
CUBEJS_DB_TYPE=snowflake
CUBEJS_DB_SNOWFLAKE_ACCOUNT=<account identifier, e.g. GQB59211 or GQB59211.us-east-1>
CUBEJS_DB_NAME=<database, e.g. ANALYTICS_DEV>
CUBEJS_DB_SNOWFLAKE_WAREHOUSE=<warehouse>
CUBEJS_DB_SNOWFLAKE_ROLE=<role>          # optional, but set it explicitly
CUBEJS_DB_USER=<user>

# Password auth …
CUBEJS_DB_SNOWFLAKE_AUTHENTICATOR=SNOWFLAKE
CUBEJS_DB_PASS=<password>

# … or key-pair auth, which is what Omni's `snowflake-keypair` connections use.
# The authenticator line is REQUIRED — without it Cube attempts password auth
# and the private key is ignored.
CUBEJS_DB_SNOWFLAKE_AUTHENTICATOR=SNOWFLAKE_JWT
CUBEJS_DB_SNOWFLAKE_PRIVATE_KEY_PATH=/cube/conf/snowflake_key.p8   # PKCS#8
CUBEJS_DB_SNOWFLAKE_PRIVATE_KEY_PASS=<passphrase>                 # encrypted keys only
# CUBEJS_DB_SNOWFLAKE_PRIVATE_KEY=<inline key content>            # alternative to the path

# Required whenever the SQL API is reachable at all.
CUBEJS_SQL_USER=cube
CUBEJS_SQL_PASSWORD=<password>
```

> **Mirror the Omni connection rather than guessing.** `omni connections list`
> exposes everything except the secret — `host` (the account identifier),
> `database`, `warehouse`, `username` and `authenticationType`. Read them off it:
>
> ```bash
> omni connections list | jq -r '.connections[]
>   | select(.id=="<connectionId>")
>   | {host, database, warehouse, username, authenticationType}'
> ```
>
> The Snowflake **role** is not exposed by the Omni API, and neither is the
> credential. Those two are the only values a human has to supply.

Find the matching Omni connection's account, database and dialect with:

```bash
omni connections list | jq -r '.connections[] | [.name, .dialect, .host, .database, .id] | @tsv'
```

Other warehouses: [data sources](https://docs.cube.dev/admin/connect-to-data/data-sources).
All variables: [environment variables](https://docs.cube.dev/reference/configuration/environment-variables).

> **Credentials are the usual blocker.** Omni's connection may use key-pair auth
> whose private key the agent does not have. Cube needs its **own** credentials
> for the same warehouse — they are not shareable from Omni. Get these from the
> user before promising a numeric parity check.

### Checks

```bash
curl -fsS http://localhost:4000/readyz
curl -sS -w '\nHTTP %{http_code}\n' http://localhost:4000/cubejs-api/v1/meta | head -c 400
```

`/v1/meta` returning **HTTP 500** is a compile error and the body is the real
message — it names the file and the member. There is no offline validate
command; **a build is the validation**.

With an empty `model/` directory and dev mode on, Cube's Playground at
<http://localhost:4000> has a connection wizard and can generate a starter data
model from your tables — a fast way to bootstrap cubes before hand-editing.

### Branching on Core

Plain git. The project directory *is* the model:

```bash
git -C cube-local init          # if not already a repo
git -C cube-local checkout -b cube-omni/o2c-<subject>-$(date +%Y%m%d)
```

Ensure `.env` and any `*.p8` / `*.pem` keys are gitignored.

---

## 6. Verify the two platforms share a warehouse

The single most important setup check, and the easiest to skip.

```bash
omni connections list | jq -r '.connections[] | [.name, .dialect, .host, .database] | @tsv'
grep -E 'CUBEJS_DB_(TYPE|NAME|SNOWFLAKE_ACCOUNT|HOST)' cube-local/.env
```

State the match explicitly before the first sync. If they differ, the
translation will still produce valid YAML and every number will be
incomparable — the worst kind of failure, because it looks like success.

---

## 7. Optional — Omni on Cube's SQL API

The alternate architecture, where Cube stays the query engine and Omni connects
to its Postgres-wire SQL API as a database connection. Tradeoffs and known
frictions:
[PIPELINE.md → Alternate architecture](./PIPELINE.md#alternate-architecture-omni-on-cubes-sql-api).

Setting it up is connection configuration, not semantic translation — use
[`omni-admin`](../../../../omni-admin/SKILL.md). Set `CUBEJS_SQL_PASSWORD`
first; without it, dev mode accepts any credentials on port 15432.

---

## Setup checklist

| | Check | Command |
|---|---|---|
| ☐ | Omni CLI ≥ 1.0.7, authenticated | `omni whoami whoami` |
| ☐ | Can branch the target Omni model | `omni whoami whoami --model-id <id>` → `QUERY_FULL_MODEL` |
| ☐ | Omni target model exists, schema refreshed | `omni models list` · `omni models refresh <id>` |
| ☐ | Cube edition identified | [EDITIONS.md](./EDITIONS.md) |
| ☐ | *Cloud:* CLI authenticated, tenant confirmed | `cube whoami` · `cube context list` |
| ☐ | *Cloud:* deployment chosen | `cube deployments list` |
| ☐ | *Core:* Docker running | `docker info` |
| ☐ | *Core:* instance healthy | `curl -fsS $CUBE_CORE_URL/readyz` |
| ☐ | *Core:* model compiles | `/cubejs-api/v1/meta` returns 200 |
| ☐ | Cube agent skills installed | `/plugin install cube@cube` |
| ☐ | **Both platforms read the same warehouse** | section 6 |
| ☐ | Branch policy understood | [BRANCHING.md](./BRANCHING.md) |

---

## Sources

- [Omni CLI](https://github.com/exploreomni/cli#readme) · [releases](https://github.com/exploreomni/cli/releases) · [`omni-api-conventions`](../../../../../rules/omni-api-conventions.mdc)
- [Cube CLI reference](https://docs.cube.dev/reference/cli) · [Cube Core getting started](https://docs.cube.dev/cube-core/getting-started/index) · [create a project](https://docs.cube.dev/cube-core/getting-started/create-a-project.md) · [deployment](https://docs.cube.dev/cube-core/deployment.md) · [environment variables](https://docs.cube.dev/reference/configuration/environment-variables) · [data sources](https://docs.cube.dev/admin/connect-to-data/data-sources)
- [`cube-agent-skills`](https://github.com/cube-js/cube-agent-skills) · [Cube connector & skills for Claude](https://cube.dev/blog/cube-connector-and-skills-for-claude) · [MCP server](https://docs.cube.dev/docs/integrations/mcp-server.md)
