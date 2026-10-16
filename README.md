# Making NVidia GPUs available in your kubernetes cluster

Getting NVidia GPUs to work in your Kubernetes cluster is fairly straightforward nowadays, it used to be much messier.

## The background

For this tutorial I assume you have the following:

 * A Working K3S cluster
 * At least one node with some sort of NVidia GPU
 * Helm installed

### Tested Linux Distros

This guide has been tested on:

 * Debian 13

Some of the instructions necessary for Talos Linux can be found [here](https://docs.siderolabs.com/talos/v1.14/configure-your-talos-cluster/hardware-and-drivers/nvidia-gpu-proprietary), when you get to the section labelled "Deploying NVIDIA GPU Operator" return here carrying on at the [Node Feature Discovery](#node-feature-discovery) section.

## Configuring repositories

The first step is to make sure the OS itself can interface with your GPU:

### Debian 13 

#### Standard installation

If you installed Debian normally you would want to edit `/etc/apt/sources.list`, the first line is likely the correct one to edit:

```sh
sudo nano /etc/apt/sources.list
deb http://mirror.csclub.uwaterloo.ca/debian/ trixie main
```

Stick `contrib non-free non-free-firmware` onto the end so that it looks something like this:

```sh
deb http://mirror.csclub.uwaterloo.ca/debian/ trixie main non-free-firmware contrib
```

Press Ctrl+X to save, then Y to confirm and then enter.

#### Cloud Image

```sh
sudo sed -i 's/^Components: main$/Components: main contrib non-free non-free-firmware/' /etc/apt/sources.list.d/debian.sources
```

#### Installing the GPU driver

```sh
sudo apt-get update
sudo apt-get install nvidia-driver nvidia-smi
```

You will get a warning about the nouveau driver, the open source driver for nvidia gpus. Just press okay.

### Ubuntu 24.04

Installing the NVidia drivers on Ubuntu is very easy:

```sh
sudo apt install nvidia-dkms-590-server nvidia-utils-590-server
```

## Installing the NVidia Container Toolkit

Now we'll install gpg and add the nvidia container toolkit repository:

```sh
sudo apt-get install -y gpg curl
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update
sudo apt-get install firmware-misc-nonfree  nvidia-container-runtime
```

And now we reboot:

```sh
sudo reboot
```

## Testing

If you want to make sure everything is working, log back in and run `nvidia-smi`. This is an optional utility to see what's going on with your GPU(s):

```sh
nvidia-smi
Fri Oct 16 18:30:35 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.91.07              Driver Version: 595.91.07      CUDA Version: 13.2     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA T400                    Off |   00000000:01:00.0 Off |                  N/A |
| 38%   39C    P8            N/A  /   31W |       1MiB /   2048MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------
```

No more needs to be done directly on your Kubernetes nodes.

## Node Feature Discovery

Node Feature Discovery adds labels to your nodes based on what hardware is available and what features your CPU(s) support. The NVidia GPU Resource Driver needs these labels for scheduling.

```sh
helm repo add node-feature-discovery https://kubernetes-sigs.github.io/node-feature-discovery/charts
helm repo update
helm upgrade --install node-feature-discovery node-feature-discovery/node-feature-discovery --create-namespace --namespace node-feature-discovery
```

## Installing the NVidia GPU Resource Driver

Next we'll install the NVidia GPU Resource Driver using helm.

```sh
helm repo add gpu-operator https://helm.ngc.nvidia.com/nvidia
helm repo update
helm upgrade --install gpu-operator/dra-driver-nvidia-gpu --create-namespace --namespace nvidia --values values.yaml
```

> [!NOTE]
> If you're running Talos Linux you'll want to edit the `values.yaml` file and uncomment the noted line.

I highly recommend using [K9S](https://k9scli.io/) to see if everything's going well. The whole setup process will take a minute or two to start up. If everything has gone well all the pods should be Running:

```sh
kubectl get pods -n nvidia
NAME                                                READY   STATUS    RESTARTS   AGE
dra-driver-nvidia-gpu-controller-7d8fb4bdc4-l5962   1/1     Running   0          134m
dra-driver-nvidia-gpu-kubelet-plugin-8ctxf          2/2     Running   0          60m
```

Next we'll install the custom runtime class needed:

```sh
kubectl apply -f runtime.yaml
```

The last piece we need is a ResourceClaimTemplate that your pods can use:

```sh
kubectl apply -f resourceClaimTemplate.yaml
```

> [!NOTE]
> ResourceClaimTemplates are namespaced objects. If your pods that you want to have use your NVidia GPU
> are in different namespaces you'll want a ResourceClaimTemplate for each namespace.

## Making sure it all works

Let's see if it works:

```sh
kubectl apply -f nvidia-smi.yaml
```

Let's see if it worked:

```sh
kubectl logs nvidia-smi
Fri Oct 16 22:39:10 2026       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.91.07              Driver Version: 595.91.07      CUDA Version: 13.2     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA T400                    Off |   00000000:01:00.0 Off |                  N/A |
| 38%   39C    P8            N/A  /   31W |       1MiB /   2048MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------
```

It worked! You can remove the test:

```sh
kubectl delete -f nvidia-smi.yaml
```

## Letting other containers use NVidia GPUs

Getting other containers to use NVidia GPUs is fairly easy. Look at the contents of jellyfin.yaml, specifically lines 24-26 and 31-33. Lines 24-26 imports the ResourceClaim created by the ResourceClaimTemplate into the pod and lines 31-33 assign the claim to the container in the pod.

## Other useful links:

If you want to easily spin up virtual machines that auto-install the NVidia driver and toolkit checkout my [Ubuntu-CloudInit-Docs](https://github.com/UntouchedWagons/Ubuntu-CloudInit-Docs) repository.

Check out my guide for using Intel GPUs in Kubernetes: https://github.com/UntouchedWagons/K3S-Intel
