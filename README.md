# dripline 💧

Query mode for agents.

Turn any API, CLI, or cloud service into a SQL table. Install a plugin, write a query, get rows. It's just SQL, DuckDB under the hood.

```bash
npm install -g dripline
```

## Quick Start

```bash
dripline init
dripline plugin install git:github.com/Michaelliv/dripline#plugins/docker
dripline query "SELECT name, image, state FROM docker_containers"
```

## Plugins

All plugins install via `dripline plugin install git:github.com/Michaelliv/dripline#plugins/<name>`.

| Plugin | Tables | Source |
|--------|--------|--------|
| **github** | repos, issues, pull_requests, stargazers | GitHub API |
| **docker** | containers, images, volumes, networks | Docker CLI |
| **brew** | formulae, casks, outdated, services | Homebrew |
| **ps** | processes, ports | ps, lsof |
| **git** | commits, branches, tags, remotes, status | Git CLI |
| **system-profiler** | software, hardware, network_interfaces, storage, displays | macOS |
| **pi** | sessions, messages, tool_calls, costs, prompt, generate | pi coding agent |
| **kubectl** | pods, services, deployments, nodes, namespaces, configmaps, secrets, ingresses | Kubernetes |
| **npm** | packages, outdated, global, scripts | npm CLI |
| **spotlight** | search, apps, recent | macOS Spotlight |
| **skills-sh** | search | skills.sh registry |
| **cloudflare** | workers, zones, dns_records, pages_projects, pages_deployments, d1_databases, kv_namespaces, r2_buckets, queues, dns_lookup, domain_check | Cloudflare API |
| **vercel** | projects, deployments, domains, env_vars | Vercel API |

## Examples

```sql
-- github: repos by stars
SELECT name, stargazers_count, language
FROM github_repos WHERE owner = 'torvalds'
ORDER BY stargazers_count DESC LIMIT 5;

-- k8s: pods with restarts
SELECT name, namespace, status, restarts
FROM k8s_pods WHERE restarts > 0
ORDER BY restarts DESC;

-- cloudflare: is your domain available?
SELECT domain, available FROM cf_domain_check
WHERE name_prefix = 'myproject' AND tlds = 'com,dev,sh,io,ai';

-- pi: how much have you spent per model?
SELECT model, COUNT(*) as sessions, ROUND(SUM(total_cost), 2) as cost
FROM pi_sessions GROUP BY model ORDER BY cost DESC;

-- skills.sh: top react skills
SELECT name, source, installs FROM skills_search
WHERE query = 'react' ORDER BY installs DESC LIMIT 5;

-- vercel: recent deployments
SELECT name, state, git_commit_message FROM vercel_deployments
WHERE project_name = 'my-blog' LIMIT 5;

-- join API data with a local CSV
SELECT r.name, r.stargazers_count, s.revenue
FROM github_repos r
JOIN read_csv_auto('./revenue.csv') s ON r.name = s.repo
WHERE r.owner = 'torvalds';

-- generate structured data with AI, query it with SQL
SELECT data->>'name' as name, CAST(data->>'age' AS INT) as age
FROM pi_generate
WHERE prompt = 'generate 5 fictional engineers with name, age, city';
```

## Writing a Plugin

Plugins are sync generators. Wrap an API with `syncGet` or a local CLI with `syncExec`:

```typescript
import type { DriplinePluginAPI } from "dripline";
import { syncExec } from "dripline";

export default function(dl: DriplinePluginAPI) {
  dl.setName("my-cli");
  dl.setVersion("1.0.0");

  dl.registerTable("my_things", {
    columns: [
      { name: "name", type: "string" },
      { name: "status", type: "string" },
    ],
    *list() {
      const { rows } = syncExec("mytool", ["list", "--json"], { parser: "json" });
      for (const r of rows) yield { name: r.name, status: r.status };
    },
  });
}
```

Key columns go in `keyColumns` and are extracted from WHERE clauses automatically. `syncExec` supports parsers: `json`, `jsonlines`, `csv`, `tsv`, `lines`, `kv`, `raw`.

See [plugins/](plugins/) for full examples.

## For Agents

Every command supports `--json`. Use `dripline tables --json` for full schemas.

A [pi](https://github.com/badlogic/pi-mono) package is included that injects available tables into the agent context on session start:

```bash
pi install git:github.com/Michaelliv/dripline
```

## CLI Reference

```bash
dripline                              # interactive REPL
dripline query "<sql>"                # execute a query (alias: q)
dripline query "<sql>" -o json        # output as json, csv, or line
dripline tables                       # list all tables and columns
dripline tables --json                # full schema as json
dripline plugin install <source>      # install from git/npm/local path
dripline plugin list                  # list installed plugins
dripline plugin remove <name>         # remove a plugin
dripline connection add <n> -p <plugin> -s key=val  # add connection
dripline connection list              # list connections
dripline init                         # create .dripline/ directory
```

REPL commands: `.tables`, `.inspect <table>`, `.connections`, `.output <format>`, `.help`, `.quit`.

## SDK

```typescript
import { Dripline } from "dripline";
import githubPlugin from "dripline-plugin-github";

const dl = await Dripline.create({
  plugins: [githubPlugin],
  connections: [{ name: "gh", plugin: "github", config: { token: "ghp_xxx" } }],
});

const repos = await dl.query("SELECT name FROM github_repos WHERE owner = 'torvalds' LIMIT 5");
await dl.close();
```

## Configuration

`.dripline/config.json`:

```json
{
  "connections": [{ "name": "gh", "plugin": "github", "config": { "token": "ghp_xxx" } }],
  "cache": { "enabled": true, "ttl": 300, "maxSize": 1000 },
  "rateLimits": { "github": { "maxPerSecond": 5 } }
}
```

Env vars override config. Plugins declare env var names in their connection schema (e.g. `GITHUB_TOKEN`).

## Development

```bash
npm install
npm run dev -- query "SELECT 1"
npm test
npm run check
```

## License

MIT
