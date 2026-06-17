# Containers from scratch

This repo provides a demo on how to build containers from scratch.

The demo does not build a production-ready container runtime. Instead, it shows
the Linux building blocks behind containers: root filesystems, `chroot`,
namespaces, and OverlayFS layers.

## Setup

Spin up a Linux virtual machine on Azure:

```bash
az group create \
  --name container-demo-rg \
  --location westeurope

az vm create \
  --resource-group container-demo-rg \
  --location westeurope \
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

Finally, install Docker and prepare an Ubuntu root filesystem for the demo:

```bash
sudo apt-get update

# Install tools required for Docker setup and for inspecting namespaces/processes.
sudo apt-get -y install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    iproute2 \
    procps \
    util-linux

# Create the keyring directory with explicit permissions:
# - install creates the directory when used with -d
# - -m 0755 sets permissions to rwxr-xr-x
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's repository signing key and make it readable by apt.
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker's official apt repository for the current Ubuntu release.
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker Engine, the CLI, containerd, Buildx, and Docker Compose.
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify that the important demo tools are available.
docker --version
unshare --version
ip -Version
ps --version

# Create a container from the Ubuntu image and install the tools that should be
# available inside the later chroot/container-like environment.
sudo docker container run --name ubuntu-fs ubuntu:latest sh -c "\
  apt-get update && \
  apt-get install -y iproute2 procps util-linux hostname && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/*"

mkdir -p ./filesystem

# Export the container filesystem and extract it into ./filesystem.
sudo docker container export ubuntu-fs | tar xf - -C ./filesystem
sudo docker container rm ubuntu-fs

# Ensure /proc exists for the later chroot + namespace demo.
mkdir -p ./filesystem/proc
```

## Hands-On

Login to the virtual machine via SSH:

```bash
publicIp=$(az vm show -d -g container-demo-rg -n container-demo-vm --query publicIps -o tsv)

ssh azureuser@$publicIp
```

### A "normal" process

First, create a new shell process:

```bash
sh
```

Now have a look at the filesystem, processes, and network:

```bash
ps -ax
ls -lisa /
ip address show
```

This is a normal host process. It sees the host filesystem, host processes, and
host network interfaces.

Then exit the process:

```bash
exit
```

### chroot command

`chroot` changes the root directory for a process. This is useful, but it is not
a container by itself because it does not isolate processes, networking, or other
kernel resources.

Create a file in the prepared root filesystem and start a shell inside it:

```bash
echo "hello" > ./filesystem/our-test-file.txt
sudo chroot ./filesystem sh
```

Look at the filesystem:

```bash
ls -lisa /
cat /our-test-file.txt
```

The root directory is now `./filesystem`, so `/our-test-file.txt` is visible.

Mount `proc` and inspect processes:

```bash
mount -t proc proc /proc

ps -ax
```

You can still see host processes. This demonstrates that `chroot` only changes
the filesystem view.

Clean up the `proc` mount and exit:

```bash
umount /proc
exit
```

### unshare command

Linux namespaces isolate process views, hostnames, IPC, networking, and mount
points. Start a shell in new namespaces:

```bash
sudo unshare --pid --fork --uts --ipc --net --mount-proc /bin/sh
```

The flags mean:

| Flag | Meaning |
| --- | --- |
| `--pid` | Creates a new PID namespace. The shell gets its own process tree. |
| `--fork` | Starts the shell as a child process, which lets it become PID 1 in the new PID namespace. |
| `--uts` | Creates a new UTS namespace, so the hostname can be changed independently. |
| `--ipc` | Creates a new IPC namespace for isolated inter-process communication resources. |
| `--net` | Creates a new network namespace with a separate network stack. |
| `--mount-proc` | Mounts a new `/proc` view for the new PID namespace, so tools like `ps` show the isolated process list. |
| `/bin/sh` | Starts a shell inside the new namespaces. |

Now try the following commands:

```bash
ps -ax

ip address show

hostname
hostname another-name
hostname
```

You should see a much smaller process list, an isolated network namespace, and a
hostname that can be changed without changing the VM hostname.

Finally exit the process:

```bash
exit
```

### chroot and unshare together

Now combine the isolated root filesystem with Linux namespaces. This is much
closer to the core idea of a container:

```bash
sudo unshare --pid --fork --uts --ipc --net --mount-proc="$(pwd)/filesystem/proc" chroot ./filesystem /bin/sh
```

This uses the same namespace flags as before, but adds `chroot`:

| Part | Meaning |
| --- | --- |
| `--mount-proc="$(pwd)/filesystem/proc"` | Mounts the new `/proc` view inside the prepared root filesystem instead of the host `/proc`. |
| `chroot ./filesystem` | Makes `./filesystem` appear as `/` for the started process. |
| `/bin/sh` | Starts the shell from inside the new root filesystem. |

Inspect the environment again:

```bash
ps -ax
ls -lisa /
cat /our-test-file.txt
ip address show
hostname
hostname mini-container
hostname
```

This shell has its own process namespace, network namespace, hostname namespace,
and root filesystem. It is still not a complete container runtime, but it shows
the most important primitives.

Exit the shell:

```bash
exit
```

### OverlayFS layers

Container images are built from layers. OverlayFS can combine a read-only lower
layer with a writable upper layer into one merged view.

Create a lower layer, an upper layer, a work directory, and a merged directory:

```bash
mkdir -p ./fs/lower ./fs/upper ./fs/work ./fs/merged

touch ./fs/lower/lower-file.txt
touch ./fs/upper/upper-file.txt

echo "hello lower" > ./fs/lower/content.txt
echo "hello upper" > ./fs/upper/content.txt

sudo mount -t overlay overlay -o lowerdir=./fs/lower,upperdir=./fs/upper,workdir=./fs/work ./fs/merged

ls -lisa ./fs/merged/
cat ./fs/merged/content.txt
```

The mount command parts mean:

| Part | Meaning |
| --- | --- |
| `mount` | Mounts a filesystem. |
| `-t overlay` | Selects OverlayFS as the filesystem type. |
| `overlay` | Source name for the mount. For OverlayFS this is conventionally just `overlay`. |
| `-o ...` | Passes OverlayFS-specific mount options. |
| `lowerdir=./fs/lower` | Lower layer. This acts like a read-only base image layer. |
| `upperdir=./fs/upper` | Writable upper layer. New and changed files are stored here. |
| `workdir=./fs/work` | Required working directory used internally by OverlayFS. |
| `./fs/merged` | Mount point where the combined view appears. |

The merged directory contains files from both layers. If the same file exists in
both layers, the upper layer wins, so `content.txt` contains `hello upper`.

Create a file through the merged view and check where it appears:

```bash
echo "created in merged" > ./fs/merged/merged-file.txt

ls -lisa ./fs/merged/
ls -lisa ./fs/upper/
ls -lisa ./fs/lower/
```

New files are written to the upper layer. This is similar to how a running
container gets a writable layer on top of image layers.

Now unmount the overlay volume again:

```bash
sudo umount ./fs/merged
```

## Cleanup

Remove demo files from the VM:

```bash
rm -rf ./filesystem ./fs
```

If you also want to delete the Azure resources after the demo, run this command
from your local machine:

```bash
az group delete --name container-demo-rg
```

And there is even more...
