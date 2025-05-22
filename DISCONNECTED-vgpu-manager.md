# Disconnected vgpu-manager

Date: May 21, 2025

In order to support vgpu-manager with disconnected, we need to package the required RPMs into the vgpu-manager image. This requires changes to the upstream NVIDIA/container-images/driver repository

## Get the updated code

```
git clone https://github.com/redhat-na-ssa/nvidia-container-images-driver
cd nvidia-container-images-driver/vgpu-manager/rhel8
git checkout -b vgpu-fixes origin/vgpu-fixes
```

## Get the driver-toolkit image for your cluster version

```sh
# login to OpenShift or use the OpenShift web-terminal

# oc get clusterversions
# export OPENSHIFT_VERSION=4.16.40

export OPENSHIFT_VERSION=$(oc get clusterversions.config.openshift.io -o custom-columns=:status.desired.version --no-headers)
export DRIVER_TOOLKIT=$(oc adm release info --image-for=driver-toolkit quay.io/openshift-release-dev/ocp-release:${OPENSHIFT_VERSION}-x86_64)
```

## Create the driver-toolkit container and download the rpms

```sh
# login with pull secret to get OpenShift images
export REGISTRY_AUTH_FILE=~/Downloads/pull-secret.txt
```

```sh
podman run -v $(pwd):/work:z --workdir /work -it --rm $DRIVER_TOOLKIT /bin/bash
```

In the container, download the required RPMs

```sh
dnf install -y yum-utils
yumdownloader --archlist x86_64,noarch pciutils hwdata pciutils-libs
exit
```

## Build the vgpu-manager image based on the driver version

```sh
podman build --build-arg DRIVER_VERSION=570.133.10 -t vgpu-manager:latest .
```
