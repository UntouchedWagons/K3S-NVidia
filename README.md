# Making NVidia GPUs available in your kubernetes cluster

Getting NVidia GPUs to work in your kubernetes cluster has been in my experience a hot mess with conflicting documentation and helm charts. Through the power of chutzpah and dumb luck I managed to get a tutorial made to provide a working setup.

## The background

For this tutorial I assume you have the following:

 * A Working K3S cluster (If you don't have this I recommend [this video](https://www.youtube.com/watch?v=CbkEWcUZ7zM) by TechnoTim to get you started)
 * At least one node with some sort of NVidia GPU
 * Helm installed

For my test cluster I'm using Debian 13, Debian is not [officially supported](https://github.com/NVIDIA/nvidia-container-toolkit/issues/147#issuecomment-1808273017) but seems to work just fine.

Talos Linux has support for nvidia gpus but I have not been able to get the GPU Operator to work properly.

## Installing the GPU driver

The first step is to make sure the OS itself can interface with your GPU. Since I use Debian's cloud images the step to enable to correct repositories is slightly different for some reason:

    sudo sed -i 's/^Components: main$/Components: main contrib non-free non-free-firmware/' /etc/apt/sources.list.d/debian.sources

If you installed Debian normally you would want to edit `/etc/apt/sources.list`, the first line is likely the correct one to edit:

    sudo nano /etc/apt/sources.list
    deb http://mirror.csclub.uwaterloo.ca/debian/ trixie main

Stick `contrib non-free non-free-firmware` onto the end so that it looks something like this:

    deb http://mirror.csclub.uwaterloo.ca/debian/ trixie main non-free-firmware contrib

Press Ctrl+X to save, then Y to confirm and then enter.

Now we'll install gpg and add the nvidia container toolkit repository:

    sudo apt-get install -y gpg curl
    curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
    curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
    sudo apt-get update

Now we'll install the driver, nvidia-smi and the container runtime:

    sudo apt-get install nvidia-driver firmware-misc-nonfree nvidia-smi nvidia-container-runtime

You will get a warning about the nouveau driver, the open source driver for nvidia gpus. Just press okay.

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

## Installing the GPU Operator

No more needs to be done directly on your Kubernetes nodes. Next we'll install the NVidia GPU Operator using helm. Helm is like apt-get but for Kubernetes.

    helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
    helm repo update

Next we're going to create a namespace specifically for the GPU Operator:

    kubectl create namespace gpu-operator

By default if you have only one GPU resource only one pod can use that resource. This custom config file that tells the GPU operator to treat each physical GPU as four logical GPUs.

    kubectl apply -f time-slicing-config.yaml

And now for the piece de la resistance we will install the GPU Operator:

    helm install gpu-operator -n gpu-operator nvidia/gpu-operator --values values.yaml

I highly recommend using [K9S](https://k9scli.io/) to see if everything's going well. The whole setup process will take a minute or two. If everything has gone well all the pods should be Running and all the jobs should be Completed. You might see some pods with high restart counts, that's fine.

## Making sure it all works

The simplest way to see if it works is to use `nvidia-smi` but this time in a container:

    kubectl apply -f nvidia-smi.yaml

Let's see if it worked:

    kubectl logs nvidia-smi
    Fri Jan 30 01:43:02 2026       
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

It worked! You can remove the nvidia-smi job:

    kubectl delete -f nvidia-smi.yaml

## Letting other containers use NVidia GPUs

Getting other containers to use NVidia GPUs is fairly easy. Look at the contents of nvidia-smi.yaml, specifically lines 6 through 8. Lines 6 and 7 tell Kubernetes what node the pod is allowed to run on and line 8 tells Kubernetes to use the nvidia runtime instead of the default containerd. The nvidia runtime will automatically copy everything needed for your pod to use the GPU, you can check this like so:

    kubectl exec -it jellyfin-65767cc56c-th4p5 -- bash
    nvidia-smi

You should see similar output to what you got earlier. Now you can go into the Jellyfin playback settings and enable NVENC hardware transcoding.

## Sources:

  * https://github.com/NVIDIA/gpu-operator/issues/522#issuecomment-1545574686
  * https://github.com/NVIDIA/nvidia-docker/issues/1616#issuecomment-1571404575
  * https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html
