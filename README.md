# ABEvalFlow Konflux Integration Example

This repository demonstrates how to integrate
[ABEvalFlow](https://github.com/RHEcosystemAppEng/ABEvalFlow) evaluation
into a Konflux application pipeline. It uses the **Google Lightspeed Agent**
as a concrete example, but the same pattern applies to any application.

## Architecture

ABEvalFlow publishes **7 generic evaluation tasks** as Tekton Bundles:

```
parse-snapshot → prepare → test → evaluate → analyze-scorecard → store → emit-result
```

This example wraps those core tasks with **application-specific** deploy and
cleanup tasks:

```
deploy-agent → [ABEvalFlow core tasks] → cleanup-agent (finally)
```

The deploy task knows how to start the Lightspeed agent (with its redis
sidecar, specific env vars, etc.). The cleanup task tears it down. ABEvalFlow
tasks only need the `AGENT_ENDPOINT` — they know nothing about how the agent
is deployed.

## Repository Structure

```
.
├── submission/                          # Evaluation definition
│   ├── metadata.yaml                    # Name, engine, experiment config
│   └── tasks/lightspeed-qa/             # A2A trial definition
│       ├── task.toml                    # Harbor task config
│       ├── instruction.md               # Multi-turn conversation steps
│       ├── tests/test.sh                # Verifier entry point
│       ├── tests/llm_judge.py           # LLM-as-judge scorer
│       └── environment/Dockerfile       # Verifier container
├── pipeline/
│   ├── pipeline.yaml                    # Full PipelineRun (deploy + eval + cleanup)
│   ├── deploy-agent.yaml                # Lightspeed-specific deploy task
│   └── cleanup-agent.yaml               # Lightspeed-specific cleanup task
└── config/
    ├── integration-test-scenario.yaml   # Konflux ITS resource
    └── secrets-template.yaml            # Required secrets template
```

## How to Use This Example

### 1. Create a Submission

A submission defines *what* to evaluate. See `submission/` for the structure:
- `metadata.yaml` — evaluation engine, trial count, gate policies
- `tasks/` — the evaluation scenarios (instructions, judge, verifier)

### 2. Add Your Deploy/Cleanup Tasks

Copy `pipeline/deploy-agent.yaml` and `pipeline/cleanup-agent.yaml` as
starting points, then modify for your application's deployment model:
- Container image, ports, sidecars
- Environment variables
- Readiness checks
- Cleanup resources

### 3. Wire the Pipeline

Edit `pipeline/pipeline.yaml` to reference your deploy/cleanup tasks
around ABEvalFlow's core evaluation tasks.

### 4. Create an IntegrationTestScenario

Apply the ITS from `config/integration-test-scenario.yaml` to your
Konflux tenant namespace, pointing to your pipeline in this repo.

### 5. Provision Secrets

Use `config/secrets-template.yaml` to create required secrets in your
Konflux tenant namespace.

## Parameters

The pipeline accepts these parameters (via the ITS):

| Parameter | Description | Default |
|-----------|-------------|---------|
| `SNAPSHOT` | Konflux Snapshot JSON (automatic) | — |
| `AGENT_IMAGE_OVERRIDE` | Override agent image from snapshot | `""` |
| `LLM_API_BASE` | LLM proxy URL | `http://litellm.ab-eval-flow.svc:4000` |
| `LLM_MODEL` | Judge model | `gpt-4o` |
| `SUBMISSION_REPO_URL` | This repo URL | `https://github.com/ikrispin/abevalflow-konflux-example.git` |
| `SUBMISSION_DIR` | Submission directory | `google-lightspeed-agent` |
| `EVAL_MODE` | `local` or `remote` | `remote` |
| `WORKLOAD_CLUSTER_URL` | Workload cluster API URL | — |
| `WORKLOAD_NAMESPACE` | Namespace on workload cluster | `ab-eval-flow` |

## Adapting for Your Application

1. **Fork this repo** (or create a new one with the same structure)
2. **Replace** `submission/` with your evaluation definition
3. **Replace** `deploy-agent.yaml` with your application's deployment logic
4. **Replace** `cleanup-agent.yaml` with your application's cleanup logic
5. **Update** `pipeline.yaml` parameters for your environment
6. **Update** `integration-test-scenario.yaml` with your app name and repo URL

The ABEvalFlow core tasks (referenced as Tekton Bundles) stay the same — no
changes needed in the ABEvalFlow repo.

## Related

- [ABEvalFlow](https://github.com/RHEcosystemAppEng/ABEvalFlow) — Generic A/B evaluation framework
- [Konflux Integration Guide](https://github.com/RHEcosystemAppEng/ABEvalFlow/blob/main/docs/konflux-integration-guide.md) — Full documentation
