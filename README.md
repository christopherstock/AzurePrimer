
Azure is Microsoft's public cloud computing platform. It provides a broad range of cloud services to develop and scale new applications or run existing applications in the public cloud.
In this workshop, we'll work through five different deployment scenarios. Every scenario will deploy a different web application into the Azure cloud and thus make them accessible over a public web or IP address.

---

# Ramp Up

## 1. Prerequisites
The following tools and accounts are required for all deployment scenarios. Basically the installation of the Azure CLI and a login to your Azure Account.

### 1.1. Install Azure CLI
Install Azure CLI client. On macOS via homebrew you can easily update brew and install it like this. Find more information and troubleshooting on [installing the Azure CLI client on other platforms](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli).
```
brew update
brew install azure-cli
```

### 1.2. Verify Azure CLI Client
Verify the successful installation of the Azure CLI client by running the following command:
```
az version
```

You should see an output similar to:
```
{
  "azure-cli": "2.56.0",
  "azure-cli-core": "2.56.0",
  "azure-cli-telemetry": "1.1.0",
  "extensions": {}
}
```

### 1.3. Login to your Azure Account via CLI
This will open the login process in your default browser. You can also create a new Microsoft account in this step.
```
az login
```

The login process requires the browser to open and a login with your Microsoft Account credentials using the web UI of your browser. After successfully logging in to your account inside your browser, you should see output like the following on your command line:

**Response**:
```
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "d4fa5945-786d-4275-ae82-ae79e0080476",
    "id": "3d43defa-baec-482d-b7b0-eb739c822806",
    "isDefault": true,
    "managedByTenants": [],
    "name": "Azure subscription 1",
    "state": "Enabled",
    "tenantId": "d4fa5945-786d-4275-ae82-ae79e0080476",
    "user": {
      "name": "email@christopherstock.de",
      "type": "user"
    }
  }
]
```

## 2. Additional Tools

### 2.1. Install local Node.js and npm
This is only required for scenario 2 "Deploy a local Node.js Web App". Check that node and npm are both installed using the following commands:
```
node --version
npm --version
```

### 2.2. Install Docker
This is only required for scenario 3 "Deploy a custom Container". You can check if the docker daemon is running by invoking the following Docker command:
```
docker --version
```

### 2.3. Install Terraform
An installation of Terraform is only required for scenario 4 "IaC Tool Terraform". You can verify that Terraform is installed by using the following command:
```
terraform --version
```

### 2.4. Install Kubernetes CLI
This is only required for scenario 5 "Kubernetes Cluster Deployment"
```
kubectl version
```

---

# Five Azure Deployment Scenarios
All scenarios are using the Azure CLI client. The first three scenarios will deploy a different web application to a public web- or IP-address. The fourth scenario will deploy a web application hosted inside a Docker container using the Infrastructure as Code (IaC) tool Terraform and is therefore not working with Azure CLI commands directly, although Terraform is using the Azure CLI client internally. And our last scenario will create and deploy an entire Kubernetes cluster inside the Azure cloud, that hosts a complex web application with an central event bus and several services. All scenarios are made publicly available over a public web adress or IP. So let's roll.

## 1. First Scenario: Deploy a Preconfigured Container
Our first scenario will deploy a ready-made container that is based on a Node.js image and contains a minimum static HTML website.

### 1.1. Setup a Resource Group and a Container

#### 1.1.1. Setup a Resource Group
Every created Azure resource needs to be assigned to a specific resource group. The following command will will create a new resource group that clusters our cloud resources for the 1st scenario. The name of the new resource group is an arbitrary value, `azure-workshop-resource-group` in our case.
```
az group create --name scenario1-resource-group --location germanywestcentral
```
You can verify that the resource group has been created using the following command:
```
az group list
```
You can retrieve a list of all regions that are available for your account using:
```
az account list-locations
```

#### 1.1.2. Setup a Container
`scenario1-container-app` is an arbitrary name and it **must** be unique inside the Azure universe:
```
az container create --resource-group scenario1-resource-group --name scenario1-container --image mcr.microsoft.com/azuredocs/aci-helloworld --dns-name-label scenario1-container-app --ports 80
```

The image `mcr.microsoft.com/azuredocs/aci-helloworld` is taken from the Docker hub and hosts a minimum HTML website inside an nginx container. You can also specify a different container image from the [Docker hub](https://hub.docker.com) here.

### 1.2. Access the running Container

#### 1.2.1. Access the running Web App
Open the following web address inside your browser in order to see the website running inside the nginx container:
http://scenario1-container-app.germanywestcentral.azurecontainer.io/

> On first access, it might take some time for the app to respond. This is because the Azure App Service needs to pull the entire image from the container registry. More steps that can be applied to the web app, like enabling CI/CD for the app service or retrieving diagnostics logs, can be found [here](https://learn.microsoft.com/en-us/azure/app-service/tutorial-custom-container?tabs=azure-cli&pivots=container-linux#vi-configure-the-web-app).

You can query the IP address and the provisioning state of the container with the following command:
```
az container show --resource-group scenario1-resource-group --name scenario1-container --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" --out table
```

#### 1.2.2. Pull Container Logs
Pull and receive the logs of the running container with the following command:
```
az container logs --resource-group scenario1-resource-group --name scenario1-container
az container list --resource-group scenario1-resource-group --output table
```

#### 1.2.3. Attach Container to local Output Streams
You can attach to the container output stream using the following command:
```
az container attach --resource-group scenario1-resource-group --name scenario1-container
```

### 1.3. Cleanup all Resources

#### 1.3.1. Delete the Container
Use the following command to delete the container, which will undeploy the running web app:
```
az container delete --resource-group scenario1-resource-group --name scenario1-container --yes
```

Verify that the container is deleted using the following command:
```
az container list --resource-group scenario1-resource-group --output table
```

#### 1.3.2. Delete the entire Resource Group
You can also just delete the resource group so all resources associated to it get deleted as well:
```
az group delete --name scenario1-resource-group --yes
```

Verify that the resource group is gone using the following command:
```
az group list
```

---

## 2. Second Scenario: Deploy a local Node.js Web App
This will create a local Node.js web application and make it publicly available using the Azure Webapp Service.

### 2.1. Create and Run an Express.js Web App
This will create a new Express.js Web Application in a subdirectory `node-express-app`. The name of the subdirectory is arbitrary.

#### 2.1.1. Create a Skeleton Express.js Web App
Use the following command to generate an Express.js app skeleton:
```
npx express-generator scenario2-node-express-app --view ejs
```

#### 2.1.2. Install all Node.js packages locally
```
cd scenario2-node-express-app
npm install
```

#### 2.1.3. Run the Express.js web app locally
```
npm start
```

#### 2.1.4. Query the Express.js routes locally
```
http://localhost:3000/
http://localhost:3000/users
```

### 2.2. Deploy the Express.js Web App to Azure

#### 2.2.1. Run the Magic Azure Web App Deploy Command
The `--sku F1` flag specifies that the web app will be created on the free pricing tier. The specified `--name` of the web app is an arbitrary value but must be unique inside the Azure universe. This command needs to be run from inside the Express.js web app directory!
```
cd scenario2-node-express-app
az webapp up --sku F1 --name scenario2-express-app --location germanywestcentral
```

#### 2.2.2. Verify the Web App Deployment
The web app is now deployed and accessible over the yielded output URL. This will return the response of the index route:
```
http://scenario2-express-app.azurewebsites.net
```

The `/users` route is also accessible:
```
http://scenario2-express-app.azurewebsites.net/users
```

#### 2.2.3. Review all created Resources
The following resources have been automatically created by the `az webapp up` command:

- 1 resource group with an automatically assigned name (e.g. `email_rg_7535`)
- 1 app service plan with an automatically assigned name (e.g. `email_asp_0497`)
- 1 app service with our arbitrary name `scenario2-express-app`

The deploy command has automatically zipped all files from the current working directory and deployed them into the created app. The parameters of the command have been cached inside file `.azure/config` so future `az webapp` commands don't need to specify them again.

### 2.3. Redeploy Updates on the Express.js
Just change contents of the web app, e.g. inside file `scenario2-node-express-app/views/index.ejs`, and run the az webapp command again. As parameters are cached, you won't need to specify them again:

```
az webapp up
```

Reload the base route and see the updated web app in action:
```
https://scenario2-express-app.azurewebsites.net/
```

### 2.4. Stream Logs from the web app
```
az webapp log tail
```
Logs render all output being produced via the `console.log` function inside our JavaScript files, e.g. by adding the following 1st line inside file `scenario2-node-express-app/app.js`:
```
console.log('Express.js web app being invoked')
```

### 2.5. Cleanup all Resources
This will delete the entire resource group with all of its contained resources. The name of the resource group is taken from the cached file `.azure/config`.
```
az group delete --yes
```

---

## 3. Third Scenario: Deploy a custom Container
This will deploy a customized Docker container. We'll run the Docker container locally in the 1st step and then deploy the container to Azure in order to make our web application publicly available.

### 3.1. Run Docker Container Application Locally

TODO 1: derive Dockerfile from nginx Docker image
TODO 2: change web app to Coding Ninjas II
TODO 3: add logos for Azure Services

```
cd scenario3-nginx-app
docker build --file 'Dockerfile-nginx' --tag scenario3-nginx-app-image .
docker run --detach --publish 1234:80 --name scenario3-nginx-app-container scenario3-nginx-app-image
http://localhost:1234
docker container rm --force scenario3-nginx-app-container
```

### 3.2. Deploy Docker Container Application to Azure

#### 3.2.1. Create Resource Group
```
az group create --name scenario3-resource-group --location germanywestcentral
```

#### 3.2.2. Create Managed Identity
```
az identity create --name azure-workshop-managed-identity --resource-group scenario3-resource-group
```

#### 3.2.3. Create and Push to Container Registry

##### 3.2.3.1. Create Container Registry
The name of the container registry must be unique across all of azure. Only lowercases and alphanumeric characters are allowed.
```
az acr create --name mayflowernginxregistry --resource-group scenario3-resource-group --sku Basic --admin-enabled true
```

##### 3.2.3.2. Retrieve Admin Credentials for newly created Container Registry
```
az acr credential show --resource-group scenario3-resource-group --name mayflowernginxregistry
```

output:
```
{
  "passwords": [
    {
      "name": "password",
      "value": "TIfAub1ewHXDmTwdrcUNSOHDwtnVj0MKE2rAObuvwc+ACRAzvXLB"
    },
    {
      "name": "password2",
      "value": "MGS6btXydkeJagK6aOq6+sXq4Nk2jlJmQQeYsNqDII+ACRDH34SS"
    }
  ],
  "username": "mayflowernginxregistry"
}
```

##### 3.2.3.3. Push local Docker Image to Azure Container Registry
Login to the Docker Registry using the local Docker command:
```
docker login mayflowernginxregistry.azurecr.io --username mayflowernginxregistry --password TIfAub1ewHXDmTwdrcUNSOHDwtnVj0MKE2rAObuvwc+ACRAzvXLB
```

##### 3.2.3.4. Tag local Docker Image to Registry
TODO Check if this step is required!
```
docker tag scenario3-nginx-app-image mayflowernginxregistry.azurecr.io/scenario3-nginx-app-image:latest
```

##### 3.2.3.5. Push Tagged local Docker Image to Registry
```
docker push mayflowernginxregistry.azurecr.io/scenario3-nginx-app-image:latest
```

#### 3.2.4. Authorize Managed Identity for the Container Registry
This step is required in order to pull from an Azure Container Registry

TODO split to sub steps
```
export principalId=$(az identity show --resource-group scenario3-resource-group --name azure-workshop-managed-identity --query principalId --output tsv)
export registryId=$(az acr show --resource-group scenario3-resource-group --name mayflowernginxregistry --query id --output tsv)
az role assignment create --assignee $principalId --scope $registryId --role "AcrPull"
```

The role `AcrPull` has now been added to the `Azure Role Assignments` of the managed identity `azure-workshop-managed-identity`.

#### 3.2.5. Create new Web App from Tagged Container Image of Azure Container Registry
This will create an app service plan for our new web app:
```
az appservice plan create --name azure-workshop-app-service-plan --resource-group scenario3-resource-group --is-linux
```

The web app can afterwards be created from the recently created app service plan:
```
az webapp create --resource-group scenario3-resource-group --plan azure-workshop-app-service-plan --name mayflowernginxapp --deployment-container-image-name mayflowernginxregistry.azurecr.io/scenario3-nginx-app-image:latest
```

#### 3.2.6. Verify the Web App
The web application is now available at
```
http://mayflowernginxapp.azurewebsites.net/
```

#### 3.2.7. Cleanup all Resources
Just delete the entire resource group in order to ditch all created resourced from this chapter.
```
az group delete --name scenario3-resource-group --yes
```

---

## 4. Fourth Scenario: Deploy a Web App inside an Application Container using the IaC-Tool Terraform
Terraform is an Infrastructure as Code Tool. It uses declarative files, specified in HCL (HashiCorp Configuration Language) and using the YAML format. The intention is to specify the entire cloud infrastructure in this declarative language and use a minimum of CLI commands in order to deploy, destroy or update the entire cloud infrastructure.

TODO try own container app?

### 4.1. New Terraform Directory
All HCL files reside in a new subdirectory `terraform` and all Terraform commands of this scenario are executed from inside this directory. So we create and change to this new directory:
```
mkdir scenario4-terraform
cd scenario4-terraform
```

### 4.2. Create all Terraform Files
The Terraform configuration is a set of multiple Terraform files. We'll create our new Terraform configuration inside directory `scenario4-terraform`.

#### 4.2.1. Providers
Inside file `providers.tf`

#### 4.2.2. Main
Inside file `main.tf`

#### 4.2.3. Variables
Inside file `variables.tf`

#### 4.2.4. Outputs
Inside file `outputs.tf`

### 4.3. Init Terraform Configuration
This will install all required Terraform packages, specified inside the `required_providers` declaration of the `providers.tf` file.

```
terraform init
```

### 4.4. Apply Terraform Configuration
This will apply the new Terraform configuration so all resources are created inside the Azure cloud.

```
terraform apply --auto-approve
```

### 4.5. Verify Container Deployment
The result of the `terraform apply` command yields the value of the output variable `container_ipv4_address`. This is the IP address of the machine where our web application has been deployed. To display all output variables at a later point of time, use the following command:

```
terraform output
```

The following resources have been created:
- One Resource Group
- One Container Instance

### 4.6. Destroy the Terraform Configuration
This will cleanup all resources that have been created after applying the Terraform configuration.

```
terraform destroy --auto-approve
```

---

## 5. Fifth Scenario: Deploy a Web App via Kubernetes using AKS (Azure Kubernetes Service)
Kubernetes is a Container Orchestration.

### 5.1. Download and Extract Kubernetes Store Demo
```
cd scenario5-kubernetes-store-demo
```

### 5.2. Run Docker Compose on sample docker-compose File
This will run a local Docker setup being specified inside Docker Compose file `docker-compose-quickstart.yml`.
```
docker compose -f docker-compose-quickstart.yml up -d
```

Make sure to kill any process blocking port `8080`:
```
sudo lsof -i:8080
kill -9 <PID>
```

You can see all created images and all running containers using the following Docker commands:
```
docker images
docker ps
```

### 5.3. Open the Store Demo Frontend
```
http://localhost:8080/
```

### 5.4. Shutdown Docker Compose Container Setup
You can stop all running docker containers specified inside the Docker compose file using the following command:
```
docker compose -f docker-compose-quickstart.yml down
```

### 5.5. Create and Push to Azure Container Registry (ACR)
Register provider:
```
az provider register -n Microsoft.ContainerRegistry
```

```
az group create --name scenario5-resource-group --location germanywestcentral
az acr create --resource-group scenario5-resource-group --name mayflowerazureprimercontainerregistry --sku Basic
az acr build --registry mayflowerazureprimercontainerregistry --image kubernetes-store-demo/product-service:latest ./src/product-service/
az acr build --registry mayflowerazureprimercontainerregistry --image kubernetes-store-demo/order-service:latest ./src/order-service/
az acr build --registry mayflowerazureprimercontainerregistry --image kubernetes-store-demo/store-front:latest ./src/store-front/
```

List your 3 images inside the repository of this container registry via:
```
az acr repository list --name mayflowerazureprimercontainerregistry --output table
```

### 5.6. Create a Kubernetes Cluster in AKS
Delete or rename your existing `~/.ssh/id_rsa` private key before this example:

```
az aks create \
    --resource-group scenario5-resource-group \
    --name mayflowerazureprimerkubecluster \
    --node-count 2 \
    --generate-ssh-keys \
    --attach-acr mayflowerazureprimercontainerregistry
```

### 5.7. Get and Store AKS Credentials and Retrieve Cluster Nodes
This merges credentials into Kubernetes config file `~/.kube/config`
```
az aks get-credentials --resource-group scenario5-resource-group --name mayflowerazureprimerkubecluster
```

You should now be able to recieve the two cluster nodes:
```
kubectl get nodes
```

### 5.8. login to Azure Kubernetes Server

Get login server address
```
az acr list --resource-group scenario5-resource-group --query "[].{acrLoginServer:loginServer}" --output table
```

### 5.9. Create Manifest File

```
code aks-store-quickstart.yaml
```

Replace server name with your output server name.

### 5.10. Deploy!
```
kubectl apply -f aks-store-quickstart.yaml
```

### 5.11. Test App
Get the external IP of the app:
```
kubectl get service store-front --watch
```

Troubleshooting -- Pods might not be running due to missing Kubernetes Authorization:
```
kubectl get pods
```

### 5.12. Delete Kubernetes Setup
```
kubectl delete -f aks-store-quickstart.yaml
```

### 5.13. Delete Entire Scenario Setup
Deleting the resource group will also delete all resources from this scenario.
```
az group delete --name scenario5-resource-group --yes
```

---

I met Azure as a fully developed and highly accessible cloud platform with a high developer experience. Deployments are performed quickly and IP adresses and DNS names are available immediately after deployment.

Using tools like Terraform or Kubernetes are designed to cut down repeated manual steps on cloud resources deployment so you can save precious development time that can be better invested into key project deliveries. The tools used in this workshop tools will enable you to automate and manage tasks with ease! So why wait? Get started with your very own Azure Cloud solution and experience the benefits when working with cloud resources today!

We encourage you to reach out to us with any questions or for further information!

*Team Multiloop*
