# Containerization Hands-on

In this hands-on, you containerize a small Node.js todo application, publish the
container image to Azure Container Registry (ACR), and run it on Azure Container
Apps (ACA).

## Prerequisites

- Docker ([Get Docker](https://docs.docker.com/get-docker/))
- Azure CLI ([Get az CLI](https://learn.microsoft.com/cli/azure/install-azure-cli))
- Git
- An Azure account with an active subscription
- Permissions in Azure to create:
    - a Resource Group
    - an Azure Container Registry
    - an Azure Container Apps environment
    - an Azure Container App
- Optional: VS Code or another IDE

> This hands-on uses the ACR admin username and password to keep the exercise
> simple. This is acceptable for the classroom scenario, but for production
> workloads you should prefer managed identity or another least-privilege access
> pattern.

The Azure resources created in this exercise can generate costs. Delete the
resource group at the end if you no longer need it.

## Goal

By the end of the hands-on, you should understand the basic container workflow:

1. Create a Dockerfile for an application.
2. Build and run a container image locally.
3. Tag and push the image to a container registry.
4. Deploy the image as a managed container application in Azure.
5. Verify the running application through a public URL.

## Instructions

1. Clone the repository.
2. Open the project in VS Code or your preferred IDE.
3. Create a file named `Dockerfile` in the root directory of the project.
4. To build the image, the Dockerfile should contain the following instructions ([docs](https://docs.docker.com/engine/reference/builder)):
    - Set the base image to a supported Node.js LTS Alpine image, for example `node:22-alpine`
    - Run the command `apk add --no-cache python3 g++ make` to install some packages
    - Set the working directory to `/app` for any following command in the Dockerfile
    - Copy all files from the current context of the Dockerfile to the current working directory
    - Enable Corepack if needed so that `yarn` is available
    - Run the command `yarn install --production` to install all the application dependencies
    - Set the default command for a starting container to `node src/index.js`
    - Expose port `3000`
5. Build the image and tag it with `getting-started:1.0`.
    - On ARM machines, add `--platform linux/amd64` if you want to build an AMD64 image for Azure.
6. Run and verify the image locally.
    - Start the container and publish container port `3000` to your local machine.
    - Open the application in your browser, for example at `http://localhost:3000`.
7. Create an ACR in Azure.
    - Sign in with `az login`.
    - Check that you are using the correct subscription with `az account show`.
    - Create a resource group named `container-gettingstarted-<shortname>` in `westeurope`.
      Replace `<shortname>` with your initials or another short unique value.
    - Create an ACR named `container0gettingstarted0<shortname>` in the resource group.
      Use the `Basic` SKU and enable the admin user.
    - Check the output and write down the `loginServer` value. You need it later.
    - Log in to the ACR with `az acr login`.
8. Re-tag the local image with `<loginServer>/getting-started:1.0`.
9. Push the re-tagged image to the ACR with `docker push`.
10. Deploy the image to Azure Container Apps.
    - Make sure the Container Apps extension for Azure CLI is available or upgraded.
    - Create an ACA environment named `getting-started-aca` in the resource group.
    - Find the ACR admin username and password in Azure.
    - Create an Azure Container App that uses:
        - image `<loginServer>/getting-started:1.0`
        - target port `3000`
        - external ingress
        - registry server `<loginServer>`
        - the ACR admin username and password
    - Copy the fully qualified domain name from the command output and verify that your app is online.

## Example command overview

Replace `<shortname>`, `<loginServer>`, `<acrUsername>`, and `<acrPassword>` with
your own values.

```bash
docker build -t getting-started:1.0 .
docker run --rm -p 3000:3000 getting-started:1.0
```

```bash
az login
az account show

az group create \
  --name container-gettingstarted-<shortname> \
  --location westeurope \
  --tags purpose=handson

az acr create \
  --resource-group container-gettingstarted-<shortname> \
  --name container0gettingstarted0<shortname> \
  --sku Basic \
  --admin-enabled true

az acr login --name container0gettingstarted0<shortname>
```

```bash
docker tag getting-started:1.0 <loginServer>/getting-started:1.0
docker push <loginServer>/getting-started:1.0
```

```bash
az extension add --name containerapp --upgrade

az containerapp env create \
  --name getting-started-aca \
  --resource-group container-gettingstarted-<shortname> \
  --location westeurope

az containerapp create \
  --name getting-started-app \
  --resource-group container-gettingstarted-<shortname> \
  --environment getting-started-aca \
  --image <loginServer>/getting-started:1.0 \
  --target-port 3000 \
  --ingress external \
  --registry-server <loginServer> \
  --registry-username <acrUsername> \
  --registry-password <acrPassword> \
  --query properties.configuration.ingress.fqdn
```

## Expected result

After deployment, Azure CLI prints the public FQDN of the Container App. Open the
URL in a browser. You should see the todo application and be able to add, update,
and delete todo items.

## Cleanup

Delete the resource group when you no longer need the resources:

```bash
az group delete --name container-gettingstarted-<shortname>
```

## Optional Steps

- Try to build a more secure Dockerfile (think about Multi-Stage Builds, using a non-root user, etc.)
- Try to add a `.dockerignore` file so files such as `.git`, `node_modules`, logs, and local databases are not copied into the image build context.
- Try to deploy the Azure Container App with Infrastructure as Code (IaC) using Terraform.

## Demo materials

Additional instructor demos are available in the [demo directory](./demo/). They
are separate from the main student hands-on above.
