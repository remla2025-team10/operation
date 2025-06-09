# Extension Proposal: Automating Cross-Platform Setup for Reproducible Environments

## Identified Shortcoming

Currently, our project setup requires a **series of manual commands** and environment-specific procedures to get running. Setting up the system on our custom Vagrant-based Kubernetes cluster with Ansible differs significantly from doing so on Minikube, and would again vary for future cloud platforms (e.g., GKE, EKS). These manual steps include:

* Manually installing `istioctl` and `prometheus` on the minikube cluster
* Setting up port forwarding or tunnels (e.g., `minikube tunnel`)
* Enabling Istio addons manually
* Managing secrets and environment variables ad hoc

This fragmentation is **error-prone**, **time-consuming**, and slows down **CI/CD integration** and **platform portability**.

## Proposed Extension

We propose introducing a **uniform automation layer** using Helm + Helmfile and optionally a Makefile or simple CLI wrapper (e.g., Bash or Python) to standardize setup across all platforms.

### Design Overview

* **Helmfile** to declaratively install Istio components and our app's Helm charts across any Kubernetes target.
* **Profiles** or environment variable injection to switch between Minikube, Vagrant, or Cloud configurations.
* Optional **Makefile** or `./run.sh` script for a single-command install:

  ```bash
  make dev     # for Minikube
  make vagrant # for local Ansible cluster
  make cloud   # for GKE or other cloud
  ```

### Benefits

* **Consistency** across platforms
* **Improved developer experience**
* **Easier onboarding**
* **Foundation for GitOps/CI/CD workflows**

### Visual Overview

```text
+------------+     +--------------+     +-------------+
| Minikube   | --> | helmfile.yaml| --> | Istio/App   |
| Vagrant    |     | profiles     |     | Deployments |
| Cloud      |     +--------------+     +-------------+
```

## Supporting References

* [Helmfile: Deployment as Code](https://github.com/helmfile/helmfile)
* [GitOps with Helmfile](https://ayham-hassan.medium.com/mastering-kubernetes-deployments-with-gitops-helm-helmfile-a-beginners-guide-part-1-ac7fc3af37cb)
* [Automating Kubernetes with Makefiles](https://github.com/davidbanham/kube_maker)

## Experiment to Validate the Change

To evaluate this extension, we propose an experiment:

**Hypothesis**: Introducing a unified setup layer will reduce the setup time and error rate when onboarding a new environment.

### Metrics:

* Time (minutes) to go from clean environment to running app
* Number of manual commands executed
* Number of failed setup attempts per platform

### Method:

1. Ask team members to set up the project from scratch on Minikube and Vagrant:

   * Before automation: log time/commands/errors
   * After automation: repeat and compare

2. Document differences in a table and visualize improvement (e.g., bar chart)

### Expected Outcome:

Significantly fewer steps, reduced setup time, and fewer manual errors. Provides a strong case for platform-agnostic deployment standardization.

