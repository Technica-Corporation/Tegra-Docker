# Tegra-Docker
This repository contains instructions and key files to enable Docker support on NVIDIA Tegra devices, specifically the TX-2.  These instructions should work for the most part for other Tegra devices but we currently only have TX-2 to test with so any feedback on getting this to run on other devices is welcome.

## Motivation
We have recently been looking into TX-2s as a development platform and were very interested in using Docker to enable of our development and deployment scenarios.  I attended GTC 2017 this year and was very excited about some of the nvidia-docker integration and was eager to try these out on the TX-2.  However, once I got back and tried these out I quickly learned that nvidia-docker was not supported on the TX-2 or other Tegra devices and probably would not be in the short term.

Not one to be discouraged, we set out to learn what was needed to make this happen.  After some frustration and trial and error we were able to successfully get Docker running on the TX-2.  Not satisfied with just getting Docker to run, we also were able to get GPU programs running in a Docker container to run.

## Challenges
There are several challenges we had to overcome in order to get this working correctly:
1. The stock kernel that is deployed when flashing the TX-2 does not have all of the kernel options necessary for containers to work on TX-2 devices.
2. The NVIDIA drivers work much differently on the TX-2 than on a normal Linux system.  The nvidia-docker wrapper written by NVIDIA will not work on the TX-2.  The nvidia-docker project basically wraps Docker commands and sets up the environment correctly so that Docker containers will have access to the GPU.  Without this your GPU programs will not work on TX-2.

## Solution
To get this working you'll need to compile a custom L4T kernel with the correct kernel configuration to allow you to use containers.  Once that is flashed and running on your device you can install Docker and then will just need to pass in specific Docker parameters to allow your Docker containers to have access to the GPU.

### Kernel Configuration
Based on the information in this thread on the TX-2 development board, [Docker on the TX2](https://devtalk.nvidia.com/default/topic/1000224/jetson-tx2/docker-on-the-tx2/), we were able to get Docker running on the TX-2.

There are multiple ways to compile the kernel and if you have never done this before it can be intimidating, but it isn't too difficult.  If you want to compile the kernel on the TX-2 device then you can follow these instructions: [buildJetsonTX2Kernel](https://github.com/jetsonhacks/buildJetsonTX2Kernel).  Just make sure to use this custom [config](https://github.com/frankjoshua/buildJetsonTX2Kernel/blob/master/docker_config/config) file so that it will enable the Docker options in the kernel.  Just note that this is for the older kernel present in JetPack 3.0, so some of the instruction below would need to be adjusted accordingly.

We did not compile on the TX-2 but rather chose to cross compile our kernel from another Linux host.  NVIDIA recommends that you use Ubuntu 14.04 for this, but we were successfully able to run using Ubuntu 16.04.

#### Kernel Compilation
To compile a custom kernel for the TX-2 on a x86 Ubuntu 16.04 machine:

***NOTE:  Before attempting this procedure, make sure you backup any important files from your TX-2.  Updating the kernel can render your system unusable and you may need to reflash the system from JetPack to get it useable again***
1. Create a directory called `kernel_build` on your Linux machine to contain the build files. Will refer to this directory as `$BUILD_ROOT`.  Change into this directory to make it your working directory.

    **NOTE:  Might be helpful to set an environment variable called $BUILD_ROOT that points to your kernel_build directory.**
2. Download the Latest Driver Package from NVIDIA, [L4T Jetson TX2 Driver Package, 28.1](https://developer.nvidia.com/embedded/dlc/l4t-jetson-tx2-driver-package-28-1) and copy to `$BUILD_ROOT`
3. Uncompress into your `$BUILD_ROOT` directory:
```
tar -jxvf Tegra186_Linux_R28.1.0_aarch64.tbz2
```
4. Change into the `$BUILD_ROOT/Linux_for_Tegra` directory and run the `source_sync.sh` script.  This will download the latest kernel sources using GIT.  When prompted to enter a tag use `tegra-l4t-r28.1`.  You will need to enter the tag five or six different times for each of the projects needed to compile the kernel.
5. Download the GCC Toolchain, [l4t-gcc-toolchain-64-bit-28-1](https://developer.nvidia.com/embedded/dlc/l4t-gcc-toolchain-64-bit-28-1) and copy to `$BUILD_ROOT`
6. Uncompress the Toolchain into a directory called `toolchain`:
```
mkdir $BUILD_ROOT/toolchain
tar -xvf gcc-4.8.5-aarch64.solitairetheme8 -C toolchain
```
**NOTE:  It appears that the toolchain file currently downloading is called 'gcc-4.8.5-aarch64.solitairetheme8' which is probably a mistake.  If/When NVIDIA fixes this, make just uncompress the correct name that was downloaded.**

7. Set the following environment variables
```
export CROSS_COMPILE=$BUILD_ROOT/toolchain/install/bin/aarch64-unknown-linux-gnu-
export TEGRA_KERNEL_OUT=$BUILD_ROOT/kernel-out
export ARCH=arm64
```
8. Copy the custom kernel config file (.config) into the $TEGRA_KERNEL_OUT directory,  [.config](https://github.com/Technica-Corporation/Tegra-Docker/blob/master/kernel_config/config).
9. Change into the kernel source directory
```
cd $BUILD_ROOT/Linux_for_Tegra/sources/kernel/kernel-4.4
```
9. Compile the Kernel Image
```
make O=$TEGRA_KERNEL_OUT zImage
```
10. Create the Kernel Device Trees (DTB)
```
make O=$TEGRA_KERNEL_OUT dtbs
```
11. Make the Kernel Modules
```
make O=$TEGRA_KERNEL_OUT modules
make O=$TEGRA_KERNEL_OUT modules_install INSTALL_MOD_PATH=$TEGRA_KERNEL_OUT/modules
```
12. Archive the kernel modules
```
cd $TEGRA_KERNEL_OUT/modules
tar --owner root --group root -cjf kernel_supplements.tbz2 *
```
13.  Assuming all went well you have successfully compiled the custom kernel.  Next step is to copy the kernel files to your TX-2.

#### Copy Kernel Files to TX-2
1. You should have already flashed your TX-2 with the base image and kernel that is provided with JetPack.  Your TX-2 should be booted and connected to the network so you can SCP the kernel files.
2. SCP the `$TEGRA_KERNEL_OUT/arch/arm64/boot/Image` and `$TEGRA_KERNEL_OUT/arch/arm64/boot/zImage` files to the TX-2 into the `/boot` directory
3. Replace all of the files in the `/boot/dtb` directory on the TX-2 with the files from `$TEGRA_KERNEL_OUT/arch/arm64/boot/dts` directory from the host machine
4. Copy the `$TEGRA_KERNEL_OUT/modules/kernel_supplements.tbz2` file to the TX-2 into the root directory
5. Uncompress the `/kernel_supplements.tbz2` file on the TX-2:
```
cd /
sudo tar -jxvf kernel_supplements.tbz2
```
6. Reboot the TX-2 for the new kernel to take effect
7. After the reboot, you should be able to log into the system.  Check the kernel version to make sure it was updated successfully:
```
uname -a
```
Should show it is running `4.4.38-container+`.

#### Docker Installation
Now that the kernel is updated you can install Docker.

1. Add the following line to the bottom of your `/etc/apt/sources.list`
```
deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable
```
2. Update the package lists
```
sudo apt-get update
```
3. Install Docker
```
sudo apt-get install docker.io
```

This should install Docker 1.12 on the system.  It is possible to install and run the latest version of Docker, 1.17, but you will need to compile from source.  See install instructions from the Docker site for information on how to compile Docker from source.

### Docker Configuration
At this point you should be able to run CPU based Docker containers on your TX-2.  Just make sure you are using images based on arm64v8.  You can test your Docker installation by running the Hello World Container:
```
docker run arm64v8/hello-world
```

If you want to run something more interesting, you can run the Ubuntu image:
```
docker run -it arm64v8/ubuntu /bin/bash
```

However, if you try to run a GPU program within a Docker container it will result in an error.

#### Simple GPU Docker Image
Let's build a simple image with deviceQuery so that we can test Docker's ability to run GPU programs.  This requires that you installed the CUDA package on your TX-2 via JetPack.  While this isn't necessary to run all CUDA programs, it is a good idea to have this installed on the base system.  If you haven't already done so, use JetPack to install CUDA on your target TX-2.
1. Build the deviceQuery sample which is located in /usr/local/cuda/samples/1_Utilities/deviceQuery
```
cd /usr/local/cuda/samples/1_Utilities/deviceQuery
make
```
This will create the deviceQuery executable.  If you run this on the native machine, it will give you information about the GPU on the TX-2

```
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "GP10B"
  CUDA Driver Version / Runtime Version          8.5 / 8.0
  CUDA Capability Major/Minor version number:    6.2
  Total amount of global memory:                 7852 MBytes (8233689088 bytes)
  ( 2) Multiprocessors, (128) CUDA Cores/MP:     256 CUDA Cores
  GPU Max Clock rate:                            1301 MHz (1.30 GHz)
  Memory Clock rate:                             13 Mhz
  Memory Bus Width:                              64-bit
  L2 Cache Size:                                 524288 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 32768
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 1 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            Yes
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Disabled
  Device supports Unified Addressing (UVA):      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 0
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 8.5, CUDA Runtime Version = 8.0, NumDevs = 1, Device0 = GP10B
Result = PASS
```
2. Create a directory for the image creation
3. Copy the /usr/local/cuda/samples/1_Utilities/deviceQuery/deviceQuery executable to this directory
4. Copy this [Dockerfile](https://github.com/Technica-Corporation/Tegra-Docker/blob/master/docker/samples/deviceQuery/Dockerfile) to the directory
5. Create the image
```
docker build -t device_query .
```
6. Run the image
```
docker run device_query
```
You should see the following output:

```
/cudaSamples/deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

cudaGetDeviceCount returned 35
-> CUDA driver version is insufficient for CUDA runtime version
Result = FAIL

```

This is because the CUDA driver and the GPU device are not visible to the Docker container.   

#### Giving Docker Access to GPU
On a normal x86 or x64 machine, the [nvidia-docker](https://github.com/NVIDIA/nvidia-docker) project would give your container access to the GPU.  However, NVIDIA has made it clear that they [will not currently support Tegra devices](https://github.com/NVIDIA/nvidia-docker/issues/214). Unfortunately, it is not as easy as compiling nvidia-docker on the TX-2 and utilizing this to give GPU access to containers.  The NVIDIA driver is very different on Tegra devices than on a normal system with an external GPU.  There are also libraries, such as NVML, which nvidia-docker uses but are not supported on Tegra devices.  

Fortunately it is possible to give containers access to the GPU by passing in some specific libraries and giving access to the devices related to the GPU to the container.

#### Device Parameters
Docker containers needs to have access to the following device files on the host:
* `/dev/nvhost-ctrl`
* `/dev/nvhost-ctrl-gpu`
* `/dev/nvhost-prof-gpu`
* `/dev/nvmap`
* `/dev/nvhost-gpu`
* `/dev/nvhost-as-gpu`

These can be passed to the container using the `--device` switch

#### Driver Library Files
The Containers also need access to the drivers.  For Tegra these are located in `/usr/lib/aarch64-linux-gnu/tegra`.  You should add this to the container using the `-v` command line switch.

The `/usr/lib/aarch64-linux-gnu/tegra` directory contains libraries that will be loaded dynamically by the CUDA appliations.  This path should also be added to the LD_LIBRARY_PATH environment variable in your Dockerfile as well.

Will most likely also need access to other libraries depending on your GPU program.  Specifically you will need access to the CUDA runtime libraries.  Up to you if you want to install the CUDA libraries on the host and pass those through to the container as a volume or to build that into the image.

#### Running Device Query
With the above information, you can now run the device_query container we built above by passing in the correct parameters to the Docker run command:

```
docker run --device=/dev/nvhost-ctrl --device=/dev/nvhost-ctrl-gpu --device=/dev/nvhost-prof-gpu --device=/dev/nvmap --device=/dev/nvhost-gpu --device=/dev/nvhost-as-gpu -v /usr/lib/aarch64-linux-gnu/tegra:/usr/lib/aarch64-linux-gnu/tegra device_query
```

This should result in deviceQuery successfully being run inside of the Docker container.


#### Wrapper Script
We've created a very simple wrapper script called [tx2-docker](https://github.com/Technica-Corporation/Tegra-Docker/blob/master/bin/tx2-docker) that will wrap your Docker commands with the specific command line parameters needed to give GPU access to Docker containers.  This is a VERY simplified version of what the nvidia-docker project does.  

To launch a Docker container that needs GPU access just run: `tx2-docker run <image_name>`.  If you container needs any additional libraries, just need to add the directory or library to the `NV_LIBS` variable for it to be included as a volume.
