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

## CoreOS Assembler

We are following the [Building Fedora CoreOS](https://github.com/coreos/coreos-assembler/blob/main/docs/building-fcos.md) instructions.

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

# Building
cosa fetch
cosa build
cosa buildextend-metal
cosa buildextend-live --fast
```

### Customisations

 ```bash
 $ coreos-installer iso ignition embed -i path/to/ignition.ign path/to/fedora-coreos-*.x86_64.iso

 $ virt-install --name cdrom --network bridge=virbr0 --memory 4096 --disk size=20 --cdrom ./fedora-coreos-31.20200310.3.0-live.x86_64.iso
 ```

The install will proceed exactly the same as in the Live PXE case above and the user will eventually be logged in on the VGA console of the machine.

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

rpm-ostree install https://rpm.rancher.io/k3s/latest/common/centos/7/noarch/k3s-selinux-0.2-1.el7_8.noarch.rpm



https://www.itix.fr/blog/build-your-own-distribution-on-fedora-coreos/
