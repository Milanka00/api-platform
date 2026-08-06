# performance-test-scripts — API Gateway EKS performance scripts

Minimal scripts for **Jenkins API Gateway EKS** performance tests.
Lives in the public `api-platform` repo under `gateway/perf/performance-test-scripts/`.

Supports the following RestApis and scenarios:

| RestApi (deploy) | Policy | JMeter scenario (`-i`) | Path (8 routes: base + `/r1`–`/r7`) |
|------------------|--------|------------------------|------|
| `perf-api-plain` | none (plain) | `api_api_plain_get` | `/api-plain/1.0.0/chat/completions` |
| `perf-api-header` | set-headers | `api_api_header_get` | `/api-header/1.0.0/chat/completions` |
| `perf-api-jwt` | jwt-auth | `api_api_jwt_get` | `/api-jwt/1.0.0/chat/completions` |

Each RestApi is deployed with `-r 8` (16 operations: GET+POST × 8 paths). JMeter randomizes among the same 8 path suffixes via `resourceSuffixes=,/r1,...,/r7`.

---

## Public-repo checklist (before opening a PR)

**Safe to commit:** `api-gateway/`, `jmeter/`, `lib/`, `common.env.example`, `README.md`, `.gitignore`.

**Never commit:**

| Path | Why |
|------|-----|
| `env.eks`, `jmeter/env.jmeter*`, `.generated-values.yaml` | Generated per run (gitignored) |
| `*-perf-test-results/`, `api_ids.csv` | Result dumps / environment-specific IDs |
| Real Asgardeo tenant URLs / OAuth secrets | Use `YOUR_TENANT` placeholders + Jenkins env |

The legacy files in the parent `gateway/perf/` directory (`*-perf-test-results/`, `api_ids.csv`, `create_apis_and_capture_stats.sh`, `weather_perf_random_50.jmx`) are outside this pack and are not fetched by the sparse clone.

`config.perf-overlay.toml` uses `admin`/`admin` for the **perf-only** controller basic auth and `YOUR_TENANT` in the jwt-auth issuer/JWKS URLs.

`YOUR_TENANT` is substituted at install time by `eks-common.sh` from `JWT_TENANT`, or derived from `JWT_OAUTH_TOKEN_URL` (the app that mints `jwt-tokens.csv`). If neither is set the placeholder stays and **every `api_api_jwt_get` request returns 401** — the install logs a warning in that case.

---

## How Jenkins gets these scripts

`api-gateway-eks-perf/run-api-gateway-eks-tests.sh` sparse-clones this tree via one Jenkins param:

| Env | Format / default |
|-----|------------------|
| `PERF_SCRIPTS` | `repo@branch:subdir` → `https://github.com/wso2/api-platform.git@main:gateway/perf/performance-test-scripts` |

After clone it copies into workspace `performance-test-scripts/`. Fork testing:

```bash
export PERF_SCRIPTS=https://github.com/<you>/api-platform.git@my-perf-branch:gateway/perf/performance-test-scripts
```

---

## Workspace layout (Jenkins slave after job start)

```
product-performance-test/              # PERF_ROOT
├── api-gateway-eks-perf/              # orchestrator (on slave)
├── performance-test-scripts/          # cloned from gateway/perf/performance-test-scripts
├── performance-common/                # cloned for jtl-splitter
└── .perf-scripts-src/                 # sparse clone cache (ephemeral)
```

---

## Jenkins job parameters

| Parameter | Example | Notes |
|-----------|---------|-------|
| `RUN_PERF_OPTS` | `-u 1000 -b 1 -s 0 -d 900 -w 180 -i api_api_plain_get` | **Load/scenario only** (see below) |
| `GATEWAY_HELM_CHART_VERSION` | `1.2.0-rc` | Pin in Jenkins; see Helm upgrades below |
| `GATEWAY_NODE_INSTANCE_TYPE` | `c5.2xlarge` | One node ≈ one 4-CPU runtime pod |
| `JWT_OAUTH_*` | Asgardeo client creds | Required for JWT token mint; `JWT_OAUTH_TOKEN_URL` also sets the gateway's JWT issuer |

### `RUN_PERF_OPTS` (configure these)

| Flag | Meaning | Multiple? |
|------|---------|-----------|
| `-u` | Concurrent users | yes (`-u 100 -u 500 -u 1000`) |
| `-b` | Message size bytes | yes |
| `-s` | Backend sleep ms | yes |
| `-d` | Test duration seconds (default 900) | no |
| `-w` | Warm-up seconds (default 300) | no |
| `-i` | Include scenario | yes (`-i api_api_plain_get -i api_api_header_get`) |
| `-e` | Exclude scenario | yes |

Scenarios: `api_api_plain_get`, `api_api_header_get`, `api_api_jwt_get`.

Do **not** put `-n/-m/-j/-k/-l/-r` in `RUN_PERF_OPTS` — the orchestrator sets those:

| Env (optional override) | Default | Role |
|-------------------------|---------|------|
| (fixed) `-n` | `2` | JMeter servers |
| `PERF_HEAP_LABEL` | `16G` | results path label only |
| `JMETER_SERVER_HEAP` | `4G` | `-j` |
| `JMETER_CLIENT_HEAP` | `2G` | `-k` (use 2G on c5.xlarge) |
| `NETTY_SERVICE_HEAP` | `4G` | `-l` |
| `RESPONSE_SIZE_BYTES` | `1` | `-r` (tiny echo body) |

Example multi-user / multi-scenario:

```text
-u 500 -u 1000 -b 1 -s 0 -d 900 -w 180 -i api_api_plain_get -i api_api_header_get
```

### `summary.csv` columns

Only these are written (GC / SAR / heap / message-size / label columns are dropped):

`Scenario Name`, `Concurrent Users`, `Throughput (Requests/sec)`, `Average Response Time (ms)`, `# Samples`, `Error Count`, `Error %`, `Average Users in the System`, `Standard Deviation of Response Time (ms)`, `Minimum Response Time (ms)`, `Maximum Response Time (ms)`, percentiles (75/90/95/98/99/99.9), `Received (KB/sec)`, `Sent (KB/sec)`.

---

## Gateway runtime tuning (where to change)

Jenkins writes `performance-test-scripts/api-gateway/eks/env.eks` each run from
`api-gateway-eks-perf/run-api-gateway-eks-tests.sh` (Step 2). Edit that block for job defaults:

```bash
export GATEWAY_RUNTIME_REPLICAS="${GATEWAY_RUNTIME_REPLICAS:-1}"
export GATEWAY_RUNTIME_CPU_LIMIT="4"
export GATEWAY_RUNTIME_MEM_LIMIT="2Gi"
export ROUTER_CONCURRENCY="4"
export GOMAXPROCS="4"
export LOG_LEVEL="error"
export POLICY_ENGINE_METRICS_ENABLED="false"
# optional:
# export GOGC="200"
# export GOMEMLIMIT="1400MiB"
```

Flow: `env.eks` → `install-gateway.sh` → `eks-common.sh` → `.generated-values.yaml` → Helm → live Deployment.

Manual EKS (outside Jenkins): edit `api-gateway/eks/env.eks` from `env.eks.example`, then `source env.eks && ./install-gateway.sh`.

Do **not** edit `.generated-values.yaml` by hand — regenerated each install.

---

## Adding a new RestApi + JMeter scenario

Example: add `api-ratelimit` with basic-ratelimit policy.

### 1. Deploy script — `api-gateway/deploy/create-rest-perf-api.sh`

- Add a new `-m` mode if the policy combination is new (or reuse `plain` / `add_headers` / `jwt_auth`).
- Implement policy YAML in the `case "$api_mode"` block.

### 2. EKS deploy — `api-gateway/eks/deploy-apis-eks-minimal.sh`

- Add the metadata name to `keep_apis=(...)`.
- Add a deploy line, e.g. `"${DEPLOY_DIR}/create-rest-perf-api.sh" -n api-ratelimit -m plain -r 8 -l`.
- Keep `-r` in sync with `PERF_API_ROUTE_COUNT` / `_perf_api_route_suffixes` in `gateway-scenarios.sh` (default **8** = base + `/r1`–`/r7`).

### 3. JMeter scenario — `jmeter/gateway-scenarios.sh`

- Add `test_scenarioN` with matching `[path]`, `[jmx]`, and register in `test_scenario` map.
- If auth tokens needed, add entries to `scenario_api_key_file` / `scenario_auth_header`.

### 4. JMX (if needed)

- Reuse `api-api-test-gateway-plain.jmx` for GET without body auth quirks.
- Use `api-api-test-gateway-jwt-plain.jmx` when `Authorization: Bearer` header is required.

### 5. Config overlay — `api-gateway/config.perf-overlay.toml`

- Add policy system config (keymanager, ratelimit backend, etc.) if the new policy needs it.
- JWT keymanager name must match `JWT_KEYMANAGER_NAME` in `env.eks` (default `test`).

### 6. Jenkins

- Document new `-i <scenario_name>` in the job parameter help.
- No orchestrator change unless deploy script name or paths change.

---

## Upgrading Helm / gateway chart version

When moving to a new chart (e.g. `1.2.0-rc` → `1.3.0`):

1. **Jenkins parameter** — set `GATEWAY_HELM_CHART_VERSION`.
2. **`env.eks.example`** — update default chart version comment if you maintain it.
3. **`run-api-gateway-eks-tests.sh`** — verify `GATEWAY_MGMT_API_BASE` (v1 vs v0.9) still correct.
4. **`eks-common.sh`** — run a test install; check for chart value renames in upstream `values.yaml`.
5. **`values.perf.yaml`** — merge any new required Helm keys from the new chart defaults.
6. **`config.perf-overlay.toml`** — confirm overlay keys still valid (some keys are chart/version-specific; comments in file note unsupported sections).
7. **Controller port** — `GATEWAY_CONTROLLER_PORT` (default `19090`) if the new chart changes management port.
8. **Images** — official chart pulls `ghcr.io/wso2/api-platform/gateway-*:<version>`; override with `GATEWAY_RUNTIME_IMAGE` only for custom builds.
9. **Re-run** `deploy-apis-eks-minimal.sh` after upgrade — RestApi CRD/API shape may change between versions.

After upgrade, smoke test from slave:

```bash
curl -sf "http://${NLB_HOST}:8080/api-plain/1.0.0/chat/completions" -o /dev/null -w '%{http_code}\n'
```

---

## File map

| Path | Role |
|------|------|
| `api-gateway/eks/install-gateway.sh` | Helm install/upgrade |
| `api-gateway/eks/eks-common.sh` | Generates Helm values from `env.eks` |
| `api-gateway/eks/deploy-apis-eks-minimal.sh` | Deploy 3 RestApis, delete others |
| `api-gateway/eks/backend-mock-eks.yaml` | In-cluster Netty mock |
| `api-gateway/eks/values.perf.yaml` | Static Helm overrides |
| `api-gateway/config.perf-overlay.toml` | Embedded gateway config (access logs off, JWT, ratelimit) |
| `api-gateway/deploy/create-rest-perf-api.sh` | RestApi YAML builder (plain / header / jwt) |
| `jmeter/run-scenario.sh` | Distributed test driver |
| `jmeter/gateway-scenarios.sh` | Scenario definitions (3 only) |
| `jmeter/generate-jwt-tokens.sh` | Asgardeo OAuth → `jwt-tokens.csv` |
| `jmeter/*.jmx` | JMeter test plans |

---

## Not included

AI gateway, EC2 docker-compose, dual-gateway, route-scale APIs, fetch-artifacts, docs from the full manual tree.
