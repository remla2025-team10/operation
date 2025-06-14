![Deployment Architecture Diagram](REMLA10_diagram.png)

# Deployment Structure
In this document we outline the architecture of this project and its deployment. We elaborate on the flow and configuration behind the setup to make it easier to understand the project on a high-level.

The project deployment process consists of the following:
- A Kubernetes cluster provisioned with Vagrant and Ansible
- An application consisting of multiple services that is deployed with Helm
- Ingress management with MetalLB and an ingress controller
- Prometheus monitoring and a simple alerting system
- Local environment possibility with Docker Compose

## Kubernetes Cluster and Deployment Structure
The Kubernetes cluster is first provisioned using Vagrant and Ansible. This results in the creation of a control node, which acts as the master node that orchestrates and manages everything, and a user specified number of worker nodes, each running the application workloads (pods).

This is followed by Ansible playbook provisioning which further configures the project setup and ensures the nodes are joined in a cluster.

When this is done, the cluster utilizes Flannel CNI for pod networking, MetalLB for load balancing and Ingress Controller for handling external traffic into the cluster.

The applications are deployed using Helm Charts (model-stack), where an Ingress Rule routes HTTP traffic from MetalLB into the cluster, app-service service exposes the app-service pod which hosts the main application logic and further forwards HTTP requests to the model-service. Model-service Service is a ClusterIP service, meaning it is only accessible internally (by app-service), and its pod hosts the logic for interacting with the machine learning model used for sentiment analysis and its training.

## Usage Flow
The user begins their interaction with the application by making an HTTP request. This request is then handled by the ingress controller to forward the request as defined by the rule. App service receives the request, processes it and internally makes a request to model-service as needed. The model-service then responds with the processed result, which the app-service returns back to the user through the ingress path.

In case of using the Docker Compose setup, the user makes requests via configurable ports for the app/model service to achieve the same behaviour.

## Monitoring and Alerting
Monitoring is done using Prometheus. Namely, it utilizes the following:
- ServiceMonitor which collects the metrics from the application
- Prometheus aggregates and stores the metrics
- Grafana: Allows for visualization of the metrics in real time via a dashboard
- AlertManager: Sends alerts via email when specific events occur (in our case a high request rate within 2 minutes)

The metrics themselves are collected from the configured pods and services, and are then used by Prometheus for analysis.

## Configurability
As mentioned, the setup allows for a lot of configurability when running the deployment. This includes port mappings, image tags, paths, and environment variables.

## Experimental Setup and Dynamic Routing
The project uses dynamic routing via the Ingress Controller which allows for A/B testing between user groups for testing new app features / versions. Sticky sessions via HTTP headers make it so the experimental setup is consistent among users, by assigning the corresponding experiment header to the chosen users.