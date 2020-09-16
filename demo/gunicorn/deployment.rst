****************************************
Deployment of TRX Server on Azure Cloud
****************************************

.. role:: bash(code)
   :language: bash

Updated Dockerfile
==================
- I noticed space for some improvements in the Dockerfile.
- There is no need to mention complete path when `WORKDIR` is mentioned.
- As we are using a ubuntu 18:04 image, it is better to update it with all the security patches and features.
- All the requirements to run the application should be inside the `requirements.txt` file.
- Only necessary files should be copied inside the container.
- Relevant port should be exposed.
- The application should be launched via the gunicorn `WSGI`.

The Task
=========
- The task was to deploy and host the Maltego TRX server on Azure cloud.
- The `URL <http://acr-tasks-maltegotrx.westeurope.azurecontainer.io:8080/>`_ for the deployed service is also added.
- The deployment guide assumed you have Azure cloud credentials and have installed CLI tool on the machine where the code is being checked out.
- Please clone the repository from the `Github URL <https://github.com/aliabbasjaffri/maltego-trx/tree/azure-acr-build>`_, where i have made the changes.
- Login to your Azure Cloud CLI tool via code :bash:`az --login` which navigates to webpage to login.
- Once the login to Azure Cloud CLI is successful, please follow the following steps:

 .. code-block::bash
 
 # ``Check the version of Azure Cloud CLI. The version must be greater than 2.10.0``

 :bash:`$ az --version`
 
 # ``Login to Azure Cloud CLI``
 
 :bash:`$ az --login`
 
 # ``Setup ENV VARS needed to name the services``
 
 | :bash:`ACR_NAME=maltegotrx`
 | :bash:`ACR_TASKS=acr-tasks`
 | :bash:`LOCATION=westeurope`
 | :bash:`IMAGE_NAME=maltegotrx-server`
 | :bash:`RES_GROUP=$ACR_NAME`
 | :bash:`AKV_NAME=$ACR_NAME-vault`
 | :bash:`EXPOSE_PORT=8080`
 
 
 # ``Create Azure Resource group``

 :bash:`$ az group create --location $LOCATION --name $RES_GROUP`
 
 # ``Create Azure Container registry``

 :bash:`$ az acr create --resource-group $RES_GROUP --name $ACR_NAME --sku Basic --location $LOCATION`
 
 # ``Login to Azure Container Registry``
 
 :bash:`$ az acr login --name $ACR_NAME`
 
 # ``Build the image inside the registry``

 :bash:`$ az acr build --registry $ACR_NAME --image $IMAGE_NAME:v1 .`
 
 # ``Create an Azure Key Vault to store credentials``

 :bash:`$ az keyvault create --resource-group $RES_GROUP --name $AKV_NAME`
 
 # ``Create service principals for registry pull and store them inside the Key Vault``

 $ az keyvault secret set                                                                                                 \
   | --vault-name $AKV_NAME                                                                                               \
   | --name $ACR_NAME-pull-pwd                                                                                            \
   | --value $(az ad sp create-for-rbac                                                                                   \
   | --name $ACR_NAME-pull                                                                                                \
   | --scopes $(az acr show --name $ACR_NAME --query id --output tsv)                                                     \
   | --role acrpull                                                                                                       \
   | --query password                                                                                                     \
   | --output tsv)
 
 # ``Store the service principal's `appid` in the Key Vault``

 $ az keyvault secret set                                                                                                 \
   | --vault-name $AKV_NAME                                                                                               \
   | --name $ACR_NAME-pull-usr                                                                                            \
   | --value $(az ad sp show --id http://$ACR_NAME-pull --query appId --output tsv)

 # ``Create container in resourcegroup, from ACR with image name``

 $ az container create                                                                                                    \
   | --resource-group $RES_GROUP                                                                                          \
   | --name $ACR_TASKS                                                                                                    \
   | --image $ACR_NAME.azurecr.io/$IMAGE_NAME:v1                                                                          \
   | --registry-login-server $ACR_NAME.azurecr.io                                                                         \
   | --ports $EXPOSE_PORT                                                                                                 \
   | --registry-username $(az keyvault secret show --vault-name $AKV_NAME --name $ACR_NAME-pull-usr --query value -o tsv) \
   | --registry-password $(az keyvault secret show --vault-name $AKV_NAME --name $ACR_NAME-pull-pwd --query value -o tsv) \
   | --dns-name-label acr-tasks-$ACR_NAME                                                                                 \
   | --query "{FQDN:ipAddress.fqdn}"                                                                                      \
   | --output table
 
 # ``Bind your local console's STDOUT and STDERR to that of the container's to check for logs.``

 :bash:`$ az container attach --resource-group $RES_GROUP --name $ACR_TASKS`
 