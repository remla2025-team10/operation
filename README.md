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

### Finalize cluster setup
Finally, you can run the following command from the host to finalize the cluster setup:
```bash
ansible-playbook -u vagrant -i 192.168.56.100, finalization.yml 
```

#### Local DNS Resolution
On your host machine, make sure to add `app.local` and `dashboard.local` in your `/etc/hosts` file:

```bash
sudo nano /etc/hosts
```

Add the following line at the end:
```bash
192.168.56.90  app.local dashboard.local
```

Save and exit (`Ctrl + O`, then `Enter`, then `Ctrl + X`)

Finally, flush the DNS cache:

```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

When everything is complete, the Kubernetes Dashboard should be accessible at [dashboard.local](dashboard.local) and our app should be accessible at [app.local](app.local).

To log in, generate an admin token by running this command on the control node:
```bash
kubectl -n kubernetes-dashboard create token admin-user
```
Copy the output token and use it to log in to the dashboard.

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

Manually create a `/mnt/shared` directory:
```bash
minikube ssh
sudo mkdir -p /mnt/shared
exit
```

#### 2. Deployment

First navigate to the model-stack directory (which stores the helm chart)
```bash
cd operation/model-stack
```
Then, install the helm release
```
helm install <release-name> . \
  --set appService.modelServiceUrl=http://<release-name>-model-service:3000
```

#### 4. Port forward

Run the following to port forward the app service: 
```bash
kubectl port-forward svc/<release-name>-app-service 8080:8080
```

Now you can access the app via `http://localhost:8080/`

### 5. Alerts

To configure alerts to your email, you first need to create a secret for the sender gmail app key (for authentication). For testing purposes we provide you with the actual app key, but this will not be shared in the final version:
```bash
kubectl create secret generic smtp-password-secret \
  --from-literal=smtp_passwordku='lmyl nlxo hjdc hpzn' \
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
    - `ansible/general.yaml`: Configures all VMs with necessary prerequisites for Kubernetes, including required system settings.
    - `ansible/ctrl.yaml`: Initializes the Kubernetes cluster on the control node.
    - `ansible/node.yaml`: Configures worker nodes and joins them to the Kubernetes cluster.
    - `ansible/finalization.yaml`: Installed MetalLB, NGinx Ingress, Dashboard, (and Istio) for load balancing and routing
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
    - `train.py` is used to train the restaurant sentiment analysis model.
    - `data/a1_RestaurantReviews_HistoricDump.tsv` is the training data that has been used to train the model.
    - `model/` contains the trained model files that is ready to use for the model-service.

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
