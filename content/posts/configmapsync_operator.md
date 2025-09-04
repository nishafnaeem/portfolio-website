+++
title = 'Building My First Kubernetes Operator 🚀'
date = 2025-09-03T07:07:07+01:00
draft = false
disableComments = true
+++

---

## From Using Operators to Understanding Them

After five years of working with Kubernetes, I'd used plenty of operators – Datadog Operator, NGINX Operator, Prometheus Operator – you name it. But there was always this black box feeling. I'd deploy these powerful tools, configure them through YAML, and watch them magically manage complex applications. But how did they actually work?

As someone who primarily worked with Python, the Go-based operator ecosystem felt like a foreign language. Every time I tried to peek under the hood of an operator to understand its logic or troubleshoot an issue, I was met with Go code that might as well have been hieroglyphics.

Recently, I decided to change that. I started learning Go, and after getting comfortable with the basics, I thought: *"Why not build a Kubernetes operator as a hands-on learning project?"*

## The Weekend Challenge: ConfigMapSync Operator

I gave myself a weekend to build something useful but manageable. The idea was simple: create an operator that automatically syncs ConfigMaps between different namespaces. 

### Why ConfigMapSync?

In many Kubernetes environments, you often need to share configuration data across namespaces. Maybe you have:
- A central configuration that multiple applications need
- Environment-specific configs that should be propagated to application namespaces
- Shared secrets or settings that need to be available cluster-wide

Sure, there are workarounds like init containers or Helm hooks, but an operator provides a more Kubernetes-native, declarative approach.

## The Learning Curve: What I Discovered

### Challenge #1: The Reconciliation Loop

Coming from imperative Python scripts, the concept of a reconciliation loop was mind-bending at first. Instead of "do this, then do that," operators work on the principle of "ensure the desired state matches reality."

```go
// This was my "aha!" moment - the reconciliation loop
func (r *ConfigMapSyncReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Get the current state
    // 2. Compare with desired state  
    // 3. Take action to reconcile differences
    // 4. Return (and Kubernetes will call us again later)
}
```

The beauty is that Kubernetes handles all the complexity of when to call your reconciler, error handling, retries, and more.

### Challenge #2: Go Language Patterns

As a Python developer, Go's explicit error handling and struct-based programming initially felt verbose. But working on this operator helped me appreciate Go's clarity:

```go
// Python brain: "Where are all the try/catch blocks?"
// Go brain: "Every error is explicit and handled"
if err != nil {
    log.Error(err, "Failed to get source ConfigMap")
    return ctrl.Result{}, err
}
```

### Challenge #3: RBAC Marker Annotations

This was the cherry on top that I never expected! Instead of manually crafting RBAC YAML files, Operator SDK uses marker annotations:

```go
//+kubebuilder:rbac:groups=apps.nishaf.info,resources=configmapsyncs,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups="",resources=configmaps,verbs=get;list;watch;create;update;patch;delete
```

These comments automatically generate the necessary Kubernetes RBAC permissions. It's like magic, but better – it keeps permissions close to the code that needs them.

## The Build Process: Creating a Working Operator

### Setting Up the Foundation

I used the Operator SDK to scaffold the project:

```bash
operator-sdk init --domain=example.info --repo=github.com/nishafnaeem/configmapsync-operator
operator-sdk create api --group=apps --version=v1 --kind=ConfigMapSync --resource --controller
```

This generated the basic structure, but the real work was implementing the business logic.

### Defining the API

The first step was defining what a ConfigMapSync should look like:

```go
type ConfigMapSyncSpec struct {
    SourceNamespace      string `json:"sourceNamespace"`
    DestinationNamespace string `json:"destinationNamespace"`
    ConfigMapName        string `json:"configMapName"`
}
```

Simple, but effective. Users can now declare their intent with a resource like:

```yaml
apiVersion: apps.nishaf.info/v1
kind: ConfigMapSync
metadata:
  name: my-config-sync
spec:
  sourceNamespace: config-central
  destinationNamespace: my-app-namespace
  configMapName: shared-config
```

### Implementing the Controller Logic

The reconciliation logic follows a clear pattern:

1. **Fetch the ConfigMapSync resource** - what does the user want?
2. **Get the source ConfigMap** - what needs to be synced?
3. **Check if destination ConfigMap exists** - what's the current state?
4. **Create or update the destination** - make reality match the desired state
5. **Update status** - let users know what happened

### Testing: The Real Learning Experience

Writing tests for the operator taught me more about Kubernetes internals than any documentation could. Using the `envtest` framework, I could run tests against a real (but lightweight) Kubernetes API server.

The most challenging part was managing test isolation – ensuring that each test had its own namespaces and resources to avoid conflicts.

## Deployment and Real-World Testing

After getting tests to pass locally, I deployed the operator to a GKE staging cluster:

```bash
# Build and push the container image
make docker-build docker-push IMG=my-registry/configmapsync:latest

# Deploy to Kubernetes
make install
make deploy IMG=my-registry/configmapsync:latest
```

Watching it work in a real cluster for the first time was incredibly satisfying. Creating a ConfigMapSync resource and seeing it automatically create and sync ConfigMaps across namespaces felt like magic.

## What I Learned Beyond Code

### 1. Operators Are Just Controllers with Opinions

At their core, operators are controllers that encode human operational knowledge. They watch for changes and react to maintain desired state – just like how a human operator would, but automated and consistent.

### 2. The Kubernetes API is Incredibly Powerful

Building an operator gave me a much deeper appreciation for Kubernetes' extensibility. The ability to define custom resources and have them feel like first-class citizens in the cluster is remarkable.

### 3. Go Makes Sense for Infrastructure Code

While I still love Python for many use cases, Go's explicit error handling, strong typing, and excellent Kubernetes client libraries make it a natural fit for infrastructure tooling.

### 4. Testing Infrastructure Code is Different

Unlike testing business logic, testing operators means thinking about state, eventual consistency, and distributed systems. It's challenging but rewarding.

## The Result: A Weekend Well Spent

In just one weekend, I went from "I wish I understood how operators work" to having a fully functional operator deployed in a staging environment. More importantly, I gained confidence in Go programming and a much deeper understanding of Kubernetes internals.

The ConfigMapSync operator is simple, but it's real and it works. It automatically keeps ConfigMaps synchronized across namespaces, handles updates to source ConfigMaps, and provides clear feedback through status updates.

## Try It Yourself

If you're curious about operator development or want to learn Go through a practical project, I'd encourage you to try building your own operator. The learning curve is steep initially, but incredibly rewarding.

You can find the complete source code, deployment instructions, and examples in my [ConfigMapSync Operator repository](https://github.com/nishafnaeem/configmapsync-operator). The README includes step-by-step instructions for building, testing, and deploying the operator.

## What's Next?

This weekend project opened up a whole new world for me. I'm already thinking about more complex operators I could build:
- An automatic backup operator for specific workloads
- A custom scaling operator based on business metrics

The possibilities are endless when you can extend Kubernetes with your own operational knowledge.

---

*Have you built any operators or are you thinking about starting? I'd love to hear about your experiences! Feel free to reach out or check out the code on GitHub.*

## Resources for Getting Started

- [Operator SDK Documentation](https://sdk.operatorframework.io/)
- [Kubebuilder Book](https://book.kubebuilder.io/)
- [A Tour of Go](https://tour.golang.org/) - Great for Python developers learning Go

---

*Tags: Kubernetes, Operators, Go, Learning, DevOps, Infrastructure, Weekend Project*
