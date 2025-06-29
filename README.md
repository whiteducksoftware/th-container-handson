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
    - Run the command `apk add --no-cache python3 g++ make` to install some packages
    - Set the working directory to `/app` for any following command in the dockerfile
    - Copy all files from the current context of the Dockerfile to the current working directory
    - Run the command `yarn install --production` to install all the application dependencies
    - Set the default command for a starting container to `node src/index.js`
    - the running container should listen on port 3000
5. Build the Image and tag it with "getting-started:1.0" (in case you are building on ARM use `--platform linux/amd64` flag)
6. Create an ACR in Azure
    - `az login` & `az account show` to check if you are in the right subscription
    - Create a resource group with `az group create --name container-gettingstarted-<shortname> --location westeurope --tags purpose=handson` (replace `<shortname>` with your initials or a short name)
    - Create an ACR with `az acr create --resource-group container-gettingstarted-<shortname> --name container0gettingstarted0<shortname> --sku Basic --admin-enabled`
    - check the output and write down the "loginServer" value (we will need it later)
    - log in to the ACR after creation with `az acr login --name container0gettingstarted0<shortname>`
7. Re-tag the image with `<loginServer>/getting-started:1.0`
8. Push the image to the ACR (`docker push` command)
9. Deploy the image to Azure Container Apps.
    - (Optional) Execute the following commands to upgrade your Azure CLI
      - `az extension add --name containerapp --upgrade`
    - Create an ACA environment with `az containerapp env create --name getting-started-aca --resource-group container-gettingstarted-<shortname> --location westeurope`
    - Deploy your image to the ACA environment (find the credentials of your ACR in Azure) with `az containerapp create --name <name> --resource-group <resourceGroupName> --environment <AcaEnvName> --image <loginServer>/<imageName:tag> --target-port <port> --ingress 'external' --query properties.configuration.ingress.fqdn --registry-server <loginServer> --registry-username <loginUsername> --registry-password <loginPassword>`
    - Copy the URL from the output and verify that your app is online

## Optional Steps

- Try to build a more secure Dockerfile (think about Multi-Stage Builds, using a non-root user, etc.)
- Try to deploy the Azure Container App with Infrastructure as Code (IaC) using Terraform.

# Demo

Find all demos in the [demo directory](./demo/).
