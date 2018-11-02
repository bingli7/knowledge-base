 TensorFlow是一个开放源代码软件库，用于进行高性能数值计算。借助其灵活的架构，用户可以轻松地将计算工作部署到多种平台（CPU、GPU、TPU）和设备（桌面设备、服务器集群、移动设备、边缘设备等）。TensorFlow最初是由 Google Brain 团队中的研究人员和工程师开发的，可为机器学习和深度学习提供强力支持，并且其灵活的数值计算核心广泛应用于许多其他科学领域。

京东云GPU云主机提供实时高速，提供卓越的并行计算及浮点计算能力，快速构建异构计算应用。基于GPU的高效计算服务，适用于人工智能、图像处理等多领域场景。

基于京东云GPU云主机搭建TensorFlow深度学习环境的大致步骤如下：

- 安装Nvidia Driver、CUDA Toolkit、cuDNN加速库
- 安装TensorFlow
- 配置Tensorboard可视化工具及Jupyter notebook
- 运行深度学习Demo示例

# 创建GPU云主机

操作系统：Ubuntu 16.04 64位  
GPU：	1 x Nvidia Tesla P40

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/JD-GPU-vm.png?raw=true)

# 安装前准备
安装pip、virtualenv：

```
# curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
# python3 get-pip.py
# pip3 --version
pip 18.1 from /usr/local/lib/python3.5/dist-packages/pip (python 3.5)

# sudo pip install virtualenv
# virtualenv --version
16.0.0

# python3 --version
Python 3.5.2
```

安装python3-dev、python3-pip：

```
# sudo apt install python3-dev python3-pip
```
安装gcc：

```
# sudo apt-get install gcc
# gcc --version
gcc (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
如果没有，需安装linux-headers：
```
# sudo apt-get install linux-headers-$(uname -r)
```
禁用Ubuntu自带Nouveau驱动：

```
# lsmod | grep nouveau
nouveau  1495040  0
mxm_wmi16384  1 nouveau
wmi20480  2 mxm_wmi,nouveau
video  40960  1 nouveau
i2c_algo_bit   16384  1 nouveau
ttm94208  1 nouveau
drm_kms_helper155648  1 nouveau
drm   364544  3 ttm,drm_kms_helper,nouveau
# vi /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
# sudo update-initramfs -u
update-initramfs: Generating /boot/initrd.img-4.4.0-62-generic
W: mdadm: /etc/mdadm/mdadm.conf defines no arrays.
```

注：Nouveau驱动与NVIDIA驱动冲突，只有在禁用掉Nouveau后才能顺利安装NVIDIA驱动

Reboot云主机：

```
# reboot
```

重启后确认Nouveau drivers没有被加载：

```
# lsmod | grep nouveau
#
```

# 安装 Nvidia Driver
登录官网下载页面：https://www.nvidia.com/drivers  

选择相应的Driver版本：

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/Nvidia-driver.png?raw=true)

查询到对应的版本为 “396.44”：

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/Nvidia-driver-download.png?raw=true)

下载驱动到本地：

```
# wget http://us.download.nvidia.com/tesla/396.44/nvidia-diag-driver-local-repo-ubuntu1604-396.44_1.0-1_amd64.deb
```

下载完成后安装package repository：

```
# sudo dpkg -i nvidia-diag-driver-local-repo-ubuntu1604-396.44_1.0-1_amd64.deb 
Selecting previously unselected package nvidia-diag-driver-local-repo-ubuntu1604-396.44.
(Reading database ... 112453 files and directories currently installed.)
Preparing to unpack nvidia-diag-driver-local-repo-ubuntu1604-396.44_1.0-1_amd64.deb ...
Unpacking nvidia-diag-driver-local-repo-ubuntu1604-396.44 (1.0-1) ...
Setting up nvidia-diag-driver-local-repo-ubuntu1604-396.44 (1.0-1) ...

The public CUDA GPG key does not appear to be installed.
To install the key, run this command:
sudo apt-key add /var/nvidia-diag-driver-local-repo-396.44/7fa2af80.pub

# sudo apt-key add /var/nvidia-diag-driver-local-repo-396.44/7fa2af80.pub
OK
# sudo apt-get update
```


安装驱动：

```
# sudo apt-get install cuda-drivers
```

验证是否安装成功：

```
# sudo dpkg -l | grep nvidia
ii  nvidia-396                                      396.44-0ubuntu1                            amd64        NVIDIA binary driver - version 396.44
ii  nvidia-396-dev                                  396.44-0ubuntu1                            amd64        NVIDIA binary Xorg driver development files
ii  nvidia-diag-driver-local-repo-ubuntu1604-396.44 1.0-1                                      amd64        nvidia-diag-driver-local repository configuration files
ii  nvidia-modprobe                                 396.44-0ubuntu1                            amd64        Load the NVIDIA kernel driver and create device files
ii  nvidia-opencl-icd-396                           396.44-0ubuntu1                            amd64        NVIDIA OpenCL ICD
ii  nvidia-prime                                    0.8.2                                      amd64        Tools to enable NVIDIA's Prime
ii  nvidia-settings                                 396.44-0ubuntu1                            amd64        Tool for configuring the NVIDIA graphics driver
# cat /proc/driver/nvidia/version
NVRM version: NVIDIA UNIX x86_64 Kernel Module  396.44  Wed Jul 11 16:51:49 PDT 2018
GCC version:  gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10)
```

查看GPU信息：

```
# nvidia-smi
Tue Oct 30 15:15:11 2018       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 396.44                 Driver Version: 396.44                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla P40           Off  | 00000000:00:07.0 Off |                    0 |
| N/A   25C    P0    45W / 250W |      0MiB / 22919MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

# 安装CUDA Toolkit
CUDA 是 NVIDIA 创造的一个并行计算平台和编程模型。它利用图形处理器 (GPU) 能力，实现计算性能的显著提高。

TensorFlow暂不支持CUDA 10.0，本文选择安装CUDA 9.0。

CUDA Toolkit安装有两种方式：
1. Package安装 (RPM and Deb packages)
2. Runfile安装

本文选择Package方式安装。

官网下载deb包：
https://developer.nvidia.com/cuda-90-download-archive

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/CUDA-download.png?raw=true)

下载deb包到本地：

```
# wget https://developer.nvidia.com/compute/cuda/9.0/Prod/local_installers/cuda-repo-ubuntu1604-9-0-local_9.0.176-1_amd64-deb
# ls
cuda-repo-ubuntu1604-9-0-local_9.0.176-1_amd64-deb
```

安装repository并update：

```
# sudo dpkg -i cuda-repo-ubuntu1604-9-0-local_9.0.176-1_amd64-deb
Selecting previously unselected package cuda-repo-ubuntu1604-9-0-local.
(Reading database ... 117383 files and directories currently installed.)
Preparing to unpack cuda-repo-ubuntu1604-9-0-local_9.0.176-1_amd64-deb ...
Unpacking cuda-repo-ubuntu1604-9-0-local (9.0.176-1) ...
Setting up cuda-repo-ubuntu1604-9-0-local (9.0.176-1) ...

The public CUDA GPG key does not appear to be installed.
To install the key, run this command:
sudo apt-key add /var/cuda-repo-9-0-local/7fa2af80.pub

# sudo apt-key add /var/cuda-repo-9-0-local/7fa2af80.pub
OK
# sudo apt-get update
```


开始安装cuda toolkit：

```
# sudo apt-get install cuda
```

安装完成需等待几分钟。

查看CUDA：

```
# sudo dpkg -l | grep -i cuda
ii  cuda                                            9.0.176-1                                  amd64        CUDA meta-package
ii  cuda-9-0                                        9.0.176-1                                  amd64        CUDA 9.0 meta-package
ii  cuda-command-line-tools-9-0                     9.0.176-1                                  amd64        CUDA command-line tools
ii  cuda-core-9-0                                   9.0.176-1                                  amd64        CUDA core tools
ii  cuda-cublas-9-0                                 9.0.176-1                                  amd64        CUBLAS native runtime libraries
ii  cuda-cublas-dev-9-0                             9.0.176-1                                  amd64        CUBLAS native dev links, headers
ii  cuda-cudart-9-0                                 9.0.176-1                                  amd64        CUDA Runtime native Libraries
ii  cuda-cudart-dev-9-0                             9.0.176-1                                  amd64        CUDA Runtime native dev links, headers
ii  cuda-cufft-9-0                                  9.0.176-1                                  amd64        CUFFT native runtime libraries
ii  cuda-cufft-dev-9-0                              9.0.176-1                                  amd64        CUFFT native dev links, headers
ii  cuda-curand-9-0                                 9.0.176-1                                  amd64        CURAND native runtime libraries
ii  cuda-curand-dev-9-0                             9.0.176-1                                  amd64        CURAND native dev links, headers
ii  cuda-cusolver-9-0                               9.0.176-1                                  amd64        CUDA solver native runtime libraries
ii  cuda-cusolver-dev-9-0                           9.0.176-1                                  amd64        CUDA solver native dev links, headers
ii  cuda-cusparse-9-0                               9.0.176-1                                  amd64        CUSPARSE native runtime libraries
ii  cuda-cusparse-dev-9-0                           9.0.176-1                                  amd64        CUSPARSE native dev links, headers
ii  cuda-demo-suite-9-0                             9.0.176-1                                  amd64        Demo suite for CUDA
ii  cuda-documentation-9-0                          9.0.176-1                                  amd64        CUDA documentation
ii  cuda-driver-dev-9-0                             9.0.176-1                                  amd64        CUDA Driver native dev stub library
ii  cuda-drivers                                    396.44-1                                   amd64        CUDA Driver meta-package
ii  cuda-libraries-9-0                              9.0.176-1                                  amd64        CUDA Libraries 9.0 meta-package
ii  cuda-libraries-dev-9-0                          9.0.176-1                                  amd64        CUDA Libraries 9.0 development meta-package
ii  cuda-license-9-0                                9.0.176-1                                  amd64        CUDA licenses
ii  cuda-misc-headers-9-0                           9.0.176-1                                  amd64        CUDA miscellaneous headers
ii  cuda-npp-9-0                                    9.0.176-1                                  amd64        NPP native runtime libraries
ii  cuda-npp-dev-9-0                                9.0.176-1                                  amd64        NPP native dev links, headers
ii  cuda-nvgraph-9-0                                9.0.176-1                                  amd64        NVGRAPH native runtime libraries
ii  cuda-nvgraph-dev-9-0                            9.0.176-1                                  amd64        NVGRAPH native dev links, headers
ii  cuda-nvml-dev-9-0                               9.0.176-1                                  amd64        NVML native dev links, headers
ii  cuda-nvrtc-9-0                                  9.0.176-1                                  amd64        NVRTC native runtime libraries
ii  cuda-nvrtc-dev-9-0                              9.0.176-1                                  amd64        NVRTC native dev links, headers
ii  cuda-repo-ubuntu1604-9-0-local                  9.0.176-1                                  amd64        cuda repository configuration files
ii  cuda-runtime-9-0                                9.0.176-1                                  amd64        CUDA Runtime 9.0 meta-package
ii  cuda-samples-9-0                                9.0.176-1                                  amd64        CUDA example applications
ii  cuda-toolkit-9-0                                9.0.176-1                                  amd64        CUDA Toolkit 9.0 meta-package
ii  cuda-visual-tools-9-0                           9.0.176-1                                  amd64        CUDA visual tools
ii  libcuda1-396                                    396.44-0ubuntu1                            amd64        NVIDIA CUDA runtime library
```

给CUDA打补丁（共四个）：

```
# wget https://developer.nvidia.com/compute/cuda/9.0/Prod/patches/1/cuda-repo-ubuntu1604-9-0-local-cublas-performance-update_1.0-1_amd64-deb
# wget https://developer.nvidia.com/compute/cuda/9.0/Prod/patches/2/cuda-repo-ubuntu1604-9-0-local-cublas-performance-update-2_1.0-1_amd64-deb
# wget https://developer.nvidia.com/compute/cuda/9.0/Prod/patches/3/cuda-repo-ubuntu1604-9-0-local-cublas-performance-update-3_1.0-1_amd64-deb
# wget https://developer.nvidia.com/compute/cuda/9.0/Prod/patches/4/cuda-repo-ubuntu1604-9-0-176-local-patch-4_1.0-1_amd64-deb

# sudo dpkg -i cuda-repo-ubuntu1604-9-0-local-cublas-performance-update_1.0-1_amd64-deb
# sudo dpkg -i cuda-repo-ubuntu1604-9-0-local-cublas-performance-update-2_1.0-1_amd64-deb
# sudo dpkg -i cuda-repo-ubuntu1604-9-0-local-cublas-performance-update-3_1.0-1_amd64-deb
# sudo dpkg -i cuda-repo-ubuntu1604-9-0-176-local-patch-4_1.0-1_amd64-deb
```


Update：

```
# sudo apt-get update
```

升级：

```
# sudo apt-get upgrade cuda
```

环境变量配置：

```
# vi /etc/profile
......
export PATH=/usr/local/cuda-9.0/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda-9.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
# source /etc/profile
```



```
# nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2017 NVIDIA Corporation
Built on Fri_Sep__1_21:08:03_CDT_2017
Cuda compilation tools, release 9.0, V9.0.176
```


Samples示例安装（安装到当前目录）：

```
# cuda-install-samples-9.0.sh .
Copying samples to ./NVIDIA_CUDA-9.0_Samples now...
Finished copying samples.
# ls NVIDIA_CUDA-9.0_Samples/
0_Simple     2_Graphics  4_Finance      6_Advanced       common    Makefile
1_Utilities  3_Imaging   5_Simulations  7_CUDALibraries  EULA.txt
```

使用deviceQuery示例验证：

```
# cd NVIDIA_CUDA-9.0_Samples/1_Utilities/deviceQuery
# ls
deviceQuery.cpp  Makefile  NsightEclipse.xml  readme.txt
# make
/usr/local/cuda-9.0/bin/nvcc -ccbin g++ -I../../common/inc  -m64    -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_70,code=compute_70 -o deviceQuery.o -c deviceQuery.cpp
/usr/local/cuda-9.0/bin/nvcc -ccbin g++   -m64      -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_70,code=compute_70 -o deviceQuery deviceQuery.o 
mkdir -p ../../bin/x86_64/linux/release
cp deviceQuery ../../bin/x86_64/linux/release
# cd ../../bin/x86_64/linux/release/
# ls
deviceQuery
```


运行：

```
# ./deviceQuery 
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "Tesla P40"
  CUDA Driver Version / Runtime Version          9.2 / 9.0
  CUDA Capability Major/Minor version number:    6.1
  Total amount of global memory:                 22919 MBytes (24032378880 bytes)
  (30) Multiprocessors, (128) CUDA Cores/MP:     3840 CUDA Cores
  GPU Max Clock rate:                            1531 MHz (1.53 GHz)
  Memory Clock rate:                             3615 Mhz
  Memory Bus Width:                              384-bit
  L2 Cache Size:                                 3145728 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 2 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Enabled
  Device supports Unified Addressing (UVA):      Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 7
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 9.2, CUDA Runtime Version = 9.0, NumDevs = 1
Result = PASS
root@libing-GPU:~/NVIDIA_CUDA-9.0_Samples/bin/x86_64/linux/release# ./deviceQuery 
./deviceQuery Starting...

 CUDA Device Query (Runtime API) version (CUDART static linking)

Detected 1 CUDA Capable device(s)

Device 0: "Tesla P40"
  CUDA Driver Version / Runtime Version          9.2 / 9.0
  CUDA Capability Major/Minor version number:    6.1
  Total amount of global memory:                 22919 MBytes (24032378880 bytes)
  (30) Multiprocessors, (128) CUDA Cores/MP:     3840 CUDA Cores
  GPU Max Clock rate:                            1531 MHz (1.53 GHz)
  Memory Clock rate:                             3615 Mhz
  Memory Bus Width:                              384-bit
  L2 Cache Size:                                 3145728 bytes
  Maximum Texture Dimension Size (x,y,z)         1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
  Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
  Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
  Total amount of constant memory:               65536 bytes
  Total amount of shared memory per block:       49152 bytes
  Total number of registers available per block: 65536
  Warp size:                                     32
  Maximum number of threads per multiprocessor:  2048
  Maximum number of threads per block:           1024
  Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
  Max dimension size of a grid size    (x,y,z): (2147483647, 65535, 65535)
  Maximum memory pitch:                          2147483647 bytes
  Texture alignment:                             512 bytes
  Concurrent copy and kernel execution:          Yes with 2 copy engine(s)
  Run time limit on kernels:                     No
  Integrated GPU sharing Host Memory:            No
  Support host page-locked memory mapping:       Yes
  Alignment requirement for Surfaces:            Yes
  Device has ECC support:                        Enabled
  Device supports Unified Addressing (UVA):      Yes
  Supports Cooperative Kernel Launch:            Yes
  Supports MultiDevice Co-op Kernel Launch:      Yes
  Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 7
  Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >

deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 9.2, CUDA Runtime Version = 9.0, NumDevs = 1
Result = PASS
```

运行正常。

# 安装 cuDNN

cuDNN的全称为NVIDIA CUDA® Deep Neural Network library，是NVIDIA专门针对深度神经网络（Deep Neural Networks）中的基础操作而设计基于GPU的加速库。cuDNN为深度神经网络中的标准流程提供了高度优化的实现方式。

官网下载：https://developer.nvidia.com/cudnn

注：下载需先注册 NVIDIA Developer Program

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/cuDNN-download.png?raw=true)

下载的deb package共三个：libcudnn7、libcudnn7-dev、libcudnn7-doc：

```
# ls libcudnn*
libcudnn7_7.3.1.20-1+cuda9.0_amd64.deb      libcudnn7-doc_7.3.1.20-1+cuda9.0_amd64.deb
libcudnn7-dev_7.3.1.20-1+cuda9.0_amd64.deb
```


安装：

```
# sudo dpkg -i libcudnn7_7.3.1.20-1+cuda9.0_amd64.deb
Selecting previously unselected package libcudnn7.
(Reading database ... 168316 files and directories currently installed.)
Preparing to unpack libcudnn7_7.3.1.20-1+cuda9.0_amd64.deb ...
Unpacking libcudnn7 (7.3.1.20-1+cuda9.0) ...
Setting up libcudnn7 (7.3.1.20-1+cuda9.0) ...
Processing triggers for libc-bin (2.23-0ubuntu10) ...
# sudo dpkg -i libcudnn7-dev_7.3.1.20-1+cuda9.0_amd64.deb
Selecting previously unselected package libcudnn7-dev.
(Reading database ... 168322 files and directories currently installed.)
Preparing to unpack libcudnn7-dev_7.3.1.20-1+cuda9.0_amd64.deb ...
Unpacking libcudnn7-dev (7.3.1.20-1+cuda9.0) ...
Setting up libcudnn7-dev (7.3.1.20-1+cuda9.0) ...
update-alternatives: using /usr/include/x86_64-linux-gnu/cudnn_v7.h to provide /usr/include/cudnn.h (libcudnn) in auto mode
# sudo dpkg -i libcudnn7-doc_7.3.1.20-1+cuda9.0_amd64.deb
Selecting previously unselected package libcudnn7-doc.
(Reading database ... 168328 files and directories currently installed.)
Preparing to unpack libcudnn7-doc_7.3.1.20-1+cuda9.0_amd64.deb ...
Unpacking libcudnn7-doc (7.3.1.20-1+cuda9.0) ...
Setting up libcudnn7-doc (7.3.1.20-1+cuda9.0) ...
```


验证cuDNN：

```
# cp -r /usr/src/cudnn_samples_v7/ $HOME
# cd  $HOME/cudnn_samples_v7/mnistCUDNN
# make clean && make
rm -rf *o
rm -rf mnistCUDNN
/usr/local/cuda/bin/nvcc -ccbin g++ -I/usr/local/cuda/include -IFreeImage/include  -m64    -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_53,code=sm_53 -gencode arch=compute_53,code=compute_53 -o fp16_dev.o -c fp16_dev.cu
g++ -I/usr/local/cuda/include -IFreeImage/include   -o fp16_emu.o -c fp16_emu.cpp
g++ -I/usr/local/cuda/include -IFreeImage/include   -o mnistCUDNN.o -c mnistCUDNN.cpp
/usr/local/cuda/bin/nvcc -ccbin g++   -m64      -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_53,code=sm_53 -gencode arch=compute_53,code=compute_53 -o mnistCUDNN fp16_dev.o fp16_emu.o mnistCUDNN.o -I/usr/local/cuda/include -IFreeImage/include  -LFreeImage/lib/linux/x86_64 -LFreeImage/lib/linux -lcudart -lcublas -lcudnn -lfreeimage -lstdc++ -lm

# ./mnistCUDNN
cudnnGetVersion() : 7301 , CUDNN_VERSION from cudnn.h : 7301 (7.3.1)
Host compiler version : GCC 5.4.0
There are 1 CUDA capable devices on your machine :
device 0 : sms 30  Capabilities 6.1, SmClock 1531.0 Mhz, MemSize (Mb) 22919, MemClock 3615.0 Mhz, Ecc=1, boardGroupID=0
Using device 0

Testing single precision
Loading image data/one_28x28.pgm
Performing forward propagation ...
Testing cudnnGetConvolutionForwardAlgorithm ...
Fastest algorithm is Algo 1
Testing cudnnFindConvolutionForwardAlgorithm ...
^^^^ CUDNN_STATUS_SUCCESS for Algo 2: 0.076544 time requiring 57600 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 1: 0.076800 time requiring 3464 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 0: 0.077824 time requiring 0 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 7: 0.108544 time requiring 2057744 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 5: 0.132064 time requiring 203008 memory
Resulting weights from Softmax:
0.0000000 0.9999399 0.0000000 0.0000000 0.0000561 0.0000000 0.0000012 0.0000017 0.0000010 0.0000000 
Loading image data/three_28x28.pgm
Performing forward propagation ...
Resulting weights from Softmax:
0.0000000 0.0000000 0.0000000 0.9999288 0.0000000 0.0000711 0.0000000 0.0000000 0.0000000 0.0000000 
Loading image data/five_28x28.pgm
Performing forward propagation ...
Resulting weights from Softmax:
0.0000000 0.0000008 0.0000000 0.0000002 0.0000000 0.9999820 0.0000154 0.0000000 0.0000012 0.0000006 

Result of classification: 1 3 5

Test passed!

Testing half precision (math in single precision)
Loading image data/one_28x28.pgm
Performing forward propagation ...
Testing cudnnGetConvolutionForwardAlgorithm ...
Fastest algorithm is Algo 1
Testing cudnnFindConvolutionForwardAlgorithm ...
^^^^ CUDNN_STATUS_SUCCESS for Algo 0: 0.048128 time requiring 0 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 2: 0.066560 time requiring 28800 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 1: 0.068608 time requiring 3464 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 4: 0.106496 time requiring 207360 memory
^^^^ CUDNN_STATUS_SUCCESS for Algo 7: 0.122880 time requiring 2057744 memory
Resulting weights from Softmax:
0.0000001 1.0000000 0.0000001 0.0000000 0.0000563 0.0000001 0.0000012 0.0000017 0.0000010 0.0000001 
Loading image data/three_28x28.pgm
Performing forward propagation ...
Resulting weights from Softmax:
0.0000000 0.0000000 0.0000000 1.0000000 0.0000000 0.0000714 0.0000000 0.0000000 0.0000000 0.0000000 
Loading image data/five_28x28.pgm
Performing forward propagation ...
Resulting weights from Softmax:
0.0000000 0.0000008 0.0000000 0.0000002 0.0000000 1.0000000 0.0000154 0.0000000 0.0000012 0.0000006 

Result of classification: 1 3 5

Test passed!
```
测试通过！

# 安装 TensorFlow
pip方式安装tensorflow-gpu:

```
# sudo pip3 install --upgrade tensorflow-gpu
```

验证：

```
# python3
Python 3.5.2 (default, Nov 23 2017, 16:37:01) 
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import tensorflow as tf
>>> hello = tf.constant("hello, tensorflow!")
>>> sess = tf.Session()
2018-10-30 18:20:35.827268: I tensorflow/core/platform/cpu_feature_guard.cc:141] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA
2018-10-30 18:20:36.260805: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:964] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero
2018-10-30 18:20:36.261468: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1411] Found device 0 with properties: 
name: Tesla P40 major: 6 minor: 1 memoryClockRate(GHz): 1.531
pciBusID: 0000:00:07.0
totalMemory: 22.38GiB freeMemory: 22.22GiB
2018-10-30 18:20:36.261505: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1490] Adding visible gpu devices: 0
2018-10-30 18:20:36.580288: I tensorflow/core/common_runtime/gpu/gpu_device.cc:971] Device interconnect StreamExecutor with strength 1 edge matrix:
2018-10-30 18:20:36.580362: I tensorflow/core/common_runtime/gpu/gpu_device.cc:977]      0 
2018-10-30 18:20:36.580371: I tensorflow/core/common_runtime/gpu/gpu_device.cc:990] 0:   N 
2018-10-30 18:20:36.580917: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1103] Created TensorFlow device (/job:localhost/replica:0/task:0/device:GPU:0 with 21549 MB memory) -> physical GPU (device: 0, name: Tesla P40, pci bus id: 0000:00:07.0, compute capability: 6.1)
>>> print(sess.run(hello))
b'hello, tensorflow!'
```


运行正常！

此时打开另一个Terminal窗口，可以看到process信息：

```
# nvidia-smi 
Tue Oct 30 18:21:48 2018       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 396.44                 Driver Version: 396.44                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla P40           Off  | 00000000:00:07.0 Off |                    0 |
| N/A   31C    P0    50W / 250W |  21785MiB / 22919MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0      6800      C   python3                                    21775MiB |
+-----------------------------------------------------------------------------+
```

# TensorBoard 可视化工具

可以用 TensorBoard 来展现 TensorFlow 图，绘制图像生成的定量指标图以及显示附加数据（如其中传递的图像）。

通过 pip 安装 TensorFlow 时，也会自动安装 TensorBoard：

```
# pip3 show tensorboard
Name: tensorboard
Version: 1.11.0
Summary: TensorBoard lets you watch Tensors Flow
Home-page: https://github.com/tensorflow/tensorboard
Author: Google Inc.
Author-email: opensource@google.com
License: Apache 2.0
Location: /usr/local/lib/python3.5/dist-packages
Requires: numpy, six, markdown, wheel, werkzeug, protobuf, grpcio
Required-by: tensorflow-gpu
```


启动TensorBoard：

```
# nohup tensorboard --logdir /tmp/tensorflow &
```


查看启动日志，TensorBoard端口为6006：

```
TensorBoard 1.11.0 at http://libing-GPU:6006 (Press CTRL+C to quit)
```

TensorBoard登录地址为：<ip address>:6006

浏览器登录正常！

# 安装Jupyter

Jupyter是一个交互式的笔记本，可以很方便地创建和共享文学化程序文档，支持实时代码，数学方程，可视化和 markdown。一般用与做数据清理和转换，数值模拟，统计建模，机器学习等等。

安装Jupyter：

```
# sudo pip3 install jupyter
```

生成配置文件：

```
# jupyter notebook --generate-config
Writing default config to: /root/.jupyter/jupyter_notebook_config.py
```

生成Jupyter密码：

```
# python3
Python 3.5.2 (default, Nov 23 2017, 16:37:01) 
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from notebook.auth import passwd;
>>> passwd()
Enter password: 
Verify password: 
'sha1:33acb81927e6:8db2c804e9015f47853ddc14f4fa92948d944b93'
```


将生成的hash串写入Jupyter配置文件：

```
## Hashed password to use for web authentication.
#  
#  To generate, type in a python/IPython shell:
#  
#    from notebook.auth import passwd; passwd()
#  
#  The string should be of the form type:salt:hashed-password.
c.NotebookApp.password = 'sha1:33acb81927e6:8db2c804e9015f47853ddc14f4fa92948d944b93'
```


保存后，启动Jupyter（启动用户为root，10.0.0.10为本机地址）：

```
# nohup jupyter notebook --allow-root --ip='10.0.0.10' &
```

查看日志，Jupyter已正常启动：

```
[I 10:02:02.468 NotebookApp] Serving notebooks from local directory: /root/jupyter
[I 10:02:02.468 NotebookApp] The Jupyter Notebook is running at:
[I 10:02:02.468 NotebookApp] http://10.0.0.10:8888/
[I 10:02:02.468 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[W 10:02:02.468 NotebookApp] No web browser found: could not locate runnable browser.
```


浏览器访问Jupyter，地址为：<ip address>:8888
（本例中116.196.118.38为云主机的公网IP）：

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/Jupyter-login.png?raw=true)

输入密码后登陆：访问正常！

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/Jupyter-mainpage.png?raw=true)

# 运行TensorFlow Demo示例

Jupyter中新建 HelloWorld 示例，代码如下：

```
import tensorflow as tf

# Simple hello world using TensorFlow

# Create a Constant op
# The op is added as a node to the default graph.
#
# The value returned by the constructor represents the output
# of the Constant op.
hello = tf.constant('Hello, TensorFlow!')

# Start tf session
sess = tf.Session()

# Run the op
print(sess.run(hello))
```


运行示例：

![image](https://github.com/bingli7/knowledge-base/blob/master/image/JDGPU-tf/TensorFlow-Demo.png?raw=true)

运行正常！