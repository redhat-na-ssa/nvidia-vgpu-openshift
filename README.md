# NVIDIA vGPU on OpenShift

This repository serves as a complement to the [NVIDIA documentation on vGPUs on OpenShift](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/openshift-virtualization.html) to make it easier to deploy the solution on a OpenShift cluster

There are two ways to configure mediated devices in OpenShift, one is with the OpenShift Virtualization operator and the other is with the NVIDIA GPU operator, this doc focuses on the latter. [Docs steps for configuring with OpenShift Virtualization](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/virtualization/managing-vms#virt-options-configuring-mdevs_virt-configuring-virtual-gpus)


## Testing environment

This repository has been tested on:

* OpenShift 4.18.13 cluster (stable-4.18 channel)
  * NVIDIA GPU Operator v25.3.0
  * NVIDIA Driver 570.133.10
  * NVIDIA T4 GPU (AWS instance [g4dn.metal](https://instances.vantage.sh/aws/ec2/g4dn.metal))

## Prerequisites

The NVIDIA vGPU software needs to be obtained from the [NVIDIA Licensing Portal](https://nvid.nvidia.com/dashboard/#/dashboard):

(Following block is replicated from the NVIDIA documentation)

> [!NOTE]
> Login to the NVIDIA Licensing Portal and navigate to the Software Downloads section.
>
> The NVIDIA vGPU Software is located on the Driver downloads tab of the Software Downloads page.
>
> Click the Download link for the Linux KVM complete vGPU package. Confirm that the Product Version column shows the vGPU version to > install. Unzip the bundle to obtain the NVIDIA vGPU Manager for Linux file, NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run.
>
> NVIDIA AI Enterprise customers must use the aie .run file for building the NVIDIA vGPU Manager image. Download the NVIDIA-Linux-x86_64-<version>-vgpu-kvm-aie.run file instead, and rename it to NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run before proceeding with the rest of the procedure.

## Instructions

### Build Driver image on local machine

Disconnected vgpu-manager [build documentation](DISCONNECTED-vgpu-manager.md)

Since we need the proprietary drivers from NVIDIA, the easiest way to get started is by building the container image locally

Local requirements:

* git
* podman

```bash
git clone https://gitlab.com/nvidia/container-images/driver
cd driver/vgpu-manager/rhel8/
cp path/to/NVIDIA-Linux-x86_64-<version>-vgpu-kvm.run .

# the DRIVER_VERSION buildarg is from driver filename: NVIDIA-Linux-x86_64-570.133.10-vgpu-kvm.run
podman build --build-arg DRIVER_VERSION=570.133.10 -t nvidia-driver:latest .
```

### (Optional) Push to internal registry

Once we have the container image we can use the OpenShift image registry to host but you can also use another registry if available

In the example we are using the driver version and the version of OpenShift in the tag in the form "driver_version-rhcos_version": `570.133.10-rhcos4.18`

```bash
oc login https://api.cluster.example.com:6443 --username admin
oc registry login

podman tag nvidia-driver:latest image-registry.openshift-image-registry.svc:5000/nvidia-gpu-operator/vgpu-manager:570.133.10-rhcos4.18
podman push --tls-verify=false image-registry.openshift-image-registry.svc:5000/nvidia-gpu-operator/vgpu-manager:570.133.10-rhcos4.18
```

## Deploy GPU Operator

### Label the nodes

Nodes that will be used for vGPU should be labeled to ensure that the GPU operator installs the correct NVIDIA drivers

> [!WARNING]
If the workload label was unset or set incorrectly you will have to remove the kernel modules that are loaded in order to change the supported workload for the node (rebooting the node is sufficient).

(from the NVIDIA documentation)
> If the node label nvidia.com/gpu.workload.config does not exist on the node, the GPU Operator assumes the default GPU workload configuration, container, and deploys the software components needed to support this workload type. To change the default GPU workload configuration, set the following value in ClusterPolicy: .sandboxWorkloads.defaultWorkload=<config>.

The options are:
* container
* vm-passthrough
* vm-vgpu

```sh
oc label node ip-10-0-2-117.us-east-2.compute.internal --overwrite nvidia.com/gpu.workload.config=vm-vgpu
```

```sh
node/ip-10-0-2-117.us-east-2.compute.internal labeled
```

Check the full list of nodes for the labels, in this case only the first node will be set up for vGPU, the rest will get the standard container-based workload support for GPUs

```sh
oc get nodes -o custom-columns=Name:.metadata.name,GPU:.metadata.labels.'nvidia\.com/gpu\.workload\.config'
```

```sh
Name                                        GPU
ip-10-0-2-117.us-east-2.compute.internal    vm-vgpu
ip-10-0-42-92.us-east-2.compute.internal    <none>
ip-10-0-5-156.us-east-2.compute.internal    <none>
```

### Install the vGPU manager image

#### With CLI

> [!NOTE]
> If you are using an external registry, change the registry URI in the YAML before applying

Apply the ClusterPolicy in this repository

```sh
oc apply -f clusterpolicy.yaml
```

#### With GUI

Once installing the GPU operator, add a new ClusterPolicy and switch to YAML

![clusterpolicy](images/clusterpolicy-create.png)
![clusterpolicy](images/clusterpolicy-yaml.png)

> [!NOTE]
> Modify the YAML and set the repository to the correct URL if you are not using the internal registry

Find the settings listed below and make sure they are set, leave everything else as the default

```sh
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  sandboxWorkloads:
    enabled: true
  vgpuManager:
    enabled: true
    image: vgpu-manager
    repository: image-registry.openshift-image-registry.svc:5000/nvidia-gpu-operator
    version: 570.133.10
...
```

### Verification

#### Check the node is advertising GPUs

```sh
oc get nodes ip-10-0-2-117.us-east-2.compute.internal -o json | jq .status.allocatable
```

```json
{
 "cpu": "95500m",
 "devices.kubevirt.io/kvm": "1k",
 "devices.kubevirt.io/tun": "1k",
 "devices.kubevirt.io/vhost-net": "1k",
 "ephemeral-storage": "191655242229",
 "hugepages-1Gi": "0",
 "hugepages-2Mi": "0",
 "memory": "394701388Ki",
 "nvidia.com/GRID_T4-8Q": "16",
 "nvidia.com/gpu": "0",
 "pods": "250"
}
```

## Configure OpenShift virtualization

### Modify Hyperconverged object

The `resourceName` should correspond to the name of the resource from the allocatable resources, in this example `nvidia.com/GRID_T4-8Q`

```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
spec:
  featureGates:
    disableMDevConfiguration: true
  permittedHostDevices:
    mediatedDevices:
    - externalResourceProvider: true
      mdevNameSelector: GRID T4-8Q
      resourceName: nvidia.com/GRID_T4-8Q
```

After which, you should be able to create or modify a VM and add a GPU device.

## Use a custom vGPU configuration

So far we have been using the default configuration. If you want to set all GPUs in the node to a specific configuration you can just use the `nvidia.com/vgpu.config` label. However to do a custom configuration per GPU in a cluster you need to apply a ConfigMap and then reference it in the ClusterPolicy.

Example in this repo:

```
oc apply -f configmap.yaml
```

```
oc patch clusterpolicy gpu-cluster-policy --type=merge -p '{"spec":{"vgpuDeviceManager":{"config":{"name":"custom-vgpu-config"}}}}'
```

Then you can apply the configuration to a node:

```
oc label node ip-10-0-31-236.us-east-2.compute.internal nvidia.com/vgpu.config=T4-custom
```

```
oc get nodes ip-10-0-31-236.us-east-2.compute.internal -o yaml | grep -A 12 allocatable| grep nvidia
```
```
    nvidia.com/GRID_T4-8C: "2"
    nvidia.com/GRID_T4-8Q: "12"
    nvidia.com/GRID_T4-16C: "1"
```

## Testing in the VM

The VM needs to use the GRID drivers which are used to communicate with the vGPU host driver. For this test I used `nvidia-linux-grid-570-570.133.20-1.x86_64.rpm`

EPEL is required for DKMS:

```
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
dnf install nvidia-linux-grid-570-570.133.20-1.x86_64.rpm
```

After which, we have the correct GPU available:

```
nvidia-smi
```
```
Wed May 28 09:44:12 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.133.20             Driver Version: 570.133.20     CUDA Version: 12.8     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  GRID T4-16C                    On  |   00000000:09:00.0 Off |                    0 |
| N/A   N/A    P8            N/A  /  N/A  |       1MiB /  16384MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

### Testing with PyTorch:

Adding cuda-toolkit per docs: https://developer.nvidia.com/cuda-downloads

```
dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo
dnf -y install cuda-toolkit-12-9
```

Creating a virtual environment and installing pytorch:

```
python3 -m venv env
env/bin/pip install torch
env/bin/pip install numpy
```

`test_torch.py`
```
import sys
import torch

def main():
    print("Python Version:", sys.version)
    print("Torch Version:", torch.__version__)
    print("CUDA Available:", torch.cuda.is_available())
    print("CUDA Device Count:", torch.cuda.device_count())

    if torch.cuda.is_available():
        print("CUDA Device Name:", torch.cuda.get_device_name(0))

if __name__ == '__main__':
    main()
```

```
env/bin/python test_torch.py
```
```
Python Version: 3.9.21 (main, Feb 10 2025, 00:00:00)
[GCC 11.5.0 20240719 (Red Hat 11.5.0-5)]
Torch Version: 2.7.0+cu126
CUDA Available: True
CUDA Device Count: 1
CUDA Device Name: GRID T4-16C
```

## Other Verification and Debugging steps

### gpu-operator must-gather

```
curl -o must-gather.sh -L https://raw.githubusercontent.com/NVIDIA/gpu-operator/main/hack/must-gather.sh
chmod +x must-gather.sh
./must-gather.sh
```

### Node labels

Initially the gpu operator should label the node with:

```
"nvidia.com/gpu.deploy.cc-manager": "true",
"nvidia.com/gpu.deploy.nvsm": "",
"nvidia.com/gpu.deploy.sandbox-device-plugin": "true",
"nvidia.com/gpu.deploy.sandbox-validator": "true",
"nvidia.com/gpu.deploy.vgpu-device-manager": "true",
"nvidia.com/gpu.deploy.vgpu-manager": "true",
"nvidia.com/gpu.present": "true",
```

After everything installs it adds

```
"nvidia.com/vgpu.config.state": "success",
```

#### Delete the node labels to reset

This is a bash function that removes all labels with the prefix `nvidia.com` from the node

```
node_cleanup() {
  local NODE="$1"
  oc get node "$NODE" -o go-template='{{range $key, $value := .metadata.labels}}{{if and (ge (len $key) 11) (eq (slice $key 0 11) "nvidia.com/")}}{{$key}}{{"\n"}}{{end}}{{end}}' | xargs -t -I {} oc label node "$NODE" {}-
}

node_cleanup <node-name>
```

### Pods

The gpu operator should control four Daemonsets for deploying vGPU manager onto the node:

```
$ oc get ds -n nvidia-gpu-operator
NAME                                                  NODE SELECTOR
nvidia-sandbox-device-plugin-daemonset                nvidia.com/gpu.deploy.sandbox-device-plugin=true
nvidia-sandbox-validator                              nvidia.com/gpu.deploy.sandbox-validator=true
nvidia-vgpu-device-manager                            nvidia.com/gpu.deploy.vgpu-device-manager=true
nvidia-vgpu-manager-daemonset-416.94.202505051351-0   feature.node.kubernetes.io/system-os_release.OSTREE_VERSION=416.94.202505051351-0,nvidia.com/gpu.deploy.vgpu-manager=true
```

### Driver version

Ensure that the NVIDIA GPU drivers are the correct version

```sh
oc debug node/ip-10-0-2-117.us-east-2.compute.internal
```

```sh
Starting pod/ip-10-0-2-117us-east-2computeinternal-debug-pk9mk ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.2.117
If you don't see a command prompt, try pressing enter.
```

```sh
cat /sys/module/nvidia/version
```

```output
570.133.10
```

#### Check PCI cards are present

```sh
oc debug node/ip-10-0-2-117.us-east-2.compute.internal
```

```output
Starting pod/ip-10-0-2-117us-east-2computeinternal-debug-pk9mk ...
To use host binaries, run `chroot /host`
Pod IP: 10.0.2.117
If you don't see a command prompt, try pressing enter.
sh-5.1#
```

```sh
lspci -nnk -d 10de:
```

```output
18:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
lspci: Unable to load libkmod resources: error -2
19:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
35:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
36:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
e7:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
e8:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
f4:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
f5:00.0 3D controller [0302]: NVIDIA Corporation TU104GL [Tesla T4] [10de:1eb8] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:12a2]
        Kernel driver in use: nvidia
```

#### Get the supported mediated devices types

```
for pci in $(lspci | grep -i nvidia | awk '{print $1}'); do  find /sys/bus/pci/devices/0000:$pci/mdev_supported_types/ -name name -exec cat {} \;; done 2> /dev/null | sort -u
```
