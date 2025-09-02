+++
title = "Deploying and Configuring Datadog in Kubernetes with the Datadog Operator"
date = 2025-08-31T05:00:00+05:00
draft = false
disableComments = true
+++


This guide will walk you through deploying the Datadog Agent in your Kubernetes cluster using the Datadog Operator and Helm. We'll cover everything from the initial setup to configuring specific features like APM, log collection, and horizontal pod autoscaling (HPA) with Datadog metrics.

## Step 1: Install the Datadog Operator

The first step is to add the Datadog Helm repository and install the Datadog Operator. The Operator simplifies the management of the Datadog Agent, ensuring it's always configured correctly across your cluster.

1.  Add the Datadog Helm repository to your local Helm configuration:

    ```bash
    helm repo add datadog https://helm.datadoghq.com
    ```

2.  Install the Datadog Operator using Helm. We'll use version `7.69.0`, which corresponds to the Datadog Agent version.

    ```bash
    helm install datadog-operator datadog/datadog-operator --version 7.69.0
    ```

    If you need to upgrade the operator later, you can use the same command with the new version number.

## Step 2: Create Your Datadog Secret

The Datadog Agent needs your API and application keys to send data to your Datadog account. For security, we'll store these in a Kubernetes secret. Replace `<DATADOG_API_KEY>` and `<DATADOG_APP_KEY>` with your actual keys.

```bash
kubectl create secret generic datadog-secret --from-literal api-key=<DATADOG_API_KEY> --from-literal app-key=<DATADOG_APP_KEY>
```

## Step 3: Configure and Deploy the Datadog Agent

Now, we'll create the `datadog-agent.yml` file, which is a custom resource definition (CRD) that tells the Datadog Operator how to configure the Datadog Agent. This single file controls the deployment of the Datadog Agent and Cluster Agent and enables various features.

Here is the YAML content for the `DatadogAgent` resource:

```yaml
kind: DatadogAgent
apiVersion: datadoghq.com/v2alpha1
metadata:
  name: datadog
spec:
  global:
    logLevel: "info"
    site: datadoghq.com
    kubelet:
      tlsVerify: false
    credentials:
      apiSecret:
        secretName: datadog-secret
        keyName: api-key
      appSecret:
        secretName: datadog-secret
        keyName: app-key
  override:
    clusterAgent:
      image:
        name: gcr.io/datadoghq/cluster-agent:latest
    nodeAgent:
      image:
        name: gcr.io/datadoghq/agent:latest
      tolerations:
       - operator: "Exists"
  features:
    liveContainerCollection:
      enabled: true
    apm:
      enabled: true
      hostPortConfig:
        enabled: true
        hostPort: 8126
    logCollection:
      enabled: true
      containerCollectAll: true
    dogstatsd:
      hostPortConfig:
        enabled: true
        hostPort: 8125
    externalMetricsServer:
      enabled: true
```

### Key Features Enabled:

  * **`liveContainerCollection`**: Provides real-time visibility into all containers running in your environment.
  * **`apm` (Application Performance Monitoring)**: Enables the collection of traces from your applications. We also specifically bind the APM `HostPort` (`8126`) so that pods can send traces to the Agent.
  * **`logCollection`**: This feature allows the Agent to collect logs from all containers, giving you centralized log management.
  * **`dogstatsd`**: Enables the collection of custom metrics from your applications. The `HostPort` is enabled and bound to `8125`, allowing any process on the node to send metrics to the Datadog Agent. This is a crucial configuration since the container port isn't bound to the `HostPort` by default.
  * **`externalMetricsServer`**: This is a key feature for enabling **Horizontal Pod Autoscaling (HPA)** based on Datadog metrics. When enabled, it allows the HPA to query custom metrics from Datadog to make scaling decisions.

After creating the file, apply it to your cluster to start the deployment:

```bash
kubectl apply -f datadog-agent.yml
```

## Step 4: Configure Your Application for Tracing

For your application pods to send traces to the Datadog Agent, you need to set the `DD_AGENT_HOST` environment variable. This variable tells the Datadog client library where to find the Agent.

Add the following to your application's deployment YAML:

```yaml
env:
  - name: DD_AGENT_HOST
    valueFrom:
      fieldRef:
        fieldPath: status.hostIP
```

This configuration dynamically sets the host IP of the node where the pod is running, allowing it to connect to the Datadog Agent.


# Conclusion

By following these steps, you have successfully deployed the Datadog Agent in your Kubernetes cluster using the Datadog Operator. This configuration enables a wide range of powerful observability features, including real-time container monitoring, application performance tracing, comprehensive log collection, and dynamic autoscaling with custom metrics. With a single, declarative YAML file, you can manage the entire Datadog Agent lifecycle and ensure your applications and infrastructure are fully observable.


# More Docs/References
- [Sample Python Observability App](https://github.com/nishafnaeem/datadog-observability-app): To test your new Datadog setup, you can deploy a sample observability application. This repository contains all the necessary code, including a Python app, Dockerfile, and Kubernetes manifests with a detailed README. 
- [Autoscale by number of requests/s in HPA](../datadog_autoscaling_hpa)
