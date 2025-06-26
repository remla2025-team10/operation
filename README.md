# Restaurant Sentiment Analysis Web Application

This application uses a machine learning model to classify given restaurant reviews based on their sentiment, being either positive (1) or negative (0).

## Docker Compose
To setup and run the application using Docker Compoase, do the following:
```bash
git clone https://github.com/remla2025-team10/operation
cd operation
docker-compose up
```

You can then access the application at [http://localhost:8080](http://localhost:8080). Here, you may enter a review into the textbox and query for a prediction.


### Additional startup configuration
There are also several environment variables which can be changed:
```bash
APP_IMAGE=ghcr.io/remla2025-team10/app-service:latest
MODEL_SERVICE_IMAGE=ghcr.io/remla2025-team10/model-service:latest
MODEL_SERVICE_PORT=3000
APP_PORT=8080
```

You can either change these manually, or when running `docker-compose up` like so:
```bash
 APP_PORT=5000 MODEL_SERVICE_PORT=3030 docker-compose up
```

## Kubernetes Cluster Setup

This repository includes configuration for deploying a Kubernetes cluster using Vagrant and Ansible. The setup consists of:
- 1 control node
    - named `ctrl`, runs on `192.168.56.100`
- 2 worker nodes by default
    - configurable, named `node-1` and `node-2`, run on `192.168.56.101` and `192.168.56.102`

The cluster uses Flannel for pod networking and includes Helm for package management.

We use SSH to access the VMs, so make sure to add your public SSH key (ends with `.pub`) to the `ansible/ssh_keys/` directory.

### Vagrant Setup

Make sure you have [Vagrant](https://www.vagrantup.com/) and [Ansible](https://www.ansible.com/) installed.

From the directory containing your `Vagrantfile`, run:
```bash
vagrant up
```

You can customize the Vagrant deployment using environment variables:

```bash
# Set the number of worker nodes (default is 2)
WORKER_COUNT=3 vagrant up
```

To apply the Ansible playbook, run:
```bash
vagrant provision
```

### Inventory config generation
Upon completion, an `inventory.cfg` file is generated, which details the controller node, and active worker node IP addresses. It has the following structure:
```
[ctrl] # controller node
ctrl=192.168.56.100

[nodes] # active worker nodes
node-1=192.168.56.101
node-2=192.168.56.102
```


### Accessing the Kubernetes Cluster

After successful provisioning, you can SSH into the control node and use kubectl:

```bash
# SSH into control node
vagrant ssh ctrl
```


## Cluster Finalization

To finalize the cluster, we will install MetalLB, Istio, and NGINX Ingress Controller.

### Finalize cluster setup
You can run the following command from the host to finalize the cluster setup:


```bash
ansible-playbook -u vagrant -i 192.168.56.100, ansible/finalize.yml
```

### Install application with Helm

First, ensure that the `KUBECONFIG` global variable is configured to `admin.conf`:

```bash
export KUBECONFIG=admin.conf
```

To enable **Alerts**, make sure to reconfigure the myprom installation:

```bash
helm upgrade myprom prom-repo/kube-prometheus-stack \
  --namespace monitoring  \
  -f prometheus-values.yaml
```

You can now install our Helm chart `model-stack` for the application stack as follows:

```bash
helm install myapp ./model-stack
```

#### Local DNS Resolution

On your host machine, make sure to add `app.local`, `dashboard.local`, `kiali.local`, `prometheus.local`, and `grafana.local` in your `/etc/hosts` file:


```bash
sudo nano /etc/hosts
```

Add the following line at the end:
```bash
192.168.56.90 dashboard.local
192.168.56.91 app.local kiali.local prometheus.local grafana.local
```

Save and exit (`Ctrl + O`, then `Enter`, then `Ctrl + X`)

Finally, flush the DNS cache.

For macOS, run:

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

For Linux, run:

```bash
sudo resolvectl flush-caches
```

#### The app is now ready!

When everything is complete, you can access the app at [app.local](app.local), the kiali dashboard should be accessible at [kiali.local](kiali.local), and the prometheus dashboard should be accessible at [prometheus.local](prometheus.local).


The grafana dashboard should be accessible at [grafana.local](grafana.local), and you can log in with username: `admin` and password: `prom-operator`.

The Kubernetes Dashboard should be accessible at [dashboard.local](dashboard.local). To log in, generate an admin token by running this command on the control node (`vagrant ssh ctrl`):
```bash
kubectl -n kubernetes-dashboard create token admin-user
```
Copy the output token and use it to log in to the dashboard.

#### Sticky sessions
After the above steps are complete, sticky sessions should be configured meaning that each subsequent request to `app.local` will lead to the same version.
Sticky sessions were configured to work only when the initial request contains the `x-user` header, in which case upon the initial 90/10 routing is determined, a cookie is set, indicating the version that user should be routed to. In that case, all subsequent requests will check for the respective cookie header and always route to that version.

If no `x-user` header is present, the requests won't be sticky and will always be 90/10 on subsequent requests.
You can best test this via Postman, as if you set the header `x-user` and make the request, you should see the cookies contain the app version afterwards, and reruning the request will always lead you to the same version of the app.

#### Rate-limiting

With istio installed, rate-limiting is implemented using `EnvoyFilter`. The rate limiting value can be configured in the `model-stack/values.yaml` file:

```
rateLimiting:
  enabled: true
  requestsPerMin: 250 # rate limit requests per minute.
```

To test it, go to your main terminal and try the following script.
```bash
for i in {1..250}; do curl -I -H "Host: app.local" app.local; sleep 0.1; done
```
 Once the local rate limit has been hit, you will receive `429` status codes. You can adjust the loop to match the rate-limit value you configured in the `values.yaml` file.

### System Requirements

The Kubernetes cluster has the following requirements:
- The default RAM requirement is 4GB for the control node and 6 GB for each worker node.
- The default CPU requirement is 2 CPUs for each node.
- You can change these in the `Vagrantfile`.

## Model stack Helm chart deployment

A Helm chart deploying 'model-service' and 'app-service' into a local Kubernetes cluster using **Minikube**.

### Requirements

- [Minikube](https://minikube.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/)

### Deployment

#### 1. Start a Minikube Cluster

```bash
minikube start
```

Enable the ingress addon:

```bash
minikube addons enable ingress
```

#### 2. Install Prometheus
Add prometheus repository to your helm repositories and update the it
```
helm repo add prom-repo https://prometheus-community.github.io/helm-charts
helm repo update
```
You can make sure it's installed with
```
helm repo list
```
Then install prometheus, you can give any prometheus release name but the default is <b>myprom</b>. If you change this be sure to also change the value of `serviceMonitor.additionalLabels.release` in `model-stack/values.yaml`.
```
helm install myprom prom-repo/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml
```

If you decide to later change something in the Prometheus stack you can update it with:

```
helm upgrade myprom prom-repo/kube-prometheus-stack \
  --namespace monitoring  \
  -f prometheus-values.yaml
```

#### 3. Install istio

Make sure that a current version of `istioctl` is installed and accessible on your machine, this example uses version 1.26.1. Then install istio in the `istio-system` namespace.

```
istioctl install
```

Then apply the preconfigured `prometheus`, `jaeger` and `kiali` files onto you minikube cluster. These files come in a parent folder that your `istioctl` installation came with. 

```
kubectl apply -f istio-1.26.1/samples/addons/prometheus.yaml
kubectl apply -f istio-1.26.1/samples/addons/jaeger.yaml
kubectl apply -f istio-1.26.1/samples/addons/kiali.yaml
```

Now make sure that istio can automatically inject a sidecar to each pod in the default namespace.

```
kubectl label namespace default istio-injection=enabled --overwrite
```
#### 3. Deployment
Manually create a `/mnt/shared` directory:
```bash
minikube ssh
sudo mkdir -p /mnt/shared
exit
```

Then set the environment variable
``` 
export KUBECONFIG=admin.conf
```

Then, install the helm release
```
helm install <release-name> ./model-stack-fresh
```

#### 4. Access application service
Open a minikube tunnel to expose minikube's istio-ingressgateway to the host, which functions as minikubes loadBalancer.

```
minikube tunnel
```

Now get the external IP address from istio-gateway:
```
kubectl get svc istio-ingressgateway -n istio-system
```

Now add these lines to your /etc/hosts file:

```
<istio-gateway-external-ip> app.local
<istio-gateway-external-ip> prometheus.local
<istio-gateway-external-ip> kiali.local
<istio-gateway-external-ip> grafana.local
```

You make sure to flush your DNS cache for previous entries for app.local and such so that these changes take place.

Linux (ubuntu >= 20.00)
```
sudo resolvectl flush-caches
```

You can now access these files in your browser form `app.local`, `grafana.local`, `prometheus.local` and `kiali.local`.

#### Access Prometheus and Grafana
If you have confirmed that the services for prometheus and grafana are running, you can access their dashboards with the following command (tip: launch new terminals for both services because the port-forwarding command is blocking):

Prometheus on localhost:9090
```
kubectl port-forward svc/myprom-kube-prometheus-sta-prometheus 9090:9090
```

Grafana on localhost:8001
```
kubectl port-forward svc/myprom-grafana 8001:80
```

Log in with username: `admin` and password: `prom-operator`

### 5. Alerts

To configure alerts to your email, you first need to create a secret for the sender gmail app key (for authentication). For testing purposes we provide you with the actual app key, but this will not be shared in the final version:
```bash
kubectl create secret generic smtp-password-secret \
  --from-literal=smtp_password='lmyl nlxo hjdc hpzn' \
  -n default
```

To set your own gmail address as the target to receive the alerts, you can do:

```bash
helm upgrade <release-name> ./model-stack \
 --set alert.receiverEmail=<your-email>
```

or alter the value of `alert.receiverEmail` in `./model-stack/values.yaml` and run:

```bash
helm upgrade <release-name> ./model-stack
```

## Relevant Files and Information
The application is structured in the following way:

**Operation Repository** is the starting point of the application and contains the files required to run the application, as explained above

- `Vagrantfile`: Defines the VM configurations.
- `ansible/`: Contains Ansible playbooks for automated setup and configuration of the Kubernetes cluster
    - `files/`:
        - `kube-flannel.yml`: KB pod network config file for deploying Flannel.
        - `metallb-native.yaml`: Config manifest for deploying MetalLB.
    - `kubernetes dashboard/`
        - `dashboard-adminuser.yaml`: Define ServiceAccount and CluterRoleBinding for a KB dashboard admin user.
        - `ingress-dashboard`: Ingress configuration for exposing the KB dashboard.
    - `general.yaml`: Configures all VMs with necessary prerequisites for Kubernetes, including required system settings.
    - `ctrl.yaml`: Initializes the Kubernetes cluster on the control node.
    - `node.yaml`: Configures worker nodes and joins them to the Kubernetes cluster.
    - `finalize.yml`: Installed MetalLB, NGinx Ingress, Dashboard, and Istio service mesh (+ addons) for load balancing and routing.
- `docs/`
    - `continuous-experimentation.md`: A/B testing experiment documentation.
    - `deployment.md`: Deployment description details.
    - `extension.md`: Extension proposal.
- `model-stack/`: Contains the Helm chart for deploying the application to Kubernetes
    - `Chart.yaml`: Metadata about the Helm chart
    - `values.yaml`: Default configuration values for the Helm chart
    - `templates/`
        - `_helpers.tpl`: Helper templates for the Helm chart
        - `alertmanager-config.yaml`: AlertManager configuration for notification settings.
        - `app-deployment.yaml`: Creates two deployments for app-service. One for stable (v1) and one for canary (v2).
        - `destinationrules.yaml`: Defines Istio DestinationRules for model-service and app-service.
        - `istio-rate-limiting.yaml`: EnvoyFilter to apply local rate limiting.
        - `isitiogateway.yaml`: Istio Gateway resource to expose HTTP traffic to the cluster for external access.
        - `model-deployment.yaml`: Creates two deployments for model-service. One for stable (v1) and one for canary (v2).
        - `model-pv.yml`: PersistentVolume for storing model files. 
        - `model-pvc.yml`: PersistentVolumeClaim for storing model files.
        - `requests-rule.yaml`: Custom Prometheus alerting rule.
        - `servicemonitor.yaml`: ServiceMonitor for prometheus to scrape metrics from app-service.
        - `services.yaml`: KB services for model-service and app-service.
        - `virtualservices.yaml`: Istio VirtualServices for traffic management.
        - `monitoring/`: directory with ConfigMaps for Grafana dashboards.
            - `a3-dashboard-configmap`
            - `ab-test-dashboard-configmap`
- `docker-compose.yml`: docker-compose configuration file

## Repository Links
For convenience, we list links to the available repositories used in this project:
- [operation](https://github.com/remla2025-team10/operation)
- [app-service](https://github.com/remla2025-team10/app-service)
- [model-service](https://github.com/remla2025-team10/model-service)
- [model-training](https://github.com/remla2025-team10/model-training)
- [lib-version](https://github.com/remla2025-team10/lib-version)
- [lib-ml](https://github.com/remla2025-team10/lib-ml)