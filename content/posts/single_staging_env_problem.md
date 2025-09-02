+++
title = 'Scaling QA: How We Solved the Single Staging Environment Problem 🚀'
date = 2025-08-25T07:07:07+01:00
draft = false
disableComments = true
+++

You’re a developer at a fast-paced startup. Every day you have to build and ship new features. The QA team is ready, the product is in sight, but there’s one major roadblock: the staging environment that we have is only one. Suddenly, you see yourself blocked because of not having parallel staging environments. This slows down your development and, to be honest, sometimes breaks that adrenaline rush to get things done on time.

This was our reality, and it’s a common problem for any company trying to move fast with a single, shared staging environment.

-----

### The Main Problem

In a remote and expanding team, a single staging environment quickly becomes a point of contention. One team deploys their feature, and the QA process begins. Meanwhile, another team finishes their feature and is left waiting.

The cost of this problem was steep:

  - **⏳ Delayed QA and Release Cycles:** Features are tested sequentially, not in parallel, slowing down the time it takes to get them to production.
  - **⏱️ Wasted Developer Time:** Engineers spend valuable hours coordinating with other teams, waiting for their turn, or dealing with unexpected environment issues.

We knew there had to be a better way to enable our team to move at the speed we needed to.

-----

### The Solution: A Personalized Portable Environment

Our solution was simple in concept. The idea was to give every developer and every feature their own isolated, personal staging environment. The core of our solution is a collection of bash scripts that can spin up a complete, working environment, from databases and pub/sub emulators to every application service in minutes. Teams were now able to test their features in isolation, without interfering with others, leading to a huge increase in our features development.

-----

### ✨ The Magic: A Customizable Command Center

The main configuration file behind this is the `branch_names.json` file. It’s the core of our system. You would have to define the `commit_hash` or the `branch_name` for each repository and that is used as a source of truth for what will be deployed in that cluster.

This means a developer can create a personal staging environment that pulls the latest version of their feature branch while still using the main branch for all other dependent services. This flexibility makes parallel testing and debugging simple. Our scripts handle everything else:

  - They intelligently fetch `.env` files from remote repositories.
  - They grab the correct container images from Google Artifact Registry (GAR) based on the specified commit hash. If the image is not present they will call the build pipeline for that specific repository to start building the image in parallel for that service.
  - They provision the entire cluster, from databases, pubsub-emulator to every service, just by using a single command.

-----

### ❤️ The Infrastructure

The heart of our solution is a **k3d cluster** that is provisioned directly on a GCP VM. This approach combines the lightweight, local development benefits of k3d with the power and accessibility of a cloud machine. K3d Cluster by default uses traefik Reverse Proxy that is already embedded with it. So I am also using the traefik to manage the Ingress for each service. By using a GCP VM, we could create an environment that was portable and accessible to our entire remote team. The `branch_names.json` file serves as the single source of truth, dictating which services and branches get deployed to the cluster.

The PS Environment provisioning system works as a comprehensive orchestration framework that automates the complete lifecycle of microservices deployment across development and staging environments. Starting with configuration initialization, the system loads environment variables, service configurations, and branch information from JSON files while handling authentication tokens specific to each environment (GitHub for development, cloud secrets for staging). The command-line interface provides multiple operational modes including cluster setup, new deployments, updates, and cleanup operations. For cluster provisioning, the system creates a K3D cluster with local registry for development or configures cloud resources with TLS certificates and Datadog monitoring for staging environments. The service deployment pipeline follows a systematic approach: first retrieving git commit hashes for version tracking, then building Docker images with commit-based tags, pushing them to appropriate registries, executing 1Password integration to generate secure environment files, and finally deploying services using Helm charts with proper configuration.

The infrastructure components include CloudSQLProxy, PubSub emulator for message queuing, Redis for caching, Traefik ingress controller for load balancing with custom middlewares, and Cloudflare integration for DNS management, creating a complete, production-like environment that can be provisioned, updated, or torn down with single commands while maintaining consistency across different deployment targets.

-----

### 🛠️ How It Works in Practice

The beauty of this solution lies in its simplicity. We provided two primary methods for managing these environments, both of which use the `branch_names.json` file as input:

1.  **The Command-Line Tool:** For local development and quick testing, our command-line tool allows developers to provision, update, and delete their own environments. You can get a full local environment up and running with a simple command:

    To start from scratch, you run one command:

    ```bash
    ./provision_ps_environment.sh --provision-environment --instance-name=<instance_name> --branch-names-json='{}'
    ```

    Here’s the Overview of the Command line tool.

    ![Target](/images/single_staging_env_problem_1.png)


2.  **The GitHub Action Workflow:** For a more automated and scalable solution, we integrated our scripts with a GitHub Action Workflow. This allowed us to provision a complete personalized staging environment on a GCP VM with a single command from the terminal, making it seamless for anyone on the team to get a clean, isolated testing environment on demand.

    ![Target](/images/single_staging_env_problem_2.png)

-----

### 🎉 The Impact: A Transformed Workflow

The shift to personalized staging environments had a **positive impact on our team’s productivity and morale**. QA teams were no longer waiting in a queue; they were testing multiple features simultaneously. By providing developers with a stable, isolated environment on demand, we empowered them to work more efficiently and confidently. What was once a painful, time-consuming process became a streamlined, automated workflow.

For any startup looking to scale their engineering and release cycles, automating your staging environments isn’t a luxury—it’s a necessity.