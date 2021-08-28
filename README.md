# Fedora CoreOS ISO Customisation

The general  idea is creating a customised Fedora CoreOS image using the same
tools and environment used on building the distribution image itself.

The image is build using the CoreOS Assembler build environment and rpm-ostree
hybrid image/package system. Using both tools we can add custom ostree layers
containing scripts, software packages as Kubernetes and Timescaledb, and embed
ignition files to configure hostname and network address into the resulting ISO
image.

## Developement Environment

To build the image locally we need to setup our environment with support to
creating virtual machine and execute containers in the user space.

* kvm
* qemu
* libvirt
* podman
* butane
* coreos-installer

## CoreOS Assembler

We are mainly following the [Building Fedora CoreOS](https://github.com/coreos/coreos-assembler/blob/main/docs/building-fcos.md) instructions.

```
# Downloading the container
$ podman pull quay.io/coreos-assembler/coreos-assembler

# Create a build working directory
$ mkdir fcos
$ cd fcos

# Define a bash alias to run cosa
cosa() {
   env | grep COREOS_ASSEMBLER
   set -x
   podman run --rm -ti --security-opt label=disable --privileged                                    \
              --uidmap=1000:0:1 --uidmap=0:1:1000 --uidmap 1001:1001:64536                          \
              -v ${PWD}:/srv/ --device /dev/kvm --device /dev/fuse                                  \
              --tmpfs /tmp -v /var/tmp:/var/tmp --name cosa                                         \
              ${COREOS_ASSEMBLER_CONFIG_GIT:+-v $COREOS_ASSEMBLER_CONFIG_GIT:/srv/src/config/:ro}   \
              ${COREOS_ASSEMBLER_GIT:+-v $COREOS_ASSEMBLER_GIT/src/:/usr/lib/coreos-assembler/:ro}  \
              ${COREOS_ASSEMBLER_CONTAINER_RUNTIME_ARGS}                                            \
              ${COREOS_ASSEMBLER_CONTAINER:-quay.io/coreos-assembler/coreos-assembler:latest} "$@"
   rc=$?; set +x; return $rc
}

# Initializing
cosa init https://github.com/guimaluf/fedora-coreos-config

```

## Customisations
The below instructions are just possible ideas, that lead to error which
requires longer investigation and research in order to work it right.

### Kubernetes
Install K3s, a lightweight Kubernetes distribution.

manifest.yaml
```yaml
packages:
  - k3s-selinux

repos:
  - rancher

postprocess:
  - |
    #!/usr/bin/env bash
    set -xeuo pipefail
    curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=1.21.3+k3s1 sh -
```

Using the above I got an error where FCOS can not find the k3s-selinux package.
Would be useful to understand why it can not see the recent added repository.

### TimescaleDB

Install Helm and use it to install TimescaleDB helm chart.

manifest.yaml
```yaml
postprocess:
  - |
    #!/usr/bin/env bash
    set -xeuo pipefail
    curl -fsSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash -
    helm repo add timescaledb 'https://raw.githubusercontent.com/timescale/timescaledb-kubernetes/master/charts/repo/'
    helm repo update
    helm install timescaledb/timescaledb-single --generate-name
```

For both k8s and TimescaleDB the network is not configured during the building
process. This requires the necessary scripts are loaded during the fetch stage,
or even installed as a package rather than a script.

### Using rpm-ostree

Another possible option to add the software layers to the image is generating, extracting, and applying the ostree during building time. An example can be found [here](https://github.com/nmasse-itix/itix-coreos-config/blob/main/build.sh)

## Generating ISO
```bash
# Building
cosa fetch
cosa build
cosa buildextend-metal
cosa buildextend-metal4k
cosa buildextend-live

IMAGE="${PWD}/builds/34.*.dev.0/x86_64/fedora-coreos-34.*.dev.0-live.x86_64.iso"
```

The above will generate a ISO image with the defined customisations.

### NetworkConfig

## Embedding configuration into ISO

Create ignition file to register an ssh key and define hostname.

### netconfig.bu
```yaml
variant: fcos
version: 1.4.0
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - {{SSH_KEY}} # Replace with sshkey

storage:
  files:
    - path: /etc/hostname
      mode: 0644
      contents:
        inline: test.aggreko.local
 ```

Generate ignition file

 ```bash
butane --pretty --strict netconfig.bu > netconfig.ign
 ```

### coreos-installer

After generating the ISO with k8s and timescaledb package layers and ignition
file with custom configs. we need to add
the network config and ignition file as kernel arguments
We use `coreos-installer` to include both ignition file and kernel arguments

```bash
coreos-installer iso ignition embed -i netconfig.ign $IMAGE
coreos-installer iso kargs modify -a ip=192.168.1.1::192.168.1.1:255.255.255.0:::off
```

## Testing ISO

```bash
IGNITION_CONFIG="${PWD}/netconfig.ign"
IMAGE="${PWD}/fcos/builds/34.*.dev.0/x86_64/fedora-coreos-34.*.dev.0-live.x86_64.iso"
VM_NAME="fcos-test-01"
VCPUS="2"
RAM_MB="2048"
DISK_GB="10"

virt-install --connect="qemu:///system" --name="${VM_NAME}" --vcpus="${VCPUS}"
  --memory="${RAM_MB}" \
  --os-variant="fedora-coreos-stable" --import --graphics=none \
  --disk="size=${DISK_GB}" \
  --cdrom ${IMAGE}
```

# References
* https://github.com/coreos/fedora-coreos-tracker/issues/377
* https://www.itix.fr/blog/build-your-own-distribution-on-fedora-coreos/
* https://github.com/timescale/timescaledb-kubernetes
* https://www.murillodigital.com/tech_talk/k3s_in_coreos/
* https://www.matthiaspreu.com/posts/fedora-coreos-embed-ignition-config/
* https://dustymabe.com/2020/04/04/automating-a-custom-install-of-fedora-coreos/
* https://github.com/coreos/fedora-coreos-config
* https://devopstales.github.io/kubernetes/k3s-fcos/
