# Progressive Delivery GitOps

A GitOps repository that bootstraps an OpenShift cluster with Argo CD, Argo Rollouts, and an AI-powered Kubernetes agent for automated canary analysis and progressive delivery demos.

## Architecture

```
+-------------------------+          +--------------------------------------------+
|   This Git Repo         |   sync   |   OpenShift Cluster                        |
|                         +--------->|                                            |
|  bootstrap/             |          |   openshift-gitops namespace               |
|  components/            |          |   +--------------------------------------+ |
|  system/                |          |   | Argo CD         (GitOps controller)  | |
|  workloads/             |          |   | Argo Rollouts   (canary controller)  | |
|                         |          |   | Kubernetes Agent (AI SRE agent)      | |
|                         |          |   +------------------+-------------------+ |
|                         |          |                      |                     |
|                         |          |   quarkus-demo namespace                   |
|                         |          |   +--------------------------------------+ |
|                         |          |   | Rollout: quarkus-demo                | |
|                         |          |   | Services: stable / canary            | |
|                         |          |   | AnalysisTemplate: AI agent call      | |
|                         |          |   +--------------------------------------+ |
+-------------------------+          +--------------------------------------------+

Rollout flow:
  1. Push an image tag change to this repo
  2. Argo CD syncs the new Rollout spec to the cluster
  3. Argo Rollouts creates canary pods (10% -> 60% -> 100%)
  4. AI metric plugin calls the Kubernetes Agent via A2A protocol
  5. Agent fetches stable + canary pod logs, sends them to an LLM
  6. Promote (healthy) or Abort + auto-rollback (errors detected)
  7. On failure, the agent auto-creates a GitHub issue with diagnostics
```

## Repository Structure

```
progressive-delivery/
|-- bootstrap/
|   |-- base/                             OpenShift GitOps operator subscription
|   +-- overlays/default/                 ArgoCD instance, RBAC policy, ApplicationSets
|-- components/
|   |-- applicationsets/                  Git-directory generators for system/* and workloads/*
|   +-- appprojects/                      Argo CD AppProjects (system, workloads)
|-- system/
|   |-- gitops-controller/                Self-managed ArgoCD (points to bootstrap)
|   |-- progressive-delivery-controller/  RolloutManager, AI metric plugin config, RBAC
|   +-- kubernetes-agent/                 Deployment, Service, RBAC, ConfigMap, Secret
|-- workloads/
|   |-- quarkus-rollouts-demo/            Primary demo workload
|   |   |-- base/                         Rollout, services, route, AnalysisTemplate
|   |   +-- overlays/
|   |       |-- scenario-1-stable/        v1.stable image   (healthy release)
|   |       |-- scenario-2-null-pointer/  v2.nullpointer    (NPE in canary)
|   |       |-- scenario-3-memory-leak/   v3.memoryleak     (OOM in canary)
|   |       +-- scenario-4-slow-dep/      v4.slowdependency (downstream timeout)
|   +-- canary-app/                       Simpler canary demo (argoproj/rollouts-demo)
```

## Prerequisites

| Requirement | Details |
|---|---|
| OpenShift cluster | v4.14+ recommended |
| `oc` CLI | Logged in as cluster-admin |
| Argo Rollouts kubectl plugin | [Installation guide](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation) |
| LLM API key | OpenAI **or** Google Gemini |
| GitHub token | Personal access token with `repo` scope |

## Bootstrap

A single command installs the entire stack. It retries automatically because the operator subscription must finish before dependent resources can be created:

```bash
until oc apply -k bootstrap/overlays/default/; do sleep 15; done
```

This installs:

1. **OpenShift GitOps operator** (Argo CD)
2. **ArgoCD instance** with RBAC and OpenShift SSO
3. **ApplicationSets** that auto-discover every directory under `system/` and `workloads/`
4. **Argo Rollouts** with the `argoproj-labs/metric-ai` plugin
5. **Kubernetes Agent** (Quarkus + LangChain4j) for AI-powered log analysis
6. **Quarkus demo workload** with a canary rollout strategy and AI analysis

### Configure the Agent Secret

After the `openshift-gitops` namespace exists, provide your API credentials:

```bash
cp system/kubernetes-agent/secret.yaml.template system/kubernetes-agent/secret.yaml
# Edit secret.yaml with your actual keys
oc apply -f system/kubernetes-agent/secret.yaml
```

Set the LLM backend in `system/kubernetes-agent/configmap.yaml`:

| Profile | `QUARKUS_PROFILE` value | Default model |
|---|---|---|
| OpenAI | `prod,openai` | `gpt-4.1-mini` |
| Gemini | `prod,gemini` | `gemini-2.5-flash` |

### Fork Note

If you fork this repo, update the `repoURL` fields in `components/applicationsets/system-appset.yaml` and `workloads-appset.yaml` to point to your fork.

## Demo Scenarios

Each scenario uses a different container image tag that exhibits a specific behavior during canary analysis. To trigger a scenario, change the image tag in `workloads/quarkus-rollouts-demo/base/rollout.yaml`:

| Scenario | Image Tag | Behavior | Agent Action |
|---|---|---|---|
| Stable release | `v1.stable` | Clean rollout -- AI finds no issues, canary is promoted | None |
| NullPointerException | `v2.nullpointer` | Canary pods throw NPE -- AI detects the error, rollout aborts | Creates **PR** with fix |
| Memory leak | `v3.memoryleak` | Canary pods leak memory -- AI detects OOM patterns, rollout aborts | Creates **Issue** with RCA |
| Slow dependency | `v4.slowdependency` | Downstream service degrades, 30% timeout rate after 60 s | Creates **Issue** with RCA |

### Running a Scenario

1. Edit the image tag in `workloads/quarkus-rollouts-demo/base/rollout.yaml`:

   ```yaml
   containers:
   - name: quarkus-demo
     image: quay.io/danieloh30/argo-rollouts-quarkus-demo:v2.nullpointer
   ```

2. Commit and push:

   ```bash
   git add workloads/quarkus-rollouts-demo/base/rollout.yaml
   git commit -m "Switch to v2.nullpointer scenario"
   git push
   ```

3. Argo CD detects the change and syncs the new Rollout spec.

4. Watch the rollout progress:

   ```bash
   oc argo rollouts get rollout quarkus-demo -n quarkus-demo --watch
   ```

5. During the canary phase the AI metric plugin calls the Kubernetes Agent, which fetches logs from stable and canary pods, analyzes them with the configured LLM, and returns a promote or abort decision. On abort, the agent creates a GitHub PR (code bugs) or Issue (operational problems) automatically.

6. To reset, switch the image tag back to `v1.stable` and push.

### Verify the Deployment

```bash
# Argo Rollouts controller
oc get pods -n openshift-gitops -l app.kubernetes.io/name=argo-rollouts

# Kubernetes Agent
oc get pods -n openshift-gitops -l app=kubernetes-agent

# Application rollout status
oc argo rollouts get rollout quarkus-demo -n quarkus-demo

# Analysis run results
oc get analysisrun -n quarkus-demo
```
