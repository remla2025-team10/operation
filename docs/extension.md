# Extension Proposal: Automate GitHub Page Helm Charts with Chart Releaser Action
## Identified Shortcoming

Currently, our project setup requires a series of manual commands and environment-specific procedures to get running. This involves manually executing each step in the Kubernetes provisioning and Helm chart deployment process, as indicated in this repository’s README file.

More specifically, the deployment of the `model-stack` Helm chart and related Kubernetes resources are a manual process. In other words, after the Kubernetes cluster is provisioned, one must manually run `helm install` to deploy the application. Any changes made to the chart requires a manual intervention (e.g. with `helm upgrade`).

This approach has several shortcomings such as:
- **No automation:** Changes to the helm charts do not automatically trigger deployments. 
- **No versioning or packaging:** There is no clear-cut versioning or packaging process. This makes rollbacks and reproducability more difficult.
- **Human error:** Manual deployment processes are prone to human error, especially when more contributors are involved. This could lead to configuration or hard to debug issues.
- **Limited traceability:** Given no CI/CD pipelines, there are no audit logs that record what has deployed, by whom, and when.

## Proposed Extension
A proposed solution is to use a **Chart Release Action** in a dedicated `helm-charts` repository within the organization, to automate releasing Helm charts with GitHub pages.

### Design Overview

Firstly, we shall define the dedicated Git repository under out organization, named `helm-charts`. This repo will manage the Helm charts for our application. The sources of all the charts will be placed under the main branch, while the charts themselves would be placed under a `/charts` directory at the top-level of the directory tree. On the main branch, we create a GitHub Actions workflow file: (`.github/workflows/release.yml`), which will use the `@helm/chart-releaser-action` configuration to turn our project into a self-hosted Helm chart repository.

```
helm-charts/
├── charts/
│   ├── model-stack/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/<...>
│   └── <other-charts>/
├── .github/
│   └── workflows/
│       └── release.yaml
└── README.md
```

Another branch named `gh-pages` will be included, which will be used to publish the charts. All changes to that branch will be automatically managed by the Chart Releaser Action (no user intervention required). This branch will host the Helm repository which will be used as the source for GitHub Pages, enabling Helm clients (e.g. for our KB cluster) to fetch the charts via:

```bash
helm repo add remla-2025-team10 https://remla-2025-team10.github.io/helm-charts
```

We could therefore configure the `operation` repository to automatically (or manually) pull the latest chart version with `helm repo update`.

Additionally, with GitOps tools like ArgoCD or Flux, it is possible to automatically update the KB cluster upon new helm chart releases (in an even further extension). 

### Benefits
In this repository, we could manage, version, and automatically release Helm charts feasibly with CI/CD pipelines via workflows (e.g. via GitHub Actions). This includes the following benefits:
- **Modular Testing:** Testing the helm charts on every PR (e.g. Linting) by further configuring the workflow pipelines.
- **Packaging and Versioning:** Using semantic versioning and git tags to manage releases.
- **Rollback:** Enable easy rollback to previous charts using Helm.
- **Traceability:** Record deployments and authors via audit logs.



## Supporting References

* [Chart Releaser Action to Automate GitHub Page Charts](https://helm.sh/docs/howto/chart_releaser_action/)
* [Chart Releaser Action (GitHub repo)](https://github.com/helm/chart-releaser-action)
* [Upgrading new version helm charts](https://www.ibm.com/docs/en/netcoolomnibus/8.0.0?topic=private-upgrading-new-version-helm-chart)
* [Auto-update helm chart versions with ArgoCD](https://medium.com/@eduard.mihai.lemnaru/auto-update-helm-chart-version-using-argocd-4936933a2bac)

## Experiment to Validate the Change

We believe that introducing a dedicated repository for managing Helm charts will reduce errors and time taken, and be positively received by the developers. The experiment will involve performing manual Helm deployments over several releases (N release iterations) between two conditions: baseline vs automated.

### Metrics
The following metrics will be collected between two conditions:
* Time (minutes) taken for deployment.
* Number of manual commands executed.
* Number of errors (error rate).
* Qualitative feedback from. contributors.

### Method:

We can compare the baseline vs the automated conditions.

1. **Baseline condition:** 
    - Perform N release iterations.
    - Record quantitative and qualitative metrics.
2. **Automated condition:** 
    - Migrate charts to the `helm-charts` repo.
    - Use GitHub Actions and Chart Releaser Actions, as indicated in the design overview.
    - Configure the `operation` repository to automatically pull and deploy new versions.
    - Perform N release iterations using the automated workflow.
    - Record quantitative and qualitative metrics.
3. **Analysis**: Compare the metrics between the conditions (average deployment time, command count, error rate). Gather feedback from contributors on perceived effort/efficiency. Document differences in a table and visualize improvement (e.g., bar chart)

### Success criteria (Expected Outcome):
- Reduced deployment errors and time (~30% reduction).
- Fewer manual steps involved (~50% reduction).
- Significant drop in deployment errors.
- Significant (positive) difference in contributor feedback scores.

