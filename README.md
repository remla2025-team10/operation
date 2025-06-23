# Restaurant Sentiment Analysis Web Application

This application uses a machine learning model to classify given restaurant reviews based on their sentiment, being either positive (1) or negative (0).

## Starting the Application
To setup and run the application, do the following:
```bash
git clone https://github.com/remla2025-team10/operation
cd operation
docker-compose up
```

You can then access the application at [http://localhost:8080](http://localhost:8080). Here, you may enter a review into the textbox and query for a prediction.


### Additional startup configuration
There are also several environment variables which can be changed:
```bash
APP_IMAGE=ghcr.io/remla2025-team10/app-service:v0.4.11
MODEL_SERVICE_IMAGE=ghcr.io/remla2025-team10/model-service:v0.0.3
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
ansible-playbook -u vagrant -i 192.168.56.100, finalize.yml
```

### Install application with Helm
After the cluster setup is done, you can install our Helm chart for the application stack as follows:

```bash
export KUBECONFIG=admin.conf
helm install myapp ./model-stack-fresh
```

#### Sticky sessions
After the above steps are complete, sticky sessions should be configured meaning that each subsequent request to `app.local` will lead to the same version.
Sticky sessions were configured to work only when the initial request contains the `x-user` header, in which case upon the initial 90/10 routing is determined, a cookie is set, indicating the version that user should be routed to. In that case, all subsequent requests will check for the respective cookie header and always route to that version.

If no `x-user` header is present, the requests won't be sticky and will always be 90/10 on subsequent requests.
You can best test this via Postman, as if you set the header `x-user` and make the request, you should see the cookies contain the app version afterwards, and reruning the request will always lead you to the same version of the app.

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
sudo systemd-resolve --flush-caches
```

When everything is complete, the Kubernetes Dashboard should be accessible at [dashboard.local](dashboard.local), our app should be accessible at [app.local](app.local), the kiali dashboard should be accessible at [kiali.local](kiali.local), the grafana dashboard should be accessible at [grafana.local](grafana.local), and the prometheus dashboard should be accessible at [prometheus.local](prometheus.local).

To log in, generate an admin token by running this command on the control node:
```bash
kubectl -n kubernetes-dashboard create token admin-user
```
Copy the output token and use it to log in to the dashboard.

#### Rate-limiting

With istio installed, you can apply rate-limiting to the app.

First, SSH into the control node:
```bash
vagrant ssh ctrl
```

Then apply the filter:
```bash
kubectl apply -f /vagrant/rate-limiting.yaml
```

(You can also delete the filter from the control node with ```kubectl delete -f /vagrant/rate-limiting.yaml```)

To test it, go to your main terminal and try the following script.
```bash
for i in {1..20}; do   curl -I -H "Host: app.local" http://192.168.56.90/;   sleep 1; done
```
 After ten `200 OK` requests, you will receive `429` status codes.


### System Requirements

The Kubernetes cluster requires:
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
  --create-namespace
```

#### 3. Deployment
Manually create a `/mnt/shared` directory:
```bash
minikube ssh
sudo mkdir -p /mnt/shared
exit
```

Then navigate to the model-stack directory (which stores the helm chart)
```bash
cd operation/model-stack
```
Then, install the helm release
```
helm install <release-name> .
```

#### 4. Port forward

Run the following to port forward the app service:
```
kubectl port-forward svc/<release-name>-app-service 8080:8080
```

Now you can access the app via `http://localhost:8080/`

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

Log in wiht username: `admin` and password: `prom-operator`

### 5. Alerts

To configure alerts to your email, you first need to create a secret for the sender gmail app key (for authentication). For testing purposes we provide you with the actual app key, but this will not be shared in the final version:
```bash
kubectl create secret generic smtp-password-secret \
  --from-literal=smtp_password='lmyl nlxo hjdc hpzn' \
  -n default
```

To set your own gmail address as the target to receive the alerts, you can do:

```bash
helm upgrade <release-name> . \
  --set alertmanager.receiverEmail=<your-email>
```

## Relevant Files and Information
The application is structured in the following way:
- **Operation Repository** is the starting point of the application and contains the files required to run the application, as explained above
    - `Vagrantfile`: Defines the virtual machines configuration.
    - `ansible/`: Contains Ansible playbooks for automated setup and configuration of the Kubernetes cluster
        - `general.yaml`: Configures all VMs with necessary prerequisites for Kubernetes, including required system settings.
        - `ctrl.yaml`: Initializes the Kubernetes cluster on the control node.
        - `node.yaml`: Configures worker nodes and joins them to the Kubernetes cluster.
        - `finalization.yaml`: Installed MetalLB, NGinx Ingress, Dashboard, (and Istio) for load balancing and routing
    - `model-stack/`: Contains the Helm chart for deploying the application to Kubernetes
        - `Chart.yaml`: Metadata about the Helm chart
        - `values.yaml`: Default configuration values for the Helm chart
        - `templates/_helpers.tpl`: Helper templates for the Helm chart
        - `templates/deployment.yaml`: Kubernetes deployment configurations for app and model services
        - `templates/service.yaml`: Service definitions for inter-service communication
        - `templates/ingress.yaml`: Ingress configuration for external access
        - `templates/servicemonitor.yaml`: ServiceMonitor configurations for Prometheus metrics collection
        - `templates/alertmanager-config.yaml`: AlertManager configuration for notification settings
        - `templates/requests-rule.yaml`: Custom Prometheus alerting rules
    - `app-ingress/app-ingress.yaml`: Define ingress configurations for application routes.
    - `kubernetes dashboard`
        - `dashboard-adminuser.yaml`: Define ServiceAccount and CluterRoleBinding for a KB dashboard admin user.
        - `ingress-dashboard`: Ingress configuration for exposing the KB dashboard.
    - `canary-release.yml`: Defines KB resources for canary deployment.
    - `finalization-istio.yml`: Install istio service mesh and addons (prometheus, jaeger, kiali).
    - `rate-limiting.yaml`: Implements rate limiting using EnvoyFilter.
    - `docs/`
        - `deployment.md`: Deployment description details.
        - `extension.md`: Extension proposal.
- **App Service Repository** holds the relevant frontend and backend code for the application
    - `app/models/model_handler.py` makes a post request to the model-service to get a sentiment prediction
    - `app/routes/__init__.py` defines the routes used by the backend application
    - `app/__init__.py` holds the overall backend application code
    - `app/templates/index.html` defines the frontend code of the application
- **Model Service Repository**
    - `model-service/download_models.py` is used to download the classifier (and vectorizer) model used for the prediction
    - `model-service/app.py` contains the Flask wrapper for the model, with an endpoint that can be accessed by the app-service to request a prediction for a given review
- **Lib-ml Repository**
    - `preprocess_sentiment_analysis/preprocess.py` is used to preprocess a given dataset by cleaning the text (removing non-alphabetical characters, converting all words to lowercase, removing stopwords, and Porter stemming). It saves the results into a processed CSV file.
- **Lib-version**
    - `src/lib_version/version_awerenes.py` is contains the package code used to retrieve the packages version at runtime.
    - `src/lib_version/VERSION` is the config file that contains the build version. This file is used at build time to determine the versioning.
- **Model training**
    - `data/`: The folder containing all the data files
        - `raw/`: Original raw data dumps
        - `processed/`: The processed data directly used by the model
    - `models/`: Containes the models which have already been trained
    - `restaurant_model_training/`: The main package (module) of this project
        - `config.py`: Contains the configurations such as default values and paths
        - `dataset.py`: Logic for loading and preprocessing the data
        - `features.py`: Logic for creating BOW features
        - `modeling/`: Module containing logic for model training (`train.py`) and predicting (`predict.py`)
    - `tests/`: The test files for the model
        - `test_data_features.py`: Tests for data and features
        - `test_infrastructure.py`: Tests for infrastructure
        - `test_model_development.py`: Tests for model training, evaluation, robustness
        - `test_monitoring.py`: Tests for model monitoring
    - `requirements.txt`: The project dependencies

## Repository Links
For convenience, we list links to the available repositories used in this project:
- [operation](https://github.com/remla2025-team10/operation)
- [app-service](https://github.com/remla2025-team10/app-service)
- [model-service](https://github.com/remla2025-team10/model-service)
- [model-training](https://github.com/remla2025-team10/model-training)
- [lib-version](https://github.com/remla2025-team10/lib-version)
- [lib-ml](https://github.com/remla2025-team10/lib-ml)

## Progress Log
### Assignment 1
#### What was done
All core components of the assignment have been implemented.

- **Lib-ml** provides a preprocessing library for cleaning up the dataset text to be used in the sentiment analysis.
- **Model-training** Trains the restaurant sentiment analysis model and publishes it to releases for access.
- **Model-service** applies preprocessing and uses the model to return a prediction.
- **Lib-version** provides a simple library that can produce its build version upon request.
- **App-service** Build the frontend and backend for the project, call model-service api, and use lib-version util.
- **Operation** builds up the containers for the model-service and app-service and hides model-service from the host (only accessible by app-service).

#### What is missing / needs improvement
- **Automated versioning** is implemented only in the lib-version repository on pushes with a release or version tag. It creates releases on main with the correct stable version tag, also pre-release commits and tags automatically on main. Though these pre-release tags are not sub-ordered for now.

### Assignment 2
#### What was done
All core components of the assignment have been implemented.

- **1.1 Basic VM Setup** Configured VMs and networking for Kubernetes. See `general.yaml`.
- **1.2 Setting up the Kubernetes Controller** Initialized the cluster and set up kubectl, Flannel, and Helm on the controller node. See `ctrl.yaml`.
- **1.3 Setting up Kubernetes Workers** Joined worker nodes to the cluster using a token from the controller. See `node.yaml`.
- **1.4 Finalizing the Cluster Setup** Installed MetalLB, NGinx Ingress, Dashboard, for load balancing and routing. See `finalization.yml`.

### Assignment 3
#### What was done

- **1.1 Migration from Docker Compose to Kubernetes** Migrated from `docker-compose` to Kubernetes using `Minikube`. Additionally deployed using `k8s`. Implemented a `helm` chart in the `model-stack` directory.
- **1.2 Monitoring** Added `Prometheus` APIs in `app-service`, configured `Prometheus` with a `ServiceMonitor` to automatically collect app-specific metrics for monitoring user interactions and model performance, integrated `AlertManager` to send email alerts for high request rates, and created a `Grafana` dashboard with visualizations for all metrics.

### What needs to be done
Prometheus dashboard not configured properly, so the actual prometheus rule doesn't fire, alerts should still work

### Assignment 4
#### What was done

- **Cookiecutter template** Refactored `model-training` to follow a cookie-cutter template.
- **Automated tests** Implemented tests following the ML-Test score methodology, all in the `/test` folder in `model-training`.
- **DVC pipeline** Implemented a DVC pipeline using gdrive remote storage in `model-training`.
- **pylint confugration** `model-training` implements pylint configuration within the workflow.

### What needs to be done
Mutamorphic testing.

## Progress Log - Assignment 5
### What was done

- **Traffic Management** Added istio gateway: Kubernetes dashboard, kiali, prometheus, app are all accessible.
- **Sticky Sessions** Added sticky sessions for user routing, so users with the correct header are always redirected to the experimental (v2) version of the app, while others never see it (v1)
- **Continuous Experimentation** Added a new feature: random gifs to the sentiment analysis with a metrics route.
- **Rate limiting** A (base) Isitio envoyfilter has been implemented that should locally rate limits to 10 requests a minute. For now, must be manually applied with `kubectl` once the app is up.
- **Documentation** Description and model are provided in `/docs`.

### What is missing / needs improvement

- Documentation is incomplete. Needs `extension.md`.
- Dashboard access must be configured properly.
- Rate limiting needs to be hotfixed.