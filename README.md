# Example Voting App

A simple distributed application running across multiple Docker containers.

## Getting started

Download [Docker Desktop](https://www.docker.com/products/docker-desktop) for Mac or Windows. [Docker Compose](https://docs.docker.com/compose) will be automatically installed. On Linux, make sure you have the latest version of [Compose](https://docs.docker.com/compose/install/).

This solution uses Python, Node.js, .NET, with Redis for messaging and Postgres for storage.

Run in this directory to build and run the app:

```shell
docker compose up
```

The `vote` app will be running at [http://localhost:5000](http://localhost:5000), and the `results` will be at [http://localhost:5001](http://localhost:5001).

Alternately, if you want to run it on a [Docker Swarm](https://docs.docker.com/engine/swarm/), first make sure you have a swarm. If you don't, run:

```shell
docker swarm init
```

Once you have your swarm, in this directory run:

```shell
docker stack deploy --compose-file docker-stack.yml vote
```

## Run the app in Kubernetes

The folder k8s-specifications contains the YAML specifications of the Voting App's services.

Run the following command to create the deployments and services. Note it will create these resources in your current namespace (`default` if you haven't changed it.)

```shell
kubectl create -f k8s-specifications/
```

The `vote` web app is then available on port 31000 on each host of the cluster, the `result` web app is available on port 31001.

To remove them, run:

```shell
kubectl delete -f k8s-specifications/
```

## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time

## Notes

The voting application only accepts one vote per client browser. It does not register additional votes if a vote has already been submitted from a client.

This isn't an example of a properly architected perfectly designed distributed app... it's just a simple
example of the various types of pieces and languages you might see (queues, persistent data, etc), and how to
deal with them in Docker at a basic level.


## Setup Ingress Controller & Cert Manager

- First, create an AKS cluster with one `agent_node` and `user_node` with `dev/test` environment in Azure
- Run the below command to get `AKS Cluster` resource group
```bash
az aks show --resource-group <resource_group_name> --name <aks_cluster_name> --query nodeResourceGroup -o tsv
```
- create the kubernetes namespace
```bash
kubectl create ns ingress-basic
```
- Install the the `Helm` if you don't have and then install the `ingress-controller` using the below command and replace the `PUBLIC_IP` with the IP that we got from the above step
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx 
--namespace ingress-basic 
--set controller.replicaCount=2 
--set controller.nodeSelector."beta\.kubernetes\.io/os"=linux 
--set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux 
--set controller.service.externalTrafficPolicy=Local 
--set controller.service.loadBalancerIP="<PUBLIC_IP>"
```

This will launch the `ingress deployment, pods and service` objects.

- Test the ingress-controller pods,
```bash
kubectl get pods -n ingress-basic
```

- Run the following commands to setup the Certificate Manager that we will use to secure our website with SSL certificates,
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.crds.yaml

helm repo add jetstack https://charts.jetstack.io

helm install my-certmanager --namespace ingress-basic --version v1.13.2 jetstack/cert-manager
```

- Now we are done with setup, go to `k8s-specifications -> ingress` directory and apply the resources objects in following order using command `kubectl apply -f <file_name.yaml>`,
    - aks-helloworld-one.yaml
    - aks-helloworld-two.yaml
    - cluster-issuer-prod.yaml
    - cert-ingress-prod.yaml

- You can test your application now on configured hostname with secure SSL certificate.

## Setup Prometheus and Grafana
- To setup prometheus, run the below commands in order,
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus -n ingress-basic
```

- To setup grafana, run the below commands in order,
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana -n ingress-basic
```

- Now we are done with setup, go to `k8s-specifications -> ingress` directory and apply the resources objects in following order using command `kubectl apply -f <file_name.yaml>`,
    - aks-helloworld-one.yaml
    - aks-helloworld-two.yaml
    - cluster-issuer-prod.yaml

- Now go to `k8s-specifications -> prometheus-grafana` directory and apply the resources objects in following order using command `kubectl apply -f <file_name.yaml>`,
    - grafana-ingress.yaml

- Also, go to `Config -> ConfigMap` in OpenLens, select the namespace `ingress-basic` (where grafana is installed) and then choose the `grafana.ini` config and update it with the following lines from the file `k8s-specifications -> prometheus-grafana -> grafana-configmap.yaml`,
```bash
[server]
    domain = 'aksdemo.westeurope.cloudapp.azure.com'
    root_url = 'https://aksdemo.westeurope.cloudapp.azure.com/grafana'
    serve_from_sub_path = false
```

- Open the grafana dashboard with `https://domain_name.com/grafana` and login with default credentials that grafana gave when we setup in above steps. Now from `data source`, configure `prometheus` with the URL that we got in above steps while installing the prometheus.

## References
- https://artifacthub.io/packages/helm/prometheus-community/prometheus
- https://github.com/grafana/grafana/issues/72577
- https://github.com/grafana/grafana/issues/70110
- https://www.youtube.com/playlist?list=PL9aNQqB-xjbAjGjkAKuydrYa8ZDM4TcI7
- https://github.com/MuhammedKalkan/OpenLens
- https://medium.com/geekculture/fix-essential-functionality-missing-in-the-new-openlens-version-f9ac862e9e27
- https://stacksimplify.com/azure-aks/azure-kubernetes-service-ingress-basics/
