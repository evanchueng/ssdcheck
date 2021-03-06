# SSDcheck

SSDcheck, a novel methodology to accurately predict the timing of futuer high latency without hardware modifications.
SSDcheck extracts device-specific feature parameters and provides performance model of black-box SSDs by using the extracted features.
Users can utilize the extracted features and performance model in various scenarios.

The proejct mainly consists of three parts: feature extraction, performance model, and use cases.
First, SSDcheck extracts device-specific features (e.g., *internal volume*, *write buffer*) by using diagnosis code snippets.
With the extracted features, SSDcheck builds the performance model to predict the latency of the next access.
Lastly, we provide two example use cases exploiting SSDcheck to improvde the overall throughput and reduce the tail latency significantly.

## How to build

To use SSDcheck, users need to build two packages: *fio* and *framework*.
To extract device-specific features, we provide diagnosis code snippets by modifying flexible I/O tester (fio) to generate manipulated access patterns.
In addition to building fio, users should build a kernel image since our all components (e.g., prediction engine, use cases) are implemented in the kernel level.
Below instructions show how to build the modified fio and our kernel framework.

### fio

We use flexible I/O tester (fio) to generate a manipulated access pattern to extract useful feature parameters. Building the modified fio is really simple. Please execute below commands in the fio's source directory (**tools/fio**)

```
./configure
make
```

### Framework

We implemented SSDcheck and two example use cases in the real system to measure the prediction accuracy and the performance improvement.
In our evaluation, we used a Dell PowerEdge R730 server with two Intel Xeon E5-2640 v3 with the linux kernel version *3.19.0-18-lowlatency*.
Since the current implementation modifies a MegaRAID SAS device driver to monitor I/O requests and measure the latency of the requests during runtime, users should attach SSDs via a hardware RAID controller using MegaRAID SAS device driver.
Apart from the latency monitor, other components (e.g., performance model, use cases) are implemented as a kernel module; therefore, users can easily port these modules to their own kernel.

The kernel source code is in **framework/linux-lts-vivid-3.19.0** directory and users can build this kernel source similar to typical kernel image compilation.
First, to setup build environment, you should install some packages. You can get these packages by following commands. (Note that we assume kernel version *3.19.0-18-lowlatency*)

```
sudo apt-get build-dep linux-image-3.19.0-18-lowlatency
```

After preparing the build environment, you can build the kernel source by following commands.

```
(in kernel source directory)
make prepare
make scripts

# Here, if the build system uses different kernel version, you can get Module.symvers by downloading kernel source code (3.19.0-18-lowlatency).
cp -v /usr/src/linux-headers-3.19.0-18-lowlatency/Module.symvers .

# To build & install base_module which contains prediction engine
KBUILD_EXTRA_SYMBOLS=base_module/Module.symvers make M=$(pwd)/drivers/block modules
make -C /usr/src/linux-headers-`uname -r` M=$(pwd)/base_module modules_install

# To build & install the megaraid module
KBUILD_EXTRA_SYMBOLS=base_module/Module.symvers make M=$(pwd)/drivers/scsi/megaraid modules
make -C /usr/src/linux-headers-`uname -r` M=$(pwd)/drivers/scsi/megaraid modules_install
```

After finishing building the kernel image, you can replace the current kernel image with the new image.

```
update-initramfs -u -k 3.19.0-18-lowlatency
```

## How to use

## Feature Extraction (by using fio)

We modify flexible I/O tester (fio) to generate a manipulated access pattern to extract device-specific feature parameters.
We provide two new options: *fix* and *flip*.
*Fix* option only fixes a value (0 or 1) for a given bit index. For example, if a user wants to fix a bit value as 1 for a bit index 17, the user can set options as follows.
  
```
bitengine=fix
bitidx_1=17
bitidx_1_value=1
```  

*Flip* option alternately performs writes to two addresses of which the only specific bit value (of the given bit index) is different. For example, the user can only change a bit value for a bit index 17 by using these options as follows.
  
```
bitengine=flip
bitidx_1=17
```

## Performance Modeling

## Use Cases

### Volume-aware Logical Volume Manager (VA-LVM)
For VA-LVM, we provide an example device mapper (dm-split.c, located in ${KERNEL}/drivers/md/dm-split.c). You can build this module (dm-split) by following commands.

```
cd ${KERNEL}/
make M=$(pwd)/drivers/md modules
```

After building successfully, you can get a kernel module (dm-split.ko) which can be loaded via 'insmod dm-split.ko'. Now, you can create volume-aware logical volumes by using 'dmsetup' command.

```
dmsetup create <volume_name> --table "0 <size> split <path of block device> 0 <bitidx1>:<bitval1> <bitidx2>:<bitval2>"
 <volume_name>: name of the volume to be created (e.g., /dev/mapper/<volume_name>)
 <size>: size of the volume
 <path of block device>: a path of physical block to be splitted (e.g., /dev/sd*)
 <bitidx*>: a bit index related to internal volume selection
 <bitval*>: a bit value for the given bit index (0 or 1)
 * Note that bit index and value are extracted from the feature extraction.
```

For now, you can create multiple volume-aware logical volumes sharing the same SSD without any interference among the volumes.


### Prediction-aware I/O Scheduler (PAS)


# Publication

This work has been published in [IEEE CAL '18](https://ieeexplore.ieee.org/document/8126227) and [MICRO '18](https://ieeexplore.ieee.org/abstract/document/8574561).
