# Containers from scratch

This repo provides a demo on how to build containers from scratch.

## Setup

Spin up a Linux virtual machine on Azure:

```bash
az group create -g container-demo-rg

az vm create \
  --resource-group container-demo-rg \
  -l westeurope \
  --name container-demo-vm \
  --image Ubuntu2404 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard \
  --public-ip-address-allocation static
```

You can now login to the virtual machine via SSH:

```bash
publicIp=$(az vm show -d -g container-demo-rg -n container-demo-vm --query publicIps -o tsv)

ssh azureuser@$publicIp
```

Finally you need to configure some stuff:

```bash
sudo apt-get update
sudo apt-get -y install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo docker container run --name ubuntu-fs ubuntu:latest

mkdir -p ./filesystem
sudo docker container export ubuntu-fs | tar xfx - -C ./filesystem
```

## Hands-On

Login to the virtual machine via SSH:

```bash
publicIp=$(az vm show -d -g container-demo-rg -n container-demo-vm --query publicIps -o tsv)

ssh azureuser@$publicIp
```

### A "normal" process

First, create a new Shell process by executing `sh`. Now have a look at the filesystem, processes and network. Some ideas:

```bash
ps -ax
ls -lisa /
ip address show
```

Then exit the process by executing `exit`.

### chroot command

Let's play with `chroot` and change some files in the filesystem:
```
echo "hello" > ./filesystem/our-test-file.txt
sudo chroot ./filesystem sh
```

But you are still able to see all processes:

```bash
mount -t proc proc /proc

ps -ax
```

Now have a look at the filesystem. Afterwards, exit the process with `exit`.

### unshare command

And now with Linux namespaces and `unshare`:
```
sudo unshare --pid --fork --uts --ipc --net --mount-proc /bin/sh
```

Now you can try following commands:

```bash
ps -ax

ip address show

hostname
hostname another-name
hostname
```

Finally exit the process with `exit`.

How layered container images work. Create files in the merged space and check the other directores:
```
mkdir -p  ./fs/lower ./fs/upper ./fs/work ./fs/merged

touch ./fs/lower/lower-file.txt
touch ./fs/upper/upper-file.txt

echo "hello lower" > ./fs/lower/content.txt
echo "hello upper" > ./fs/upper/content.txt

sudo mount -t overlay overlay -o lowerdir=./fs/lower,upperdir=./fs/upper,workdir=./fs/work ./fs/merged

ls -lisa ./fs/merged/
cat ./fs/merged/content.txt
```

Now unmount the overlay volume again:

```bash
sudo umount overlay
```

And there is even more...
