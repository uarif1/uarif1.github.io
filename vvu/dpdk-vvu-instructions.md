# Virtio Vhost User Demo

## Introduction

virtio-vhost-user (vvu) allows moving the vhost-user process inside a VM. This is done by moving
vhost device backend into a guest and tunneling the vhost-user protocol over a new type
of device type called virtio-vhost-user. Further information can be found at
https://wiki.qemu.org/Features/VirtioVhostUser

This page outlines the steps to recreate a setup showcasing virtio-vhost-user with DPDK
testpmd app. The patches for vvu are on top of DPDK v22.03 and QEMU v7.0.50.

### 1. Clone the repositories
```
mkdir vvu
cd vvu

git clone git@github.com:uarif1/qemu.git qemu -b vvu_7.0.50
git clone git@github.com:uarif1/dpdk.git dpdk -b vvu
```

### 2. Build the components
```
# Build DPDK
cd dpdk/
meson build; ninja -C build

# Build QEMU
cd ../qemu
mkdir build/; cd build
../configure --target-list=x86_64-softmmu; make -j40
```

### 3. Launch the "Backend VM" with the virtio-vhost-user device backend
```
./qemu/build/qemu-system-x86_64 \
  -machine q35,accel=kvm -cpu host -smp 4 -m 4G \
  -drive file=ubuntu_frontend.img,if=virtio \
  -netdev user,id=hostnet,hostfwd=:127.0.0.1:10082-:80,hostfwd=:127.0.0.1:10024-:22 \
  -device virtio-net-pci,netdev=hostnet,mac=00:8c:fa:e4:a3:53 \
  -netdev user,id=netdev1 \
  -device virtio-net-pci,netdev=netdev1,addr=06.0 \
  -chardev socket,id=chardev0,path=vhost-user.sock,server=on,wait=off \
  -device virtio-vhost-user-pci,chardev=chardev0,addr=07.0,disable-legacy=on \
  -nographic -monitor unix:dpdk-qemu-monitor-socket,server,nowait

```

### 4. copy the DPDK testpmd app to the Backend VM from host
```
# For example via scp:
scp -P 10024 ./dpdk/build/app/dpdk-testpmd root@localhost:/root/
scp -P 10024 ./dpdk/usertools/dpdk-devbind.py root@localhost:/root/

```

### 5. Bind the devices and launch the DPDK testpmd app in the Backend VM
```
export VVU_DEVICE="0000:00:07.0"
export VNET_DEVICE="0000:00:06.0"
modprobe vfio enable_unsafe_noiommu_mode=1
echo Y > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
modprobe vfio-pci
python3 ./dpdk-devbind.py -b vfio-pci --force '00:06.0'
python3 ./dpdk-devbind.py -b vfio-pci '00:07.0'
echo 256 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
./dpdk-testpmd -l 0-1 --allow "$VVU_DEVICE" \
    --vdev net_vhost0,iface="$VVU_DEVICE",virtio-transport=1 \
    --allow "$VNET_DEVICE" -- -i
```

### 6. Launch the "Frontend VM" which has the vhost-user frontend
```
./qemu/build/qemu-system-x86_64 \
  -M accel=kvm -cpu host -m 1G \
  -object memory-backend-file,id=mem0,mem-path=/dev/shm/ivshmem,size=1G,share=on \
  -numa node,memdev=mem0 \
  -drive file=ubuntu_backend.img,if=virtio \
  -chardev socket,id=chardev0,path=vhost-user.sock \
  -netdev vhost-user,chardev=chardev0,id=netdev0 \
  -device virtio-net-pci,netdev=netdev0 \
  -nographic -monitor unix:app-qemu-monitor-socket,server,nowait \
```

### 7. Start forwarding packets using the backend VM and validate
```
testpmd> set verbose 3
testpmd> show config fwd
testpmd> start

testpmd> show fwd stats all
```