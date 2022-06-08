# Hands-on

## Prerequisites
- Docker ([Get Docker](https://docs.docker.com/get-docker/))
- Azure CLI ([Get az CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli))
## Instructions
1. Clone the repository
2. Open the project in VS Code (or your preferred IDE)
3. Create a file named `Dockerfile` in the root directory of your project
4. To build the image, the Dockerfile should contain the following instructions ([docs](https://docs.docker.com/engine/reference/builder))
    - Set the base image to `node:12-alpine`
    - Run the command `apk add --no-cache python2 g++ make` to install some packages
    - Set the working directory to `/app` for any following command in the dockerfile
    - Copy all files from the current context of the Dockerfile to the current working directory
    - Run the command `yarn install --production` to install all the application dependencies
    - Set the default command for a starting container to `node src/index.js`
    - the running container should listen on port 3000
5. Build the Image and tag it with "th-vorlesung:1.0" (in case you are building on ARM use `docker buildx` with `--platform linux/amd64` flag)
6. Create an ACR in Azure
    - `az login` & `az account show` to check if you are in the right subscription
    - Create a resource group with `az group create --name container-vorlesung-<shortname> --location westeurope`
    - Create an ACR with `az acr create --resource-group container-vorlesung-<shortname> --name container0vorlesung0<shortname> --sku Basic --admin-enabled`
    - log in to the ACR after creation with `az acr login --name container0vorlesung0<shortname>`
    - check the output and write down the "loginServer" value
7. Re-tag the image with `<loginServer>/th-vorlesung:1.0`
8. Push the image to the ACR
9. Deploy the image to Azure Container Apps.
    - (Optional) Execute the following commands to upgrade your Azure CLI
      - `az extension add --name containerapp --upgrade`
    - Create an ACA environment with `az containerapp env create --name th-vorlesung-aca --resource-group container-vorlesung-<shortname> --location westeurope`
    - Deploy your image to the ACA environment (find the credentials of your ACR in Azure) with `az containerapp create --name <name> --resource-group <resourceGroupName> --environment <AcaEnvName> --image <imageName:tag> --target-port <port> --ingress 'external' --query properties.configuration.ingress.fqdn --registry-server <loginServer> --registry-username <loginUsername> --registry-password <loginPassword>`
    - Copy the URL from the output and verify that your app is online
