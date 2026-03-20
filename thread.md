1/ if there's code mode, there's query mode.

introducing dripline — turns any api, cli, or cloud service into a sql table. install a plugin, write a query, get rows back. joins, aggregations, window functions — duckdb handles the rest.

13 plugins, 57 tables. github, docker, brew, kubectl, cloudflare, vercel, and more.

obligatory @mariozechner pi extension that injects all available tables into your agent's context so it knows what it can query 💧🧵

[snippet: docker example with output]

---

2/ how it works: plugins are sync generators. they call an api or shell out to a cli, yield rows. dripline materializes them into duckdb temp tables. then you write normal sql.

two patterns — wrap an api with syncGet, or wrap a local cli with syncExec:

[snippet: brew plugin code ~10 lines]

---

3/ the pi plugin is my favorite. it reads all your @mariozechner pi session files and turns them into queryable tables.

how much have i spent per model? one query.

[snippet]
SELECT model, COUNT(*) as sessions,
       ROUND(SUM(total_cost), 2) as cost
FROM pi_sessions
GROUP BY model ORDER BY cost DESC;

┌─────────────────┬──────────┬────────┐
│ model           │ sessions │ cost   │
├─────────────────┼──────────┼────────┤
│ claude-opus-4   │ 82       │ 991.09 │
│ claude-sonnet-4 │ 32       │ 173.75 │
│ gpt-5           │ 10       │ 124.15 │
└─────────────────┴──────────┴────────┘

---

4/ what tools does pi actually use the most?

[snippet]
SELECT tool_name, COUNT(*) as calls
FROM pi_tool_calls
GROUP BY tool_name
ORDER BY calls DESC;

┌─────────────────┬───────┐
│ tool_name       │ calls │
├─────────────────┼───────┤
│ bash            │ 18081 │
│ edit            │ 5971  │
│ read            │ 5297  │
│ write           │ 1663  │
│ AskUserQuestion │ 104   │
│ show_widget     │ 66    │
└─────────────────┴───────┘

bash wins by a landslide. obviously.

---

5/ it gets weirder. pi_prompt lets you send prompts to pi and get responses back as sql rows. pi_generate generates structured data with ai and returns it as queryable json.

[snippet]
SELECT data->>'name' as name,
       CAST(data->>'age' AS INT) as age,
       data->>'city' as city
FROM pi_generate
WHERE prompt = 'generate 5 fictional engineers with name, age, city';

┌────────────────┬─────┬──────────┐
│ name           │ age │ city     │
├────────────────┼─────┼──────────┤
│ Talia Vasquez  │ 29  │ Portland │
│ Jun Nakamura   │ 34  │ Tokyo    │
│ Elise Fournier │ 41  │ Lyon     │
│ Kofi Mensah    │ 26  │ Accra    │
│ Darya Sokolova │ 37  │ Berlin   │
└────────────────┴─────┴──────────┘

---

6/ the @CloudflareDev plugin uses their 1.1.1.1 dns api for domain availability checks. no auth needed.

is your project name taken?

[snippet]
SELECT domain, available
FROM cf_domain_check
WHERE name_prefix = 'dripline'
  AND tlds = 'com,dev,sh,io,ai';

┌──────────────┬───────────┐
│ domain       │ available │
├──────────────┼───────────┤
│ dripline.com │ false     │
│ dripline.dev │ true      │
│ dripline.sh  │ true      │
│ dripline.io  │ false     │
│ dripline.ai  │ false     │
└──────────────┴───────────┘

dripline.dev is available btw 👀

---

7/ or query your actual cloudflare infra. workers, zones, pages, d1, kv, r2 — all as tables.

[snippet]
SELECT name, status, plan FROM cf_zones;

┌────────────────┬────────┬──────────────┐
│ name           │ status │ plan         │
├────────────────┼────────┼──────────────┤
│ myapp.dev      │ active │ Free Website │
│ coolproject.sh │ active │ Free Website │
└────────────────┴────────┴──────────────┘

---

8/ shoutout @nichochar — the skills.sh plugin queries the skills registry with sql. what are the most popular react skills?

[snippet]
SELECT name, source, installs
FROM skills_search
WHERE query = 'react'
ORDER BY installs DESC LIMIT 5;

┌─────────────────────────────┬──────────────────────────┬──────────┐
│ name                        │ source                   │ installs │
├─────────────────────────────┼──────────────────────────┼──────────┤
│ vercel-react-best-practices │ vercel-labs/agent-skills │ 231411   │
│ vercel-react-native-skills  │ vercel-labs/agent-skills │ 65689    │
│ react:components            │ google-labs-code/stitch  │ 18740    │
│ react-native-best-practices │ callstackincubator       │ 7909     │
│ react-doctor                │ millionco/react-doctor   │ 6193     │
└─────────────────────────────┴──────────────────────────┴──────────┘

---

9/ vercel plugin auto-detects your auth from vercel login. deployment history as sql.

[snippet]
SELECT name, state, target, git_commit_message
FROM vercel_deployments
WHERE project_name = 'my-blog'
LIMIT 3;

┌─────────┬───────┬────────────┬─────────────────────────────┐
│ name    │ state │ target     │ git_commit_message          │
├─────────┼───────┼────────────┼─────────────────────────────┤
│ my-blog │ READY │ production │ feat: add dark mode support │
│ my-blog │ READY │ production │ fix: mobile nav overflow    │
│ my-blog │ READY │ preview    │ wip: auth flow              │
└─────────┴───────┴────────────┴─────────────────────────────┘

---

10/ k8s too. pods, services, deployments, nodes, configmaps, secrets, ingresses — all queryable.

[snippet]
SELECT name, namespace, status, ready, restarts
FROM k8s_pods WHERE restarts > 0
ORDER BY restarts DESC;

┌──────────────────┬─────────────┬─────────┬───────┬──────────┐
│ name             │ namespace   │ status  │ ready │ restarts │
├──────────────────┼─────────────┼─────────┼───────┼──────────┤
│ kube-scheduler   │ kube-system │ Running │ 1/1   │ 9        │
│ kube-controller  │ kube-system │ Running │ 1/1   │ 9        │
│ api-gateway      │ default     │ Running │ 1/1   │ 3        │
└──────────────────┴─────────────┴─────────┴───────┴──────────┘

---

11/ 13 plugins. 57 tables. all installable from one repo:

dripline plugin install git:github.com/Michaelliv/dripline#plugins/<name>

github · docker · brew · ps · git · system-profiler · pi · kubectl · npm · spotlight · skills-sh · cloudflare · vercel

writing a new plugin is ~30 lines. wrap any cli or api, yield rows, done.

github.com/Michaelliv/dripline 💧
