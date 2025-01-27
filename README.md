# Build and Deploy Azure Cloud-based e-Commerce Platform
![Deploy e-Commerce Platform in Azure drawio](https://github.com/user-attachments/assets/a2f6177e-456e-4f7b-ae40-42be31bf388a)

## Project Overview  

This project involves creating a secure and scalable architecture for a multi-tier application, typically with:  

- **Frontend**: Hosted on Azure App Service.
- **Backend**: Deployed on Azure Kubernetes Service (AKS).
- **Database**: Azure SQL Database for product, user, and order data.
- **Storage**: Azure Blob Storage for static assets (e.g., images).


Skills Demonstrated:  

- Infrastructure as Code (IaC).
- Networking and security (VNets, NSGs, WAF and Private Endpoints).
- CI/CD pipelines for automated deployments.
- Monitoring and scaling.

## Step 1: Set Up Azure Environment

This step prepares the Azure environment for managing and hosting various resources necessary for the e-commerce platform.  

### 1.1. Create a Resource Group

A Resource Group in Azure is a logical container that holds related Azure resources.  
It helps organize resources, simplify billing, and apply management policies like access control.  

<br/>Steps: 

1\. Log in to the Azure Portal or use the CLI.  
2\. To create a Resource Group:  
```bash
az group create --name EcommerceRG --location eastus  
```

- Replace EcommerceRG with your preferred name.
- Choose a region (eastus in this case) based on where you want your resources located for optimal performance.
  
### 1.2. Create a Virtual Network (VNet)

A Virtual Network (VNet) provides isolated, secure communication between Azure resources. The VNet will host all your application resources and control network traffic between them.

<br/>Steps:

1\. Create the VNet: 
```bash
az network vnet create --name EcommerceVNet --resource-group EcommerceRG \  
--address-prefix 10.0.0.0/16 --subnet-name FrontendSubnet --subnet-prefix 10.0.1.0/24  
```

- **10.0.0.0/16**: The address space for the entire VNet.
- **10.0.1.0/24**: A subnet dedicated to the frontend (you will define others for backend and databases).

2\. Add additional subnets for the Backend and Database:  
```bash
az network vnet subnet create --address-prefixes 10.0.2.0/24 \  
--name BackendSubnet --vnet-name EcommerceVNet --resource-group EcommerceRG
```
```bash
az network vnet subnet create --address-prefixes 10.0.3.0/24 \ 
--name DatabaseSubnet --vnet-name EcommerceVNet --resource-group EcommerceRG  
```

### 1.3. Secure Subnets with NSGs (Network Security Groups)

NSGs allow you to define inbound and outbound rules to control network traffic.  
By applying NSGs, you can restrict traffic to specific services for better security.  

<br/>Steps:

1\. Create an NSG for the Frontend subnet:  
```bash
az network nsg create --name FrontendNSG --resource-group EcommerceRG
```
 
<br/>2\. Add inbound rules for HTTP and HTTPS traffic:  
```bash
az network nsg rule create --nsg-name FrontendNSG --resource-group EcommerceRG \ 
--name AllowHTTP --priority 100 --protocol Tcp --destination-port-ranges 80
```
```bash
az network nsg rule create --nsg-name FrontendNSG --resource-group EcommerceRG \  
--name AllowHTTPS --priority 101 --protocol Tcp --destination-port-ranges 443
```

<br/>3\. Associate the FrontendNSG with the Frontend subnet:  
```bash
az network vnet subnet update --name FrontendSubnet --vnet-name EcommerceVNet \  
--resource-group EcommerceRG --network-security-group FrontendNSG
```
<br/>4\. Repeat for Backend and Database subnets, restricting access to only the necessary ports.  

## Step 2: Deploy the Frontend

<br/>This step focuses on deploying the React/Angular frontend application using Azure App Service.  

### 2.1. Create App Service Plan  

An App Service Plan defines the compute resources for the web app, determining scalability and performance.  
A suitable App Service Plan ensures the frontend can scale and be highly available.  

<br/>Steps:

1\. Create an App Service Plan:  
```bash
az appservice plan create --name EcommerceAppPlan --resource-group EcommerceRG \  
--sku P1v2 --is-linux
```  

- **P1v2**: A Premium tier providing autoscaling support.
- **--is-linux**: Denotes that the plan is for Linux-based applications.

### 2.2. Deploy Frontend Application

Deploy the frontend (e.g., React or Angular app) to Azure App Service.  

Azure App Service is a fully managed platform for web applications, providing easy deployment, scaling, and monitoring. 

<br/>Steps:  

1\. Create a web app: 
```bash
az webapp create --name EcommerceFrontend --resource-group EcommerceRG \  
--plan EcommerceAppPlan --runtime "NODE|16-lts"
```

<br/>2\. Deploy the app from a Git repository:  
```bash
az webapp deployment source config --name EcommerceFrontend \  
--resource-group EcommerceRG --repo-url <repository-url> --branch main --manual-integration
```

### 2.3. Enable HTTPS and Custom Domain

Secure the web application with HTTPS and attach a custom domain.  
HTTPS ensures secure communication between clients and your website. A custom domain improves brand recognition.  

<br/>Steps:  

1\. Configure a custom domain with App Service

a) Get the Default Domain of the App Service
```bash
az webapp show --name EcommerceFrontend --resource-group EcommerceRG --query defaultHostName
```

b) Add the Custom Domain to the App Service
```bash
az webapp config hostname add --webapp-name EcommerceFrontend --resource-group EcommerceRG --hostname <CustomDomain>
```

c) Go to your DNS provider’s portal and add a CNAME record to map the Custom Domain to the App Service hostname

- Hostname: **www**
- Value: **<AppServiceDefaultHostName>**

d) Verify Custom Domain ownership by:

- Retrieve TXT record value
```bash
az webapp config hostname list --webapp-name EcommerceFrontend --resource-group EcommerceRG
```

- Add TXT record to DNS provider
- Record Type: **TXT**
- Hostname: **\_dnsauth.<CustomDomain>**
- Value: **TXT record value**

**\*\*Note: DNS Propagation can take between a few minutes to 24 hours**

2\. Create and bind an SSL certificate using Azure App Service Managed Certificate (Free) to enable HTTPS.
```bash
az webapp config ssl create --resource-group EcommerceRG --name EcommerceFrontend --hostname <CustomDomain>
```
```bash
az webapp config ssl bind --resource-group EcommerceRG --name EcommerceFrontend --certificate-thumbprint <Thumbprint> --ssl-type SNI
```

3\. Force HTTPS for all Traffic: to ensure all HTTP traffic is redirected to HTTPS
```bash
az webapp update --resource-group EcommerceRG --name EcommerceFrontend --set httpsOnly=true
```

## Step 3: Set Up Backend (Azure Kubernetes Service)  

Deploy backend APIs to Azure Kubernetes Service (AKS), a scalable, managed platform for containerized applications.  

### 3.1. Create an AKS Cluster  

AKS is a managed Kubernetes service for deploying and managing containerized applications.  
It allows you to scale and manage your backend services seamlessly.  

Steps:  

1\. Create the AKS Cluster:  
```bash
az aks create --resource-group EcommerceRG --name EcommerceAKS \  
--node-count 3 --enable-managed-identity --generate-ssh-keys
```
 
<br/>2\. Connect to the cluster:  
```bash
az aks get-credentials --resource-group EcommerceRG --name EcommerceAKS
```

### 3.2. Deploy Backend Microservices

Containerize backend services and deploy them to the AKS cluster.  
Containerized services are portable, scalable, and easy to manage.

<br/>Steps:  

1\. Build and push Docker images to Azure Container Registry (ACR): 
```bash
az acr build --registry EcommerceACR --image backend:v1 .
```
 
<br/>2\. Deploy the backend microservices using a YAML deployment manifest:  
```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
name: backend  
spec:  
replicas: 3  
selector:  
matchLabels:  
app: backend  
template:  
metadata:  
labels:  
app: backend  
spec:  
containers:  
- name: backend  
image: backend:v1  
ports:  
- containerPort: 80
```

<br/>3\. Apply the manifest: 
```bash
kubectl apply -f backend-deployment.yaml
```

### 3.3. Set Up Ingress Controller

An ingress controller routes external HTTP/S traffic to services running within the Kubernetes cluster.  
It helps in managing and load-balancing incoming traffic to your backend services.  

<br/>Steps:  

1\. Install the NGINX ingress controller:  
```bash
helm repo add ingress-nginx <https://emea01.safelinks.protection.outlook.com/?url=https%3A%2F%2Fkubernetes.github.io%2Fingress-nginx&data=05%7C02%7C%7Ca9cc823a4387478054bc08dd278ca238%7C84df9e7fe9f640afb435aaaaaaaaaaaa%7C1%7C0%7C638710207140228540%7CUnknown%7CTWFpbGZsb3d8eyJFbXB0eU1hcGkiOnRydWUsIlYiOiIwLjAuMDAwMCIsIlAiOiJXaW4zMiIsIkFOIjoiTWFpbCIsIldUIjoyfQ%3D%3D%7C0%7C%7C%7C&sdata=UxZ9wbpiJ9ReuNqQGvX%2BU2kj99vK2F%2BQY2xpDmaVn3M%3D&reserved=0>  
helm install ingress-nginx ingress-nginx/ingress-nginx
```

<br/>2\. Define an ingress rule for the backend service:  
```yaml
apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
name: backend-ingress  
spec:  
rules:  
- host: api.example.com  
http:  
paths:  
- path: /  
pathType: Prefix  
backend:  
service:  
name: backend-service  
port:  
number: 80
```
 
<br/>3\. Apply the ingress rule:
```bash
kubectl apply -f ingress.yaml
```

## Step 4: Configure the Database

<br/>This step involves setting up the database for storing product data, user accounts, and transaction records.  

### 4.1. Create Azure SQL Database  

This is a relational database that will hold your application data (products, users, orders, etc.).  
Azure SQL Database is a fully managed, scalable relational database with built-in security and performance features.  

<br/>Steps:  

1\. Create an SQL Server:  
```bash
az sql server create --name EcommerceSQLServer --resource-group EcommerceRG \  
--location eastus --admin-user adminuser --admin-password <password>
```
  
<br/>2\. Create a Database:  
```bash
az sql db create --name EcommerceDB --resource-group EcommerceRG \  
--server EcommerceSQLServer --service-objective S1
```

### 4.2. Secure Database with Private Endpoint

A Private Endpoint restricts access to the database, ensuring that it’s not exposed to the public internet.  
This enhances security by limiting access to trusted resources within your network.  

<br/>Steps:  

1\. Create a Private Endpoint for the database: 
```bash
az network private-endpoint create --name SqlPrivateEndpoint --resource-group EcommerceRG \  
--vnet-name EcommerceVNet --subnet DatabaseSubnet --private-connection-resource-id <SQL_RESOURCE_ID>
```

## Step 5: Add Storage for Static Assets  

This step sets up Blob Storage to host product images and other static assets.  

### 5.1. Create Blob Storage  

Blob Storage provides scalable, low-cost storage for large amounts of unstructured data (images, files, etc.).  
It’s optimized for storing large files and objects.  

<br/>Steps: 

1\. Create a Storage Account:  
```bash
az storage account create --name EcommerceStorage --resource-group EcommerceRG --location eastus
```
 
<br/>2\. Create a Blob Container:  
```bash
az storage container create --name images --account-name EcommerceStorage
```

### 5.2. Secure Blob Storage  

Use Shared Access Signatures (SAS) to control access to the storage container.  
SAS tokens provide temporary, secure access without exposing storage keys.  

<br/>Steps: 

1\. Generate a SAS token:  
```bash
az storage blob generate-sas --account-name EcommerceStorage --container-name images \ 
--permissions rwd --expiry 2024-12-31T23:59:00Z
```

## Step 6: Set Up Continuous Integration (CI)

<br/>**What is CI?** 
<br/>Continuous Integration (CI) involves automatically building and testing code every time changes are committed to a repository. It ensures that code changes are integrated smoothly, reducing the chances of integration issues later in the process.  

<br/>**Why is CI Important?**
<br/>It allows developers to continuously deliver high-quality code, catch errors early, and accelerate the release cycle. In an Azure-based e-commerce platform, it helps ensure frontend and backend code is consistently tested and deployed.  

<br/>Steps to Set Up CI for the Frontend (React/Angular) Application:

1\. Create a GitHub Repository (or other Git-based repo):

- Push your frontend code (React/Angular app) to a GitHub repository

2\. Create an Azure DevOps Organization:

- Sign in to Azure DevOps and create an organization.
- Create a project under your organization.

3\. Configure CI Pipeline for Frontend:

- Navigate to Pipelines > New Pipeline.
- Select GitHub as the repository source.
- Configure the pipeline using YAML or the classic editor.

For YAML, here’s an example for a Node.js-based React app: 
```yaml
trigger:  
branches:  
include:  
- main  
pool:  
vmImage: 'ubuntu-latest'  
steps:  
- task: NodeTool@0  
inputs:  
versionSpec: '16.x'  
addToPath: true  
- script: |  
npm install  
npm run build  
displayName: 'Install and build frontend'  
- task: AzureWebApp@1  
inputs:  
appName: 'EcommerceFrontend'  
package: $(System.DefaultWorkingDirectory)/\*\*/build.zip
```

<br/>4\. Configure CI for Backend (Dockerized APIs):

- For the backend, the process involves building Docker images and pushing them to Azure Container Registry (ACR).  

Example YAML pipeline for Docker:  
```yaml
trigger:  
branches:  
include:  
- main  
pool:  
vmImage: 'ubuntu-latest'  
steps:  
- task: Docker@2  
inputs:  
containerRegistry: 'EcommerceACR'  
repository: 'backend'  
command: 'buildAndPush'  
Dockerfile: '$(Build.SourcesDirectory)/Dockerfile'  
tags: 'latest'  
```

## Step 7: Set Up Continuous Deployment (CD)  
 
<br/>**What is CD?**
<br/>Continuous Deployment (CD) automates the deployment of applications to production environments every time changes pass through the CI pipeline.  

<br/>**Why is CD Important?** 
<br/>With CD, updates to the application are automatically deployed, ensuring a smooth release cycle and minimizing manual intervention. For an e-commerce platform, it ensures the latest features or fixes are deployed quickly.  

<br/>Steps to Set Up CD

1\. Link Your CI Pipeline to CD:

- In Azure DevOps, after the CI pipeline runs successfully, create a Release Pipeline to deploy your app.

2\. Deploy Frontend to Azure App Service:

- Use the Azure App Service task in your release pipeline.  
Example steps for deployment:  
```yaml
    - task: AzureWebApp@1  
    inputs:  
    appName: 'EcommerceFrontend'  
    package: $(System.DefaultWorkingDirectory)/\_FrontendApp/build.zip
```

3\. Deploy Backend to AKS (Azure Kubernetes Service):

- For backend services, use kubectl in the CD pipeline to apply updates to the AKS cluster.  
Example:
```yaml
- task: AzureCLI@2  
inputs:  
azureSubscription: 'AzureSubscription'  
scriptType: 'bash'  
scriptLocation: 'inlineScript'  
inlineScript: |  
kubectl set image deployment/backend backend=&lt;ACR_NAME&gt;.azurecr.io/backend:latest
```

## Step 8: Setting Up Application Gateway (WAF)

Steps:

1\. Create a Public IP Address for the gateway:
```bash
az network public-ip create \
--resource-group EcommerceRG \
--name AppGatewayPublicIP \
--sku Standard
```

2\. Deploy Application Gateway in the Frontend Subnet:

- Use the following command:
```bash
az network application-gateway create \
--name AppGateway \
--location eastus \
--resource-group EcommerceRG \
--sku WAF_v2 \
--capacity 2 \
--vnet-name EcommerceVNet \
--subnet FrontendSubnet \
--public-ip-address AppGatewayPublicIP
```

Parameters explained:

**--sku WAF_v2**: Ensures WAF (Web Application Firewall) is enabled.

**--capacity 2**: Configures an autoscaling gateway with two instances.

**--subnet FrontendSubnet**: Deploys the Application Gateway in the frontend subnet.

3\. Set Up Backends in the Application Gateway

The Application Gateway needs to know where to send traffic. You’ll define two backend pools: one for the App Service (frontend) and another for the AKS backend.

- Add Backend Pool for App Service
```bash
az network application-gateway address-pool create \
--gateway-name AppGateway \
--resource-group EcommerceRG \
--name FrontendAppPool \
--backend-addresses <FrontendApp>.azurewebsites.net"
```

Replace <FrontendApp> with the name of your App Service (e.g. EcommerceFrontend).

- Add Backend Pool for AKS
- Obtain the internal IP of the AKS service exposed via a Kubernetes Service:
```bash
kubectl get svc -n <namespace> backend-service
```

Note the **CLUSTER-IP** (if internal). Use this IP address as the backend address.

- Create the backend pool:
```bash
az network application-gateway address-pool create \
--gateway-name AppGateway \
--resource-group EcommerceRG \
--name BackendAppPool \
--backend-addresses <AKS_BACKEND_IP>
```

Replace **<AKS_BACKEND_IP>** with the service’s IP address.

4\. Configure HTTP Settings

HTTP settings define the port and protocol used to connect to the backend.

- Configure HTTP Settings for App Service
```bash
az network application-gateway http-settings create \
--gateway-name AppGateway \
--resource-group EcommerceRG \
--name FrontendAppHttpSettings \
--port 80 \
--protocol Http \
--host-name "<FrontendApp>.azurewebsites.net" \
--pick-hostname-from-backend-address
```

- Configure HTTP Settings for AKS
```bash
az network application-gateway http-settings create \
--gateway-name AppGateway \
--resource-group EcommerceRG \
--name BackendAppHttpSettings \
--port 80 \
--protocol Http
```

5\. Set Up Routing Rules

Routing rules determine how traffic is forwarded to backend pools based on URL paths or hostnames.

- Create a Listener
- Add a listener for HTTPS traffic:
```bash
az network application-gateway http-listener create \
--gateway-name AppGateway \
--resource-group EcommerceRG \
--name HttpsListener \
--frontend-port 443 \
--ssl-cert <CERT_NAME>
```

Replace **<CERT_NAME>** with the SSL certificate name uploaded to the Application Gateway.

- Define Routing Rules
- For frontend (App Service):
```bash
az network application-gateway rule create \
--gateway-name AppGateway \
--resource-group EcommerceRG \
--name FrontendRule \
--http-listener HttpsListener \
--rule-type Basic \
--http-settings FrontendAppHttpSettings \
--address-pool FrontendAppPool
```

- For backend (AKS):
```bash
az network application-gateway rule create \
--gateway-name AppGateway \
--resource-group EcommerceRG \
--name BackendRule \
--http-listener HttpsListener \
--rule-type PathBasedRouting \
--http-settings BackendAppHttpSettings \
--address-pool BackendAppPool \
--paths /api/\*
```

Explanation:

- **FrontendRule** forwards general traffic to the App Service.
- **BackendRule** forwards traffic to /api/\* (e.g., /api/orders, /api/products) to the AKS backend.

## Step 9: Monitoring and Logging

<br/>**What is Monitoring and Logging?**  
<br/>Monitoring involves continuously checking the health, performance, and availability of your application and infrastructure. Logging collects application logs, error messages, and diagnostic data. 

<br/>**Why is Monitoring and Logging Important?** 
<br/>For an e-commerce platform, uptime and smooth user experience are critical. Monitoring and logging help detect issues early, ensuring you can fix problems before they impact customers. 

<br/>Steps to Set Up Monitoring and Logging

1\. Azure Monitor:  

Azure Monitor is the central service for collecting, analyzing, and acting on telemetry data from your resources.  
Set up Application Insights to monitor your application’s performance.
```bash
az monitor app-insights component create --app EcommerceFrontend --location eastus \  
--resource-group EcommerceRG
```

<br/>2\. Log Analytics:  

Use Log Analytics workspaces to aggregate logs and events from Azure resources.  
Example to create a Log Analytics workspace:  
```bash
az monitor log-analytics workspace create --resource-group EcommerceRG \  
--workspace-name EcommerceLogs --location eastus
```
  
<br/>3\. Set up Alerts:  

Set up alerts for critical issues like low disk space, high CPU usage, or failed transactions.  
```bash
az monitor metrics alert create --resource-group EcommerceRG --name HighCPUAlert \ 
--scopes /subscriptions/{subscription-id}/resourceGroups/EcommerceRG/providers/Microsoft.Compute/virtualMachines/EcommerceVM \\  
--condition "avg CPUUsage > 80" --action-group HighCPUActionGroup
```
<br/><br/>4\. Container Monitoring (AKS):  

Use Azure Monitor for containers to track the health and performance of your AKS cluster.  
Enable container logs and metrics collection.  
```bash
az aks enable-addons --resource-group EcommerceRG --name EcommerceAKS --addons monitoring
```

5\. Diagnostic Settings:  
• Enable diagnostic settings on all resources to capture logs and metrics.  
```bash
az monitor diagnostic-settings create --name Diagnostics \\  
--resource /subscriptions/{subscription-id}/resourceGroups/EcommerceRG/providers/Microsoft.Web/sites/EcommerceFrontend \  
--logs '\[{"category": "AppServiceAppLogs", "enabled": true}\]' \  
--metrics '\[{"category": "AllMetrics", "enabled": true}\]' \  
--workspace /subscriptions/{subscription-id}/resourceGroups/EcommerceRG/providers/Microsoft.OperationalInsights/workspaces/EcommerceLogs
```

<br/>Additional Best Practices:

6\. Azure Application Gateway Monitoring:  
\- Use Application Gateway Analytics to monitor traffic to your frontend and backend services.  
\- Configure WAF logs and traffic analytics for real-time visibility.

7\. Alerting and Dashboards:  
\- Create custom dashboards in Azure Monitor to visualize the health and performance of your platform.  
\- Set up alert rules to notify administrators about critical issues (e.g., high latency, failed requests).  

<br/><br/>Conclusion:  

<br/>This guide has covered the end-to-end process for setting up and securing a cloud-based e-commerce platform using Microsoft Azure. We’ve addressed key stages, from environment setup to monitoring, ensuring a smooth and secure deployment process for your application.  
<br/>By following these detailed steps, you’ll have a robust, scalable, and secure platform capable of handling e-commerce workloads, traffic spikes, and sensitive user data.

**Step 8: Secure Your Infrastructure**  
<br/>What is Infrastructure Security?  
Infrastructure security is about protecting your cloud resources (VMs, containers, databases, etc.) from unauthorized access, attacks, or data breaches. It ensures that your application and data are safe.  
<br/>Why is Infrastructure Security Critical?  
In an e-commerce platform, sensitive data such as user information, payment details, and business data must be protected to maintain customer trust and comply with regulations (e.g., GDPR, PCI DSS).  
<br/>Steps to Secure Infrastructure  

1\. Identity and Access Management (IAM):  
• Azure Active Directory (AAD) is the core service to manage users and roles.  
• Assign Role-Based Access Control (RBAC) to limit access to resources based on the user’s role.  
Example of assigning roles:  
<br/>az role assignment create --assignee &lt;user-email&gt; --role "Contributor" --resource-group EcommerceRG  

2\. Network Security:  
• Ensure Network Security Groups (NSGs) are properly configured for each subnet (frontend, backend, database).  
<br/>3\. Data Encryption:  
• Enable encryption at rest and in transit for sensitive data:  
• SQL Databases: Enable Transparent Data Encryption (TDE).  
• Blob Storage: Enable Storage Service Encryption (SSE).  
4\. Private Endpoints:  
• Use private endpoints for services like Azure SQL Database to ensure that they are not accessible over the internet, only within the VNet.  
5\. Web Application Firewall (WAF):  
• Set up Azure Application Gateway with WAF to filter malicious HTTP requests.  
7\. DDoS Protection:  
• Enable Azure DDoS Protection for your resources to defend against Distributed Denial-of-Service attacks.
