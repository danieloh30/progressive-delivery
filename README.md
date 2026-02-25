# OpenShift Service Mesh with Argo Rollouts + AI Metrics Plugin

Example Repo that does what the title says, now with AI-powered canary analysis. You can watch a demo of the base setup: [HERE](https://youtu.be/xURFbR7zNIE?si=OZH4QmMFsTfWW3Py)

## What's New: AI Metrics Plugin

This repo now includes an AI-powered metrics plugin that analyzes your canary deployments using an autonomous Kubernetes agent. The agent:

- Fetches logs from stable and canary pods
- Analyzes them with AI (Gemini or OpenAI)
- Decides whether to promote or abort the canary
- Can create GitHub PRs with fixes when issues are found

### How it works

```
Argo Rollouts (with AI plugin) 
    ↓ (A2A protocol)
Kubernetes Agent (Quarkus + LangChain4j)
    ↓ (fetches logs)
Your Pods (stable + canary)
```

The plugin delegates all AI work to the agent - no LLM config needed in the plugin itself.

# Prereqs

* OpenShift Cluster v4.10+
* [Argo Rollout Kubernetes CLI Plugin](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation)
* **Google API Key** (for Gemini) OR **OpenAI API Key**
* **GitHub Personal Access Token** (with `repo` scope)

This repo assumes that this is a "fresh" cluster. Don't run these steps on a production cluster or a cluster you care about. This is for testing only.

# Setup

## Fork the repo

First, fork this repo. You will need to change the [Application Sets in this directory](components/applicationsets) to point to your fork.

## Create Secrets

Before deploying, you need to create a secret with your API keys.

### Kubernetes Agent Secret

```shell
# Copy the template
cp system/kubernetes-agent/secret.yaml.template system/kubernetes-agent/secret.yaml

# Edit and add your keys
vim system/kubernetes-agent/secret.yaml
```

Fill in your credentials:

```yaml
stringData:
  # For Gemini (recommended)
  google_api_key: "YOUR_GOOGLE_API_KEY"
  
  # OR for an OpenAI-spec supported model:
  openai_api_key: "YOUR_OPENAI_API_KEY"
  openai_model: "my-model"
  openai_base_url: "https://my-llm-server.com/v1"
  
  # GitHub token for PR & issue creation
  github_token: "YOUR_GITHUB_TOKEN"
```

**Where to get keys:**
- Google API Key: https://aistudio.google.com/app/apikey
- OpenAI API Key: https://platform.openai.com/api-keys (in case you're using openai. you can also use any compatible OpenAI server like vLLM, Ollama, etc.)
- GitHub Token: https://github.com/settings/tokens (needs `repo` scope)

Then apply it:

```shell
oc apply -f system/kubernetes-agent/secret.yaml
```

## Deploy the Application

After editing the [Application Sets in this directory](components/applicationsets) to point to your fork, apply it to your OCP cluster

> **NOTE** Errors are expected in this step. Go get some coffee, go for a walk, then comeback to this.

```shell
until oc apply -k bootstrap/overlays/default/; do sleep 15; done
```

This deploys:
- OpenShift GitOps (Argo CD)
- Argo Rollouts with AI metrics plugin
- Kubernetes Agent
- Istio Service Mesh
- Sample canary app

## Verify Everything is Running

```shell
# Check Argo Rollouts
oc get pods -n openshift-gitops | grep argo-rollouts

# Check Kubernetes Agent
oc get pods -n openshift-gitops | grep kubernetes-agent

# Test agent health
oc port-forward -n openshift-gitops svc/kubernetes-agent 8080:8080 &
curl http://localhost:8080/q/health
# Should return: {"status":"UP",...}

# Check the plugin is loaded
oc logs deployment/argo-rollouts -n openshift-gitops | grep -i "metric-ai"
# Should see: "Successfully loaded plugin: argoproj-labs/metric-ai"
```

## See the App

You should see the app on your browser; first export your Gateway

```shell
export GATEWAY_URL=$(oc -n istio-system get route istio-ingressgateway -o jsonpath='{.spec.host}')
```

Then open in browser, example

```shell
firefox $GATEWAY_URL
```

You should see this

![sample-app](https://i.ibb.co/G2gY1b5/sample-app.png)

# Testing

## Make update

Update the `workloads/canary-app/kustomization.yaml` file from `blue` to `yellow`. Edit the file by hand but if you're brave, you can run a `sed` on the file.

```shell
sed -i 's/blue/yellow/g' workloads/canary-app/kustomization.yaml
```

Then commit/push to your fork

```shell
git add .
git commit -am "yellow"
git push
```

## Observe

Get status using the plugin (running a `watch` is helpful)

```shell
oc argo rollouts get rollout rollouts-demo -n canary
```

You'll see that the blue squares turn into yellow ones. It increases every 15 (or so) seconds until you're fully yellow.

**What's happening with AI analysis:**

At each canary step, the AI metrics plugin:
1. Sends a request to the Kubernetes Agent
2. Agent fetches logs from stable and canary pods
3. Agent analyzes with AI (Gemini/OpenAI)
4. Agent returns promote/abort decision
5. Rollout continues or aborts based on analysis

You can see the analysis results:

```shell
# List analysis runs
oc get analysisrun -n canary

# View specific analysis
oc get analysisrun <name> -n canary -o yaml
```

## Auto-Rollback

In the UI, you'll see a slider that causes the application to return a 500 error

![500err](https://i.ibb.co/LzzWqX4/witherr.jpg)

Change the application back to blue

```shell
sed -i 's/yellow/blue/g' workloads/canary-app/kustomization.yaml
```

Commit and push...

```shell
git add .
git commit -am "blue with errors"
git push
```

Initiate the rollout by refreshing the Argo CD "canary-app" Application. This will initiate a rollout.

Once the rollout has started, slide the error slider on the application to 100% and watch as the rollout fails

```shell
oc argo rollouts get rollout rollouts-demo -n canary
```

It should fail and it should look something like this...

```shell
NAME                                       KIND         STATUS        AGE    INFO
⟳ rollouts-demo                            Rollout      ✖ Degraded    3h27m  
├──# revision:26                                                             
│  ├──⧉ rollouts-demo-6499d5bbb9           ReplicaSet   • ScaledDown  16m    canary,delay:passed
│  └──α rollouts-demo-6499d5bbb9-26        AnalysisRun  ✖ Failed      2m44s  ✖ 3
├──# revision:25                                                             
│  ├──⧉ rollouts-demo-785bb66569           ReplicaSet   ✔ Healthy     41m    stable
│  │  ├──□ rollouts-demo-785bb66569-7gwpj  Pod          ✔ Running     13m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-hqmlc  Pod          ✔ Running     13m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-rm7j5  Pod          ✔ Running     13m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-mjcq5  Pod          ✔ Running     12m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-h4tzf  Pod          ✔ Running     12m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-v6qn2  Pod          ✔ Running     12m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-fvvmb  Pod          ✔ Running     11m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-jdt27  Pod          ✔ Running     11m    ready:2/2
│  │  ├──□ rollouts-demo-785bb66569-f6krk  Pod          ✔ Running     11m    ready:2/2
│  │  └──□ rollouts-demo-785bb66569-w7w6w  Pod          ✔ Running     11m    ready:2/2
```

Your app should have failed back to being all yellow since you had errors during the rollout. The AI agent detected the errors in the canary logs and recommended aborting.

To restart/fix this. Move the error slider to 0% and restart the rollout.

```shell
oc argo rollouts retry rollout rollouts-demo -n canary
```

This time, the rollout should finish and you should have blue squares.

# Architecture Details

## How the Plugin is Registered

The AI metrics plugin is registered in the Argo Rollouts ConfigMap:

```yaml
# system/progressive-delivery-controller/argo-rollouts-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-rollouts-config
  namespace: openshift-gitops
data:
  metricProviderPlugins: |-
    - name: argoproj-labs/metric-ai
      location: file:///home/argo-rollouts/rollouts-plugin-metric-ai
```

The plugin binary is baked into the Argo Rollouts container image:

```yaml
# system/progressive-delivery-controller/rolloutmanager.yaml
spec:
  image: ghcr.io/argoproj-labs/rollouts-plugin-metric-ai
  version: latest
```

## How AnalysisTemplates Use the Plugin

The AnalysisTemplate references the plugin by name:

```yaml
# workloads/canary-app/analysistemplate-ai-agent.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: ai-analysis-agent
  namespace: canary
spec:
  metrics:
    - name: ai-analysis
      provider:
        plugin:
          argoproj-labs/metric-ai:
            # Agent URL (required)
            agentUrl: http://kubernetes-agent:8080
            # Pod selectors for log fetching
            stableLabel: role=stable
            canaryLabel: role=canary
            # Optional: GitHub integration
            githubUrl: https://github.com/kdubois/progressive-delivery
            baseBranch: main
            # Optional: Extra context for AI
            extraPrompt: "Ignore color changes. Consider LoadBalancerNegNotReady a temporary condition."
```

## How the Agent Works

The Kubernetes Agent is deployed as a separate service:

```yaml
# system/kubernetes-agent/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-agent
  namespace: openshift-gitops
spec:
  template:
    spec:
      containers:
        - name: agent
          image: quay.io/kevindubois/kubernetes-agent:latest
          env:
            - name: GOOGLE_API_KEY
              valueFrom:
                secretKeyRef:
                  name: kubernetes-agent
                  key: google_api_key
            - name: GITHUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: kubernetes-agent
                  key: github_token
```

When the plugin calls the agent:

1. Plugin sends A2A request to `http://kubernetes-agent:8080/a2a/analyze`
2. Agent uses Kubernetes API to fetch logs (has RBAC permissions)
3. Agent analyzes logs with AI model (Gemini or OpenAI)
4. Agent returns structured JSON with promote/abort decision
5. Plugin uses the decision to pass/fail the metric
6. Argo Rollouts promotes or aborts based on metric result

# Troubleshooting

## Plugin not loading

```shell
# Check ConfigMap
oc get configmap argo-rollouts-config -n openshift-gitops -o yaml

# Check Rollouts logs
oc logs deployment/argo-rollouts -n openshift-gitops | grep -i plugin
```

## Agent connection failed

```shell
# Check agent is running
oc get pods -n openshift-gitops | grep kubernetes-agent

# Check agent logs
oc logs deployment/kubernetes-agent -n openshift-gitops

# Test connectivity
oc exec -it deployment/argo-rollouts -n openshift-gitops -- \
  curl http://kubernetes-agent:8080/q/health
```

## Agent health check failing

```shell
# Check logs for errors
oc logs deployment/kubernetes-agent -n openshift-gitops | grep -i error

# Common issues:
# 1. Missing API key - check secret
oc get secret kubernetes-agent -n openshift-gitops -o yaml

# 2. Invalid API key - check agent logs for auth errors

# 3. Out of memory - check resource limits
oc top pod -n openshift-gitops | grep kubernetes-agent
```

## Analysis always fails

```shell
# Check AnalysisTemplate config
oc get analysistemplate ai-analysis-agent -n canary -o yaml

# Verify pod labels match
oc get pods -n canary --show-labels

# Check agent can fetch logs
oc logs deployment/kubernetes-agent -n openshift-gitops | grep -i "fetching logs"
```

## GitHub PRs not created

```shell
# Check GitHub token
oc get secret kubernetes-agent -n openshift-gitops -o jsonpath='{.data.github_token}' | base64 -d

# Token needs 'repo' scope

# Check agent logs
oc logs deployment/kubernetes-agent -n openshift-gitops | grep -i github
```

## Debug mode

Enable debug logging:

```shell
# For plugin (in Rollouts controller)
oc set env deployment/argo-rollouts LOG_LEVEL=debug -n openshift-gitops

# For agent
oc set env deployment/kubernetes-agent QUARKUS_LOG_LEVEL=DEBUG -n openshift-gitops

# View logs
oc logs -f deployment/argo-rollouts -n openshift-gitops
oc logs -f deployment/kubernetes-agent -n openshift-gitops
```

## Common errors

| Error | Fix |
|-------|-----|
| `agentUrl is required` | Add `agentUrl: http://kubernetes-agent:8080` to AnalysisTemplate |
| `agent health check failed` | Check agent pod is running and healthy |
| `failed to fetch logs` | Check RBAC permissions for agent ServiceAccount |
| `authentication failed` | Check API key in secret is valid |
| `context deadline exceeded` | Check network connectivity to agent |

# Advanced Config

## Switch AI models

Edit the secret to switch between Gemini and OpenAI:

```shell
oc edit secret kubernetes-agent -n openshift-gitops

# For Gemini:
# google_api_key: "YOUR_KEY"

# For OpenAI:
# openai_api_key: "YOUR_KEY"
# openai_model: "gpt-4o"

# Restart agent
oc rollout restart deployment/kubernetes-agent -n openshift-gitops
```

## Custom analysis prompts

Add context-specific instructions:

```yaml
extraPrompt: |
  This is a payment service.
  Focus on transaction errors and database issues.
  Ignore UI color changes.
```

## More info

- Plugin README: `rollouts-plugin-metric-ai/README.md`
- Agent README: `kubernetes-agent/README.md`
- Argo Rollouts docs: https://argoproj.github.io/argo-rollouts/
