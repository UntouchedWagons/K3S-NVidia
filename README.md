# Making NVidia GPUs available in your kubernetes cluster

Getting NVidia GPUs to work in your kubernetes cluster has been in my experience a hot mess with conflicting documentation and helm charts. Through the power of chutzpah and dumb luck I managed to get a tutorial made to provide a working setup.

## The background

For this tutorial I assume you have the following:

 * A Working K3S cluster with at least one node with some sort of NVidia GPU
 * Helm installed

### Tested Linux Distros

This guide has been tested on:

 * Debian 13
 * Ubuntu 24.04

Instructions for Talos Linux can be found here: https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/hardware-and-drivers/nvidia-gpu-proprietary Do note that you need to install [Node Feature Discovery](https://artifacthub.io/packages/helm/node-feature-discovery/node-feature-discovery) for the NVidia helm chart to work properly

## Configuring repositories

The first step is to make sure the OS itself can interface with your GPU:

### Debian 13 

#### Standard installation

If you installed Debian normally you would want to edit `/etc/apt/sources.list`, the first line is likely the correct one to edit:

    sudo nano /etc/apt/sources.list
    deb http://mirror.csclub.uwaterloo.ca/debian/ trixie main

Stick `contrib non-free non-free-firmware` onto the end so that it looks something like this:

    deb http://mirror.csclub.uwaterloo.ca/debian/ trixie main non-free-firmware contrib

Press Ctrl+X to save, then Y to confirm and then enter.

#### Cloud Image

    sudo sed -i 's/^Components: main$/Components: main contrib non-free non-free-firmware/' /etc/apt/sources.list.d/debian.sources

#### Installing the GPU driver

    sudo apt-get update
    sudo apt-get install nvidia-driver nvidia-smi

You will get a warning about the nouveau driver, the open source driver for nvidia gpus. Just press okay.

### Ubuntu 24.04

Installing the NVidia drivers on Ubuntu is very easy:

    sudo apt install nvidia-dkms-590-server nvidia-utils-590-server

## Installing the NVidia Container Toolkit

Now we'll install gpg and add the nvidia container toolkit repository:

    sudo apt-get install -y gpg curl
    curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
    curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    sudo apt-get update
    sudo apt-get install firmware-misc-nonfree  nvidia-container-runtime

And now we reboot:

    sudo reboot

If you want to make sure everything is working, log back in and run `nvidia-smi`. This is an optional utility to see what's going on with your GPU(s):

    nvidia-smi
    Fri Jan 30 01:41:02 2026       
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 550.163.01             Driver Version: 550.163.01     CUDA Version: 12.4     |
    |-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  Quadro P400                    Off |   00000000:06:10.0 Off |                  N/A |
    | 40%   53C    P8             N/A /  N/A  |       2MiB /   2048MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+
                                                                                            
    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |  No running processes found                                                             |
    +-----------------------------------------------------------------------------------------+

No more needs to be done directly on your Kubernetes nodes.

## Install Node Feature Discovery

Node Feature Discovery adds labels to your nodes based on what hardware is available and what features your CPU(s) support. The NVidia Device Plugin needs these labels for scheduling.

    helm repo add node-feature-discovery https://kubernetes-sigs.github.io/node-feature-discovery/charts
    helm upgrade --install node-feature-discovery node-feature-discovery/node-feature-discovery --create-namespace --namespace node-feature-discovery

## Installing the K8s Device Plugin

Next we'll install the NVidia K8s Device Plugin using helm. Helm is like apt-get but for Kubernetes.

    helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
    helm repo update

By default if you have only one GPU installed only one pod can use that resource. In `values.yaml`, line 18 tells the device plugin to treat that one GPU as if it were actually four GPUs.

And now we will install the K8s Device Plugin:

    helm upgrade --install nvdp nvdp/nvidia-device-plugin --create-namespace --namespace nvidia --values values.yaml

I highly recommend using [K9S](https://k9scli.io/) to see if everything's going well. The whole setup process will take a minute or two to start up. If everything has gone well all the pods should be Running and all the pods should be Running:

    kubectl get pods -n nvidia
    NAME                                                    READY   STATUS    RESTARTS   AGE
    nvdp-node-feature-discovery-gc-5f949b6d-zhkg4           1/1     Running   0          18m
    nvdp-node-feature-discovery-master-b85494b56-qhwx4      1/1     Running   0          18m
    nvdp-node-feature-discovery-worker-72fck                1/1     Running   0          18m
    nvdp-nvidia-device-plugin-cbfz4                         2/2     Running   0          18m
    nvdp-nvidia-device-plugin-gpu-feature-discovery-t9dxw   2/2     Running   0          18m

## Making sure it all works

Let's see if it works:

    kubectl apply -f nvidia-smi.yaml

Let's see if it worked:

    kubectl logs nvidia-smi
    Sun Mar  8 02:20:49 2026       
    +-----------------------------------------------------------------------------------------+
    | NVIDIA-SMI 550.163.01             Driver Version: 550.163.01     CUDA Version: 12.5     |
    |-----------------------------------------+------------------------+----------------------+
    | GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
    | Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
    |                                         |                        |               MIG M. |
    |=========================================+========================+======================|
    |   0  Quadro P400                    On  |   00000000:01:00.0 Off |                  N/A |
    | 46%   59C    P8             N/A /  N/A  |       2MiB /   2048MiB |      0%      Default |
    |                                         |                        |                  N/A |
    +-----------------------------------------+------------------------+----------------------+
                                                                                            
    +-----------------------------------------------------------------------------------------+
    | Processes:                                                                              |
    |  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
    |        ID   ID                                                               Usage      |
    |=========================================================================================|
    |  No running processes found                                                             |
    +-----------------------------------------------------------------------------------------+

It worked! You can remove the test:

    kubectl delete -f nvidia-smi.yaml

## Letting other containers use NVidia GPUs

Getting other containers to use NVidia GPUs is fairly easy. Look at the contents of jellyfin.yaml, specifically lines 23 and 27-29. Line 23 tells Kubernetes to use the nvidia runtime instead of the default containerd and lines 27 to 29 set GPU limits. The nvidia runtime will automatically copy everything needed for your pod to use the GPU.

## Sources:

  * https://docs.siderolabs.com/talos/v1.12/configure-your-talos-cluster/hardware-and-drivers/nvidia-gpu-proprietary
  * https://github.com/mirceanton/home-ops/blob/6c8f7e23d7531489eaa273500fa37fbb2ea523a2/apps/kube-system/nvidia-device-plugin/app/helm-release.yaml

## Other useful links:

If you want to easily spin up virtual machines that auto-install the NVidia driver and toolkit checkout my [Ubuntu-CloudInit-Docs](https://github.com/UntouchedWagons/Ubuntu-CloudInit-Docs) repository.

Check out my guide for using Intel GPUs in Kubernetes: https://github.com/UntouchedWagons/K3S-Intel
