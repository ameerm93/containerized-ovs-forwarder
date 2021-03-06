# Containerized-ovs-forwarder
This repository builds a container image for ovs forwarder.
Also it implement two ovs modules:
- The containerovsdb which connect to ovs container and create bridges and vdpa ports.
- The ovsdb which connect to ovs on the host  

The ovs modules were taken from [openstack/os-vif](https://github.com/openstack/os-vif/).

## Build ovs-forwarder container image

Go to build directory
```
$ cd build/
```
Specify the MLNX_OFED_VERSION and run the build command
```
$ MLNX_OFED_VERSION=50218 /bin/bash build.sh
```
Now the ovs-docker image created successfully

## Build OVS container
Once the image is ready, you can create the container by running container_create script:  
Pass the required arguements to container_create,

```--port <port_number> OVS manager port default to 6000```

```--pci-args <pci_address> <vfs_range> A pci address of dpdk interface and range of vfs```

In case of bonding or you have more than one NIC for dpdk, reuse the --pci-args

```--pmd-cpu-mask <mask> A core bitmask that sets which cores are used by OVS-DPDK for datapath packet processing```  

```
$ mkdir -p /forwarder/var/run/openvswitch/
$ MLNX_OFED_VERSION=50218 /bin/bash container_create.sh --pci-args 0000:02:00.0 0-3 --port 6000
```
Now the ovs-forwarder created successfully

## Enable UCTX
Make sure that you have UCTX_EN enabled in FW configuration  
  ```
  $ mlxconfig -d mlx5_0 q UCTX_EN
  ```
if it's disabled run this command and reboot the server  
  ```
  $ mlxconfig -d mlx5_0 s UCTX_EN=1
  ```

## Enable switchdev mode
  Before starting ovs container, make sure to have vfs in switchdev mode and the vfs are binded
- None vf-lag case
  - Create vfs on mlnx port  
    ```$ echo 4 > /sys/class/net/p4p1/device/sriov_numvfs```  
  - Unbind vfs for mlnx port  
    ```$ for i in `lspci -D | grep nox | grep Virt| awk '{print $1}'`; do echo  $i > /sys/bus/pci/drivers/mlx5_core/unbind; done```  
  - Move mlnx port to switchdev mode  
    ```$ /usr/sbin/devlink dev eswitch set pci/0000:03:00.0 mode switchdev```  
  - Bind vfs for mlnx port  
    ```$ for i in `lspci -D | grep nox | grep Virt| awk '{print $1}'`; do echo  $i > /sys/bus/pci/drivers/mlx5_core/bind; done```  

- Vf-lag case
  - Create vfs on both mlnx ports   
    ``` echo 4 > /sys/class/net/p4p1/device/sriov_numvfs```  
    ``` echo 4 > /sys/class/net/p4p2/device/sriov_numvfs```  
  - Unbind vfs for both mlnx ports  
    ```$ for i in `lspci -D | grep nox | grep Virt| awk '{print $1}'`; do echo  $i > /sys/bus/pci/drivers/mlx5_core/unbind; done```  
  - Move mlnx ports to switchdev mode  
    ```/usr/sbin/devlink dev eswitch set pci/0000:03:00.0 mode switchdev```  
    ```/usr/sbin/devlink dev eswitch set pci/0000:03:00.1 mode switchdev```  
  - Plug mlnx ports to bond interface  
  - Bind vfs for both mlnx ports  
    ```for i in `lspci -D | grep nox | grep Virt| awk '{print $1}'`; do echo  $i > /sys/bus/pci/drivers/mlx5_core/bind; done```

## Start OVS container
Then start the contaier by running:
```
$ docker start ovs-forwarder:$MLNX_OFED_VERSION
```

## Use ovs_modules
You can use the python/example.py script in order to use ovs modules  
The script creates netdev bridge and vdpa port on the container,
and also create a bridge and plug representor port on the host  
Before using the ovs modules make sure that the requirements in requirements.txt are installed  
```
$ cd python
$ python example.py
```

## Supported Hardware
Containerized OVS Forwarder has been validated to work with the following Mellanox hardware:
- ConnectX-5 family adapters
- ConnectX-6Dx family adapters
