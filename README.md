 # kube-agent
 **kube-agent** is a Kubernetes-native AI agent that lets me interact with Kubernetes clusters using natural language while keeping the operational workflow grounded in real `kubectl` actions.

 Instead of remembering every `kubectl` command, I can ask questions like:

```
What's wrong with the api deployment?
```
```
Why are these pods pending?
```
```
Show me the unhealthy workloads in production.
```
```
Fix the nginx deployment and verify the rollout.
```

 The goal is simple: **turn Kubernetes operational intent into observable, explainable cluster actions.**

 This project is built with a production/SRE mindset: inspect first, understand the failure, make the smallest appropriate change, and verify the result.

---

 ## Why kube-agent?

 Kubernetes is extremely powerful, but operating it at scale still involves a large amount of repetitive investigation:

 - checking pods, nodes, deployments, and events
- tracing scheduling failures
- investigating failed image pulls
- understanding rollout failures
- inspecting resources and their relationships
- remembering the right `kubectl` command
- validating that a remediation actually worked

 I built `kube-agent` to make that workflow more natural.

 Instead of:

```
kubectl get pods
kubectl describe pod <pod>
kubectl get events
kubectl describe deployment <deployment>
kubectl get nodes --show-labels
kubectl rollout status deployment/<deployment>
```

 I can start with:

```
Why isn't my application running?
```

 The agent can inspect the cluster, reason about the available evidence, execute the required Kubernetes operations, and explain what it found.

---

 ## Core principles

 `kube-agent` is designed around a few principles I consider important for production Kubernetes tooling.

 ### 1\. Observe before changing

 The agent should understand the current cluster state before attempting remediation.

```
Inspect → Diagnose → Explain → Change → Verify
```

 ### 2\. Prefer the smallest safe change

 Kubernetes problems often have several possible solutions.

 The agent should identify the root cause and present reasonable remediation paths rather than blindly modifying resources.

 ### 3\. Verify every operational change

 A successful API request does not necessarily mean the workload is healthy.

 After making a change, `kube-agent` should verify the resulting state:

```
Apply change
     ↓
Observe rollout
     ↓
Check workload health
     ↓
Confirm expected state
```

 ### 4\. Kubernetes remains the source of truth

 The agent does not maintain a separate representation of cluster state.

 It works with the actual Kubernetes API through the configured Kubernetes context and uses `kubectl` for cluster operations.

 ### 5\. Explain the work

 When the agent performs multiple operations, I want to be able to understand what happened.

 For example:

```
I found three Pending pods.

The deployment requires a node with:
    no-such-node=true

No node currently satisfies that requirement.

I inspected the available nodes and found that the
control-plane node can satisfy the requirement after
adding the label.

I added the label and verified that all three pods
successfully transitioned to Running.
```

 The objective isn't just automation.

 It is **observable automation**.

---

 # Architecture

 At a high level:

```
              ┌─────────────────────┐
              │       Operator       │
              │                     │
              │ "Why is nginx down?"│
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │     kube-agent      │
              │                     │
              │  Intent / Reasoning │
              │  Tool Selection     │
              │  Execution          │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │       kubectl       │
              │                     │
              │ Kubernetes API      │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │ Kubernetes Cluster  │
              │                     │
              │ Pods                │
              │ Deployments         │
              │ Nodes               │
              │ Services            │
              │ Events              │
              └─────────────────────┘
```

 The LLM provides the reasoning layer.

 Kubernetes remains the system of record.

---

 # Getting started

 ## Prerequisites

 You'll need:

 - Python 3
- `kubectl`
- access to a Kubernetes cluster
- a configured Kubernetes context
- an OpenAI API key
- permission to perform the Kubernetes operations you ask the agent to perform

 Verify your Kubernetes access first:

```
kubectl cluster-info
kubectl get nodes
```

 Make sure the active context is the cluster you actually intend to operate:

```
kubectl config current-context
```

 > **Important:** `kube-agent` can execute Kubernetes operations. Treat the active kubeconfig context and RBAC permissions as production credentials.

---

 # Installation

 Clone the repository:

```
git clone https://github.com/dan9el8/kube-agent.git
cd kube-agent
```

 Create an isolated Python environment:

```
python -m venv venv
source venv/bin/activate
```

 Install dependencies:

```
pip install -r requirements.txt
```

 Configure the OpenAI API key:

```
export OPENAI_API_KEY="your-api-key"
```

 Verify Kubernetes connectivity:

```
kubectl cluster-info
kubectl get nodes
```

---

 # Usage

 Start the agent:

```
python main.py
```

 You should see something similar to:

```
☸️  kube-agent

Interactive Kubernetes AI assistant.
Type 'exit' to quit.

----------------------------------------------------

You:
```

 From here, Kubernetes operations can be expressed in natural language.

 For example:

```
show me the pods
```

```
what is unhealthy in the cluster?
```

```
why is my deployment failing?
```

```
show me the events for the nginx deployment
```

```
what nodes are available?
```

```
find pods that aren't running
```

---

 # Example: troubleshooting a scheduling failure

 To demonstrate the workflow, I use a disposable [kind](<https://kind.sigs.k8s.io/>) cluster.

 Create a cluster:

```
kind create cluster -n kube-agent
```

 Verify it:

```
kubectl cluster-info --context kind-kube-agent
```

 Now create a deliberately broken deployment:

```
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: some-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: some-app
  template:
    metadata:
      labels:
        app: some-app
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: no-such-node
                    operator: In
                    values:
                      - "true"
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.9
EOF
```

 Check the pods:

```
kubectl get pods
```

 The pods should remain `Pending` because no node satisfies the required affinity rule.

 Now ask `kube-agent`:

```
Why are the some-app pods pending?
```

 A useful answer should identify the actual scheduling constraint rather than simply reporting:

```
STATUS: Pending
```

 For example:

```
The pods cannot be scheduled because the Deployment
requires a node with:

    no-such-node=true

The current nodes do not have that label.

The Kubernetes scheduler therefore has no eligible
node for these pods.
```

 That distinction is important.

 **The status tells me what happened.\
 The diagnosis tells me why.**

---

 # Example: remediation

 Once the root cause is understood, I can ask:

```
Add the required label to the appropriate node.
```

 The agent can inspect the nodes, determine the relevant target, execute the Kubernetes operation, and verify the result.

 For example:

```
kubectl get nodes
```

 followed by:

```
kubectl label node <node> no-such-node=true
```

 Then verification:

```
kubectl get pods
```

 The desired result is:

```
some-app-...   1/1   Running
some-app-...   1/1   Running
some-app-...   1/1   Running
```

 The important part of the workflow is not simply executing the label command.

 It is:

```
Discover
   ↓
Diagnose
   ↓
Choose remediation
   ↓
Execute
   ↓
Observe
   ↓
Verify
```

---

 # Example: image failure

 Create another intentionally broken deployment:

```
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nnnnnnnnginx
EOF
```

 Now ask:

```
Why isn't nginx running?
```

 The investigation should identify the image failure:

```
ErrImagePull
ImagePullBackOff
```

 and correlate that with the configured image:

```
nnnnnnnnginx
```

 Rather than immediately modifying the deployment, ask:

```
Suggest safe options to fix it.
```

 Possible approaches include:

 - correct the image reference
- use an explicitly supported image version
- verify that a private registry is reachable
- verify registry authentication
- roll back if a previous deployment revision is known to be valid

 Then, if the intended remediation is clear:

```
Update nginx to nginx:latest and verify the rollout.
```

 The final step is verification:

```
Is the deployment healthy?
Are the pods Ready?
Did the rollout complete?
```

---

 # Production mindset

 `kube-agent` is intentionally more than a chatbot wrapped around `kubectl`.

 For real Kubernetes operations, the important engineering concerns are:

 ## Least privilege

 The Kubernetes identity used by the agent should have only the permissions required for the intended workload.

 Do not give an AI agent unrestricted `cluster-admin` access unless there is a deliberate reason to do so.

 Use Kubernetes RBAC to define the operational boundary.

 ## Explicit environments

 Before executing an operation, know which cluster and context are active:

```
kubectl config current-context
```

 A production agent should make cluster identity obvious before destructive or high-impact operations.

 ## Read before write

 Operational actions should be based on current cluster state rather than assumptions.

 ## Small blast radius

 Prefer:

```
one deployment
one namespace
one resource
one controlled change
```

 over broad cluster-wide mutations.

 ## Verification

 Every change should have a measurable expected result.

 For a Deployment:

```
desired replicas
    ↓
updated replicas
    ↓
available replicas
    ↓
ready pods
```

 For a Node:

```
node state
    ↓
labels / taints
    ↓
scheduler behavior
    ↓
workload placement
```

 For an image:

```
image reference
    ↓
pull
    ↓
container start
    ↓
readiness
```

---

 # Useful prompts

 ## Cluster health

```
Give me a health summary of the cluster.
```

```
Which workloads are unhealthy?
```

```
Are any nodes NotReady?
```

```
Show me pods that are Pending, Failed, or restarting.
```

 ## Troubleshooting

```
Why is this pod failing?
```

```
Why is this deployment not becoming Ready?
```

```
Investigate the ImagePullBackOff.
```

```
Find the root cause of the pending pods.
```

```
Check recent Kubernetes events for this workload.
```

 ## Remediation

```
Suggest ways to fix this problem before changing anything.
```

```
Fix the deployment and verify the rollout.
```

```
Make the smallest change required to resolve this issue.
```

```
Tell me exactly what kubectl commands you executed.
```

 ## Verification

```
Verify that the application is healthy now.
```

```
Show me the resulting pod status.
```

```
Did the rollout actually complete?
```

---

 # What I want kube-agent to become

 The long-term goal is a **production-grade Kubernetes AI operator**, not simply a natural-language wrapper around `kubectl`.

 That means building toward capabilities such as:

 - deterministic Kubernetes inspection
- structured diagnostics
- safe tool execution
- RBAC-aware operations
- explicit confirmation for high-impact changes
- dry-run support
- audit-friendly command history
- rollout verification
- event correlation
- workload dependency analysis
- security-aware operations
- namespace and cluster scoping
- observability integration
- GitOps-aware workflows
- incident investigation
- automated remediation with guardrails
- comprehensive tests against disposable Kubernetes clusters

 The agent should become increasingly capable without becoming increasingly reckless.

 **More automation should mean more guardrails, not fewer.**

---

 # Development philosophy

 I approach `kube-agent` as infrastructure software rather than a demo application.

 The standard I am aiming for is:

```
Correctness
    +
Safety
    +
Observability
    +
Testability
    +
Operational simplicity
```

 AI-generated decisions should ultimately be grounded in actual Kubernetes state.

 When the agent doesn't have enough evidence, it should investigate rather than guess.

 When multiple remediations are possible, it should explain the trade-offs.

 When it changes something, it should verify the outcome.

 When an operation is potentially dangerous, the workflow should make that risk visible.

---

 # Testing locally

 For experiments, I recommend using a disposable Kubernetes cluster such as [kind](<https://kind.sigs.k8s.io/>).

 Create one:

```
kind create cluster -n kube-agent
```

 Run the agent against it:

```
python main.py
```

 When finished:

```
kind delete cluster -n kube-agent
```

 This makes it possible to intentionally introduce failures without putting a production cluster at risk.

---

 # Upstream inspiration

 `kube-agent` was inspired by the Kubernetes AI experimentation demonstrated in k8s-ai.

 The original project demonstrated an important idea:

 > Kubernetes operations can be expressed naturally while an AI system translates intent into concrete cluster operations.

 `kube-agent` takes that idea and focuses it around a more production-oriented engineering direction: **safe operations, diagnosis, verification, and operational discipline.**

---

 # Security

 Treat `kube-agent` as an operational tool with access to your Kubernetes API.

 Before using it against a production cluster:

 1. Review the active kubeconfig context.
2. Use dedicated Kubernetes identities where possible.
3. Apply least-privilege RBAC.
4. Avoid unnecessary `cluster-admin` permissions.
5. Test changes against disposable clusters first.
6. Require confirmation for high-impact operations.
7. Keep an audit trail of agent actions.
8. Never expose Kubernetes credentials or secrets to the model unnecessarily.

 If the agent can change your cluster, the agent should be treated with the same care as any other production automation system.

---

 # Status

 🚧 **Active development**

 `kube-agent` is evolving toward a production-grade Kubernetes AI operations platform.

 The current implementation demonstrates the core workflow:

```
Natural language
      ↓
AI reasoning
      ↓
Kubernetes inspection
      ↓
kubectl operations
      ↓
Verification
      ↓
Human-readable explanation
```

 The engineering goal is to make every layer increasingly reliable, observable, testable, and safe.

---

 # Author

 Built and maintained by **dan9el8**.

 The project is intentionally opinionated about Kubernetes operations:

 > **Understand the cluster. Make the smallest useful change. Verify the result.**

---

 # License

 See the repository license for the current licensing terms.