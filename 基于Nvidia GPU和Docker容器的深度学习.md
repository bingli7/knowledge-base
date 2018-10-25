# 基于Nvidia GPU和Docker容器的深度学习 #

**GPU云主机：**

操作系统：Ubuntu 16.04 64位  
GPU：	1 x Nvidia Tesla P40  

## 1. 安装CUDA Driver ##

### 1.1 Pre-installation Actions ###

安装gcc、g++、make：

    # sudo apt-get install gcc g++ make
    # gcc --version
    gcc (Ubuntu 5.4.0-6ubuntu1~16.04.10) 5.4.0 20160609
    Copyright (C) 2015 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

如果没有，需安装linux-headers：

    # sudo apt-get install linux-headers-$(uname -r)


### 1.2 安装NVIDIA driver ###

CUDA安装有两种方式：  
1.Package安装  
2.Runfile安装  

本文选择**runfile**安装方式。

首先禁用Nouveau：

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

Reboot云主机：

    # reboot

重启后check下Nouveau drivers没有被load：

    # lsmod | grep nouveau
    # 

登录：http://developer.nvidia.com/cuda-downloads 下载相应的runfile：

    # wget https://developer.nvidia.com/compute/cuda/10.0/Prod/local_installers/cuda_10.0.130_410.48_linux

开始安装CUDA Driver:

    # chmod +x cuda_10.0.130_410.48_linux
    # sudo sh ./cuda_10.0.130_410.48_linux 
    Logging to /tmp/cuda_install_1699.log
    Using more to view the EULA.
    Do you accept the previously read EULA?
    accept/decline/quit: accept
    
    Install NVIDIA Accelerated Graphics Driver for Linux-x86_64 410.48?
    (y)es/(n)o/(q)uit: y
    
    Do you want to install the OpenGL libraries?
    (y)es/(n)o/(q)uit [ default is yes ]: y
    
    Do you want to run nvidia-xconfig?
    This will update the system X configuration file so that the NVIDIA X driver
    is used. The pre-existing X configuration file will be backed up.
    This option should not be used on systems that require a custom
    X configuration, such as systems with multiple GPU vendors.
    (y)es/(n)o/(q)uit [ default is no ]: 
    
    Install the CUDA 10.0 Toolkit?
    (y)es/(n)o/(q)uit: y
    
    Enter Toolkit Location
     [ default is /usr/local/cuda-10.0 ]: 
    
    Do you want to install a symbolic link at /usr/local/cuda?
    (y)es/(n)o/(q)uit: y
    
    Install the CUDA 10.0 Samples?
    (y)es/(n)o/(q)uit: y
    
    Enter CUDA Samples Location
     [ default is /root ]: 
    
    Installing the NVIDIA display driver...
    Installing the CUDA Toolkit in /usr/local/cuda-10.0 ...
    Missing recommended library: libGLU.so
    Missing recommended library: libX11.so
    Missing recommended library: libXi.so
    Missing recommended library: libXmu.so
    
    Installing the CUDA Samples in /root ...
    Copying samples to /root/NVIDIA_CUDA-10.0_Samples now...
    Finished copying samples.
    
    ===========
    = Summary =
    ===========
    
    Driver:   Installed
    Toolkit:  Installed in /usr/local/cuda-10.0
    Samples:  Installed in /root, but missing recommended libraries
    
    Please make sure that
     -   PATH includes /usr/local/cuda-10.0/bin
     -   LD_LIBRARY_PATH includes /usr/local/cuda-10.0/lib64, or, add /usr/local/cuda-10.0/lib64 to /etc/ld.so.conf and run ldconfig as root
    
    To uninstall the CUDA Toolkit, run the uninstall script in /usr/local/cuda-10.0/bin
    To uninstall the NVIDIA Driver, run nvidia-uninstall
    
    Please see CUDA_Installation_Guide_Linux.pdf in /usr/local/cuda-10.0/doc/pdf for detailed information on setting up CUDA.
    
    Logfile is /tmp/cuda_install_1699.log

安装成功！


Reboot云主机：

    # reboot

设备验证：

    # ls /dev/nvidia*
    ls: cannot access '/dev/nvidia*': No such file or directory
    # vi nvidia-probe.sh
    
    #!/bin/bash
    ### BEGIN INIT INFO
    # Provides:          jd.com
    # Required-Start:    $local_fs $network
    # Required-Stop:     $local_fs
    # Default-Start:     2 3 4 5
    # Default-Stop:      0 1 6
    # Short-Description: nvidia service
    # Description:       nvidia service daemon
    ### END INIT INFO
    
    /sbin/modprobe nvidia
    
    if [ "$?" -eq 0 ]; then
      # Count the number of NVIDIA controllers found.
      NVDEVS=`lspci | grep -i NVIDIA`
      N3D=`echo "$NVDEVS" | grep "3D controller" | wc -l`
      NVGA=`echo "$NVDEVS" | grep "VGA compatible controller" | wc -l`
    
      N=`expr $N3D + $NVGA - 1`
      for i in `seq 0 $N`; do
    mknod -m 666 /dev/nvidia$i c 195 $i
      done
    
      mknod -m 666 /dev/nvidiactl c 195 255
    
    else
      exit 1
    fi
    
    /sbin/modprobe nvidia-uvm
    
    if [ "$?" -eq 0 ]; then
      # Find out the major device number used by the nvidia-uvm driver
      D=`grep nvidia-uvm /proc/devices | awk '{print $1}'`
    
      mknod -m 666 /dev/nvidia-uvm c $D 0
    else
      exit 1
    fi
        
    # chmod +x nvidia-probe.sh 
    # ./nvidia-probe.sh
    # ls /dev/nvidia*
    /dev/nvidia0  /dev/nvidiactl  /dev/nvidia-uvm

/dev下成功发现设备!

配置开机自启动：

    # cp nvidia-probe.sh /etc/init.d/
    # sudo update-rc.d nvidia-probe.sh defaults 95


### 1.3 Post-installation Actions ###

配置环境变量：
    
    # vi /etc/profile
    ......
    export PATH=/usr/local/cuda-10.0/bin${PATH:+:${PATH}}
    export LD_LIBRARY_PATH=/usr/local/cuda-10.0/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
    
开机启动Persistence Daemon：

    # vi /etc/rc.local
    ......
    /usr/bin/nvidia-persistenced --verbose
    
    exit 0


### 1.4 CUDA driver验证 ###

查看Driver Version：
    
    # cat /proc/driver/nvidia/version
    NVRM version: NVIDIA UNIX x86_64 Kernel Module  410.48  Thu Sep  6 06:36:33 CDT 2018
    GCC version:  gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10) 

使用deviceQuery示例验证：
    
    # cd ~/NVIDIA_CUDA-10.0_Samples/1_Utilities/deviceQuery/
    # make
    "/usr/local/cuda-10.0"/bin/nvcc -ccbin g++ -I../../common/inc  -m64-gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_75,code=compute_75 -o deviceQuery.o -c deviceQuery.cpp
    "/usr/local/cuda-10.0"/bin/nvcc -ccbin g++   -m64  -gencode arch=compute_30,code=sm_30 -gencode arch=compute_35,code=sm_35 -gencode arch=compute_37,code=sm_37 -gencode arch=compute_50,code=sm_50 -gencode arch=compute_52,code=sm_52 -gencode arch=compute_60,code=sm_60 -gencode arch=compute_61,code=sm_61 -gencode arch=compute_70,code=sm_70 -gencode arch=compute_75,code=sm_75 -gencode arch=compute_75,code=compute_75 -o deviceQuery deviceQuery.o 
    mkdir -p ../../bin/x86_64/linux/release
    cp deviceQuery ../../bin/x86_64/linux/release
    # cd ../../bin/x86_64/linux/release/
    # ls
    deviceQuery
    # ./deviceQuery 
    ./deviceQuery Starting...
    
     CUDA Device Query (Runtime API) version (CUDART static linking)
    
    Detected 1 CUDA Capable device(s)
    
    Device 0: "Tesla P40"
      CUDA Driver Version / Runtime Version  10.0 / 10.0
      CUDA Capability Major/Minor version number:6.1
      Total amount of global memory: 22919 MBytes (24032378880 bytes)
      (30) Multiprocessors, (128) CUDA Cores/MP: 3840 CUDA Cores
      GPU Max Clock rate:1531 MHz (1.53 GHz)
      Memory Clock rate: 3615 Mhz
      Memory Bus Width:  384-bit
      L2 Cache Size: 3145728 bytes
      Maximum Texture Dimension Size (x,y,z) 1D=(131072), 2D=(131072, 65536), 3D=(16384, 16384, 16384)
      Maximum Layered 1D Texture Size, (num) layers  1D=(32768), 2048 layers
      Maximum Layered 2D Texture Size, (num) layers  2D=(32768, 32768), 2048 layers
      Total amount of constant memory:   65536 bytes
      Total amount of shared memory per block:   49152 bytes
      Total number of registers available per block: 65536
      Warp size: 32
      Maximum number of threads per multiprocessor:  2048
      Maximum number of threads per block:   1024
      Max dimension size of a thread block (x,y,z): (1024, 1024, 64)
      Max dimension size of a grid size(x,y,z): (2147483647, 65535, 65535)
      Maximum memory pitch:  2147483647 bytes
      Texture alignment: 512 bytes
      Concurrent copy and kernel execution:  Yes with 2 copy engine(s)
      Run time limit on kernels: No
      Integrated GPU sharing Host Memory:No
      Support host page-locked memory mapping:   Yes
      Alignment requirement for Surfaces:Yes
      Device has ECC support:Enabled
      Device supports Unified Addressing (UVA):  Yes
      Device supports Compute Preemption:Yes
      Supports Cooperative Kernel Launch:Yes
      Supports MultiDevice Co-op Kernel Launch:  Yes
      Device PCI Domain ID / Bus ID / location ID:   0 / 0 / 7
      Compute Mode:
     < Default (multiple host threads can use ::cudaSetDevice() with device simultaneously) >
    
    deviceQuery, CUDA Driver = CUDART, CUDA Driver Version = 10.0, CUDA Runtime Version = 10.0, NumDevs = 1
    Result = PASS
    
参考：
> https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#how-do-i-install-the-nvidia-driver

> https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html

## 2. 安装Nvidia-docker ##

### 2.1 安装Docker ###


安装docker-ce：
    
    #sudo apt-get remove docker docker-engine docker.io
    
    # sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
    # curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    # sudo add-apt-repository \
       "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
       $(lsb_release -cs) \
       stable"
    # sudo apt-get update
    # sudo apt-get install docker-ce
    # docker version
    Client:
     Version:   18.06.1-ce
     API version:   1.38
     Go version:go1.10.3
     Git commit:e68fc7a
     Built: Tue Aug 21 17:24:56 2018
     OS/Arch:   linux/amd64
     Experimental:  false
    
    Server:
     Engine:
      Version:  18.06.1-ce
      API version:  1.38 (minimum version 1.12)
      Go version:   go1.10.3
      Git commit:   e68fc7a
      Built:Tue Aug 21 17:23:21 2018
      OS/Arch:  linux/amd64
      Experimental: false
    
### 2.2 安装nvidia-docker ###

安装nvidia-docker：
    
    # Add the package repositories
    curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
      sudo apt-key add -
    distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
    curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
      sudo tee /etc/apt/sources.list.d/nvidia-docker.list
    sudo apt-get update
    
    # Install nvidia-docker2 and reload the Docker daemon configuration
    sudo apt-get install -y nvidia-docker2
    sudo pkill -SIGHUP dockerd
    
验证nvidia-docker：
    
    # docker run --runtime=nvidia --rm nvidia/cuda:9.0-base nvidia-smi
    Thu Oct 25 09:03:27 2018   
    +-----------------------------------------------------------------------------+
    | NVIDIA-SMI 410.48 Driver Version: 410.48|
    |-------------------------------+----------------------+----------------------+
    | GPU  NamePersistence-M| Bus-IdDisp.A | Volatile Uncorr. ECC |
    | Fan  Temp  Perf  Pwr:Usage/Cap| Memory-Usage | GPU-Util  Compute M. |
    |===============================+======================+======================|
    |   0  Tesla P40   On   | 00000000:00:07.0 Off |0 |
    | N/A   20CP8 9W / 250W |  0MiB / 22919MiB |  1%  Default |
    +-------------------------------+----------------------+----------------------+
       
    +-----------------------------------------------------------------------------+
    | Processes:   GPU Memory |
    |  GPU   PID   Type   Process name Usage  |
    |=============================================================================|
    |  No running processes found |
    +-----------------------------------------------------------------------------+
   
 
### 2.3 配置Docker默认runtime ###
    
cat /etc/docker/daemon.json
  
    {
        "default-runtime": "nvidia",
        "runtimes": {
            "nvidia": {
                "path": "nvidia-container-runtime",
                "runtimeArgs": []
            }
        }
    }

重启服务：  

    # systemctl restart docker
    # systemctl status docker

### 2.4 运行TensorFlow卷积神经Model ###

Docker运行：

    # docker run --rm --name tensorflow -ti tensorflow/tensorflow:r0.9-devel-gpu
    root@bd0fb3758da2:~# python --version
    Python 2.7.6
    root@bd0fb3758da2:~# python -m tensorflow.models.image.mnist.convolutional

参考：

> https://docs.docker.com/install/linux/docker-ce/ubuntu/

> https://github.com/NVIDIA/nvidia-docker#quickstart