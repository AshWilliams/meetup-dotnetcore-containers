## Objectives of this script
The idea is to have a simple to follow step by step procedure in order to:
* Learn and comprehend the basic steps, concepts and terminology of the containers world.
* Generate an image from our sample or existing .NET Core app using both Linux and Windows containers.
* Create and run containers from your local images in you own local machine.
* Publish your images into your own private Azure Registry Container.
* Create and run your containers, from your private images in Azure Container Instances.
* Create and populate an Azure Kubernetes Service with your own private images, using secrets.
* Change your mind about how to design, develop and deploy apps from now on :)

As of today (April 28, 2018), AKS is running in preview mode, so only Linux Containers are supported as pods, but will support both Windows and Linux containers by GA time, or (I hope) before of that.

For questions, mistakes from my side, suggestions or anything else you can reach me by email at sandrof at lagash dot com.

## Requisites
An active Azure account, if you don't have one please get a free demo account:
* [Get an Azure Free Account](https://azure.microsoft.com/es-es/free/)

Docker installed and running in your local machine:
* [Install Docker Community Edition](https://www.docker.com/community-edition)

Git command line:
* [Install Git](https://git-scm.com/downloads)

## Azure resources manually created
### Resource Groups
This resources can be also created by command line, but is simpler via the Azure portal interface:
- RG-DOTNET-MEETUP
- RG-DOTNET-MEETUP-ACI
- RG-DOTNET-MEETUP-AKS

### Azure Registry Container
- dotnetmeetupcl

## Note
Each command specified from now on should be executed in a CMD/Terminal prompt. If you're running this samples on Windows please execute your cmd/Powershell with elevated rights. 
All the steps not referred to local work (Image creation, tagging and publishing into the Azure Container Registry) can be also performed in the Cloud Shell into the Azure Portal.

## Clone this repo
In your preferred folder, checkout a local copy of this repo:
```bash
git clone https://gitlab.lagash.com/sandrof/meetup-dotnetcore-containers
```
All the commands stated in this file must be executed in the folder where you just cloned this repo.

## Login to Azure via the Azure CLI
In order to perform any action against a specific Azure subscription we must to be logged on. If your don't have the CLI tool installed, install it now from the [following link](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

After installing, log in following the on screen instructions.
```bash 
az login
```
After successful login, a results JSON file will be displayed showing all of your current subscriptions, please verify the default one because all the commands will target that one by default

## Image generation
```bash
docker-compose build
```

## Container generation and startup
```bash
docker run -d --name api dotnetcore-rest-api:latest
docker start api
```
### Assigned IP
In order to get the IP address assigned to your container type the following commands
#### Windows containers
```bash
docker exec -it api ipconfig
```
#### Linux containers
```bash
docker inspect api
```
### Test the API running in your local machine
In your browser, type http://\<ipaddress\>/api/values replacing \<ipaddress\> with the results from the previous step output (IPv4 value)

## Publishing to ACR (Azure Container Registry)
### Tag the original image in order to push it to the ACR
```bash
docker tag dotnetcore-rest-api:latest dotnetmeetupcl.azurecr.io/dotnetcore-rest-api:v1
```
### Login to the ACR
```bash
docker login --username dotnetmeetupcl --password vumSewVl87ceI6A/DUERQFGRP75vA1fb dotnetmeetupcl.azurecr.io
```
### Push our tagged image to the registry
```bash
docker push dotnetmeetupcl.azurecr.io/dotnetcore-rest-api:v1
```

## Publish our image to ACI from the ACR
#### Windows containers
```bash
az container create --image dotnetmeetupcl.azurecr.io/dotnetcore-rest-api:v1 --resource-group RG-DOTNET-MEETUP-ACI --location eastus --name api-windows --os-type Windows --cpu 1 --memory 1.5 --dns-name-label api-dns-win --ip-address public --ports 80 --registry-login-server dotnetmeetupcl.azurecr.io --registry-username dotnetmeetupcl --registry-password vumSewVl87ceI6A/DUERQFGRP75vA1fb --verbose
```
#### Linux containers
```bash
az container create --image dotnetmeetupcl.azurecr.io/dotnetcore-rest-api:v2 --resource-group RG-DOTNET-MEETUP-ACI --location eastus --name api-linux  --cpu 1 --memory 1.5 --dns-name-label api-dns-linux --ip-address public --ports 80 --registry-login-server dotnetmeetupcl.azurecr.io --registry-username dotnetmeetupcl --registry-password vumSewVl87ceI6A/DUERQFGRP75vA1fb --verbose
```
### Check status and information about the just created container
#### Windows containers
```bash
az container show --name api-windows --resource-group RG-DOTNET-MEETUP-ACI
``` 
#### Linux containers
```bash
az container show --name api-linux --resource-group RG-DOTNET-MEETUP-ACI
```
Two sections of the displayed JSON result are important:
#### Running state
The "state" field will show "Running" as soon as our container has been deployed and started by Azure.
```JSON
"currentState": {
    "additionalProperties": {},
    "detailStatus": "",
    "exitCode": null,
    "finishTime": null,
    "startTime": "2018-04-25T15:33:12+00:00",
    "state": "Running"
},
```
#### Container instance fqdn (fully qualified domain name)
The value for the "fqdn" field -in this case "api-dns.eastus.azurecontainer.io"- will be the base URL for our API endpoints.
```JSON
"ipAddress": {
"additionalProperties": {},
"dnsNameLabel": "api-dns",
"fqdn": "api-dns.eastus.azurecontainer.io",
"ip": "40.121.199.84",
"ports": [
    {
    "additionalProperties": {},
    "port": 80,
    "protocol": "TCP"
    }
]
}
```

## AKS Cluster creation
First we will create the actual AKS kluster into the resource group we've already created
```bash
az aks create -n DOTNET-MEETUP-CLUSTER -g RG-DOTNET-MEETUP-AKS -c 2 -k 1.8.7 --generate-ssh-keys -l eastus
```
The cluster creation will take some minutes (~15), after that get the credentials in order to operate against our managed Kubernetes instance.
```bash
az aks get-credentials -n DOTNET-MEETUP-CLUSTER -g RG-DOTNET-MEETUP-AKS
```
### Kubernetes dashboard
Kubernetes offers a complete dashboard but in order to gain access to it we need to tunnel our computer's requests to the cluster's loopback IP address (127.0.0.1) typing:
```bash
kubectl proxy
```
After that we can reach the dashboard from the following URL: http://127.0.0.1:8001/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
### Create Kubernetes secrets
```bash
kubectl create secret docker-registry dotnet-meetup-cluster-secrets --docker-server dotnetmeetupcl.azurecr.io --docker-email sandrof@lagash.com --docker-username=dotnetmeetupcl --docker-password vumSewVl87ceI6A/DUERQFGRP75vA1fb
```
### Create a deployment
A deployment is basically an object representing a pod's content, defined via a YAML or JSON file. We need them in order to populate our kluster nodes.
```bash
kubectl apply -f dotnetcore-rest-api.yml
```
In order to know the published IP address for our published pod, we need to ask to our cluster:
```bash
kubectl get service
```


## Useful links:
* [Visual Studio Tools for Docker with ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/visual-studio-tools-for-docker?view=aspnetcore-2.1)
* [Dockerfile reference (Official)](https://docs.docker.com/engine/reference/builder/)
* [Is there a Windows Docker image for ...?](https://stefanscherer.github.io/is-there-a-windows-docker-image-for/)
* [Deploy to Azure Container Instances from Azure Container Registry](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-using-azure-container-registry)
* [Azure Container Instances with multiple containers](https://medium.com/@lizrice/azure-container-instances-with-multiple-containers-512c022c04ec)
* [Deploy a container group (in Azure Container Instances)](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-multi-container-group)
* [Containers as a Service in Azure](https://xpirit.com/2017/11/12/containers-as-a-service-in-azure/)
* [Running Windows Containers on Kubernetes Cluster on Azure Container Service](https://koukia.ca/running-windows-containers-on-kubernetes-cluster-on-azure-container-service-99e8de0d9cf4)
* [Docker glossary](https://docs.docker.com/glossary/)
* [Kubernetes Standarized Glossary](https://kubernetes.io/docs/reference/glossary/?fundamental=true)
* [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
* [Kubernetes Monitoring with Prometheus & Grafana](https://itnext.io/creating-a-full-monitoring-solution-for-arm-kubernetes-cluster-53b3671186cb)
