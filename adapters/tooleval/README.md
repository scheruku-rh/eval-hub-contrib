# ToolEval / ToolBench adapter

Community EvalHub provider for **tool-use sequencing** (ToolBench G1/G2/G3)
against a cache-only [StableToolBench](https://github.com/THUNLP-MT/StableToolBench)
virtual API server.

| Image | Role | Who deploys |
|-------|------|-------------|
| `quay.io/evalhub/community-tooleval:latest` | Evaluation Job runtime | EvalHub via `provider.yaml` |
| `quay.io/evalhub/community-toolbench-server:latest` | StableToolBench `/virtual` API | **User / cluster admin** |

## Benchmarks

| ID | Mode | Default `max_steps` |
|----|------|---------------------|
| `tooleval_single_tool` | G1 — one tool call | 1 |
| `tooleval_multi_tool` | G2 — several tools | 3 |
| `tooleval_multi_step` | G3 — iterative observe loop | 5 |

## Behavior

Per task:

1. **MUT** via sidecar (`model.url` + `api-key:ref`) plans tool calls (`action=call|finish`)
2. Adapter **POST /virtual** (cache-only; miss = no egress)
3. Multi-tool / multi-step: repeat until `finish` or `max_steps`
4. **Judge** via sidecar (`judge_api-key:ref` + `judge_url`) scores:
   - **pass_rate** — Solved=1 / Unsure=0.5 / Unsolved=0
   - **win_rate** — WIN=1 / LOSE=0 vs `reference_calls`
5. **MLflow** — `callbacks.mlflow.save(...)` with `trajectories.json` + `summary.json`, then `report_results` (same pattern as ragas/lighteval)
6. **OCI** — when `exports.oci` is set on the job, writes those files under `results/` and calls `callbacks.create_oci_artifact` (ragas/lm-eval pattern)

Fail-fast: tool-server `/health`, optional MUT/judge probe (`probe_models`, default true).

## Lifecycle / callbacks

```text
INITIALIZING → RUNNING_EVALUATION → POST_PROCESSING → PERSISTING_ARTIFACTS
→ callbacks.mlflow.save(results, job_spec, artifacts=...)
→ callbacks.report_results(results)   # owns COMPLETED + additional_info
```

## Judge + MUT secret keys

| Secret key | Purpose |
|------------|---------|
| `api-key` / `*_api-key` | MUT via sidecar |
| `judge_api-key` + `judge_url` | Judge via sidecar |

NetworkPolicy must allow the **sidecar** to reach MUT and judge hosts. Example
ingress policy for the tool server is in `tool-server/deploy.yaml`.

## Build / deploy

```sh
make image-tooleval
make image-toolbench-server                 # fixtures
make image-toolbench-server FETCH_FULL_CACHE=1
make test-tooleval

kubectl -n <tenant> apply -f adapters/tooleval/tool-server/deploy.yaml
```

Example job specs: `examples/job-single-tool.json`, `examples/job-multi-step.json`.

## Testing checklist (before cluster e2e)

- [ ] `make test-tooleval`
- [ ] Tool server: `/health`, `/tools` lists `echo` + `uppercase`, `/virtual` cache hit+miss
- [ ] Job with `tooleval_single_tool` → metrics + MLflow run
- [ ] Job with `tooleval_multi_tool` / `tooleval_multi_step`
- [ ] Secret with `judge_api-key` + `judge_url` (or same MUT endpoint)
- [ ] NetworkPolicy: job → tool server; sidecar → MUT/judge

## Still deferred (not blocking first merge)

- Full ToolBench query dumps / official SoPR dataset packaging
- MirrorAPI for cache misses (GPU / platform decision)
- P2 searchable MLflow params (`tool_subset`, `seed`, …)
- Operator-managed tool-server install; Konflux/FIPS polish
