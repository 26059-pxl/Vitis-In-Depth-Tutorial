# 3.7 Acceleration with HLS kernel

**_Note_**: You need to use the new image you generated in Section_3-Module_7. After the OS boot succesfully, you can refer to [quick_start](https://github.com/Xilinx/Vitis-AI/tree/master/Vitis-AI-Library#quick-start-for-edge) guide to learn how to prepare the development evironment.
This example suite, for the Vitis AI Library and Vitis Accelerated Kernel, shows how to use the Vitis AI Library runs neural networks on DPUs and how to use the HLS kernel to speed up pre/... Postprocessing. About how to immigrate from OpenCV to HLS, please refer to [README.md](app/README.md)

Some system level functions:

- working with DRM on ZynqMP

  - Using the DRM to display

- V4L2 initialization and control
  - Streaming video capture from V4L2, such as USB camera.

The directory structure and brief explanations as below:

```bash
├── app
│   ├── include                    #headers directory of work pipeline
│   │   ├── decodethread.hpp       #header file of decode thread
│   │   ├── dputhread.hpp          #header file of dpu threadd
│   │   ├── filter.hpp             #dpu filer header 
│   │   ├── frameinfo.hpp          #important data structure 
│   │   ├── guithread.hpp          #display thread headerfile
│   │   ├── mythread.hpp           #thread pool 
│   │   └── sortthread.hpp         #sort thread header
│   ├── README.md                  #instruction of how to migrate from OpenCV to HLS
│   ├── src                        #implementation of the work threads
│   │   ├── decodethread.cpp
│   │   ├── dputhread.cpp
│   │   ├── guithread.cpp
│   │   ├── mythread.cpp
│   │   └── sortthread.cpp
│   └── test
│       └── usb_input_multi_threads_xcl_refinedet.cpp #main application
├── kernel                            #HLS kernel work directory
│   ├── build                         #directory of config file
│   │   ├── dpu_conf.vh               #dpu IP config file
│   │   ├── Makefile                  #The Makefile used to replace the file in ${DPU_TRD}
│   │   ├── preprocessor_config.ini   #config file of pre_processor
│   │   └── prj_config_104_2dpu       #Config file required be Makefile for IP integration
│   ├── flash_sd_card.sh              #Script to generate the OS image
│   └── src                           #HLS kernel source codes directory
│       ├── pre_processor.cpp   
│       └── pre_processor.h
├── my_V4l2s                          # V4l2 libraries work directory to get image data from USB camera
│   ├── CMakeLists.txt                # cmake file to build libs
│   ├── include                       
│   │   ├── dpdrm.hpp                 #Encapsulate drm library
│   │   ├── line_exception.hpp        #Encapsulate throw error
│   │   ├── V4l2Access.hpp            #Interface to access the camera device
│   │   ├── V4l2Capture.hpp           #Methods to get the image
│   │   ├── V4l2Device.hpp            #Device capablity 
│   │   ├── V4l2MmapDevice.hpp        #Device type
│   │   ├── V4l2Output.hpp            #asistant class
│   │   ├── V4l2ReadWriteDevice.hpp   #Device type
│   │   └── xcl2.hpp                  #Encapsulate the kernel usage
│   ├── src
│   │   ├── V4l2Access.cpp
│   │   ├── V4l2Capture.cpp
│   │   ├── V4l2Device.cpp
│   │   ├── V4l2MmapDevice.cpp
│   │   ├── V4l2Output.cpp
│   │   ├── V4l2ReadWriteDevice.cpp
│   │   └── xcl2.cpp
│   └── test
│       ├── build.sh                #compile script
│       ├── test_hls_kernel.cpp     #test case for hls kernel
│       └── test.yuv                #test data
└── README.md
└── CMakeLists.txt

```

## Platform integrated

- Hardware required:
  - ZCU104 evaluation board
  - Micro-USB cable, connect to lattop for the terminal emulator.
  - SD card to burn the OS image.
- Software required:
- Vitis 2019.2 [Vitis Core Development Kit](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vitis/2019-2.html) install in server.
- [Silicon Labs quad CP210x USB-to-UART bridge driver](https://www.silabs.com/products/development-tools/software/usb-to-uart-bridge-vcp-drivers) install in laptop.
- Serial terminal emulator e.g. teraterm, Mobaxterm install in laptop
- [XRT 2019.2](https://github.com/Xilinx/XRT/tree/2019.2) install in laptop
- [zcu104 dpu platform](https://github.com/Xilinx/Vitis_Embedded_Platform_Source/tree/2019.2/Xilinx_Official_Platforms/zcu104_dpu) install in server
- [Vitis AI runtime package](https://xilinx-ax-dl.entitlenow.com/dl/ul/2020/04/27/R210316037/vitis-ai-runtime-1.1.2.tar.gz/73b825a7fd3fb54fa9693eeb634ac491/5F0EBBB3?akdm=0&filename=vitis-ai-runtime-1.1.2.tar.gz)
- [Vitis AI model packages for ZCU104](https://xilinx-ax-dl.entitlenow.com/dl/ul/2020/03/19/R210290235/vitis_ai_model_ZCU104_2019.2-r1.1.0.deb/960da8e4ed428df080602e9ba62e09f3/5F0EBB60?akdm=0&filename=vitis_ai_model_ZCU104_2019.2-r1.1.0.deb)

---

## [Building the Vitis Platform](https://github.com/Xilinx/Vitis_Embedded_Platform_Source/tree/2019.2/Xilinx_Official_Platforms/zcu104_dpu)

Last Tested Vivado Release: 2019.2
The platform build process is entirely scripted. Note that as this platform build process involves cross-compiling Linux, build of the platform is supported on Linux environments only (although it is possible to build inside a VM or Docker container).

Also note that the default PetaLinux configuration uses local scratchpad areas. This will not work if you are building on a networked file system; Yocto will error out. Update PetaLinux to change the build area to a locally-mounted hard drive (most Xilinx internal network servers have a /scratch or /tmp area for this purpose).

After cloning the platform source, and with both Vivado and PetaLinux set up, run make from the top-level platform directory.

Note that by default this Makefile will install the platform to "platform_repo/zcu104_dpu/export/zcu104_dpu/"

### Installing the Yocto SDK

A bundled Yocto SDK "sysroot" is not available with this package by default. To build non-trivial Linux software for this platform sysroot need to be built and installed. This can be done with command "make peta_sysroot" It is installed to "platform_repo/sysroot" once the build completes.

To cross-compile against this platform from the command line, source the environment-setup-aarch64-xilinx-linux script to set up your environment (cross compiler, build tools, libraries, etc).

### Build instructions

This packages comes with sources to generate hardware specification file (xsa) from Vivado, petalinux sources to generate the image.ub and platform sources to generate the Vitis platform.

Build platform from scratch: make all

Build a platform without modifying hardware: make petalinux_proj XSA_DIR= make pfm XSA_DIR=

```
example:
	make petalinux_proj XSA_DIR=/home/user/zcu104_dpu/vivado
	make pfm /home/user/zcu104_dpu/vivado
```

---

### IP Integrated

- Step1: Use below commands to clone the Vitis-AI from github project.

```
$ git clone git@github.com:Xilinx/Vitis-AI.git
```

- Step2: copy the files below to under the directory "\${Vitis-AI/DPU-TRD}DPU-TRD/prj/Vitis"

```
$ cp ${Module_7}/kernel/build/* ${Vitis-AI/DPU-TRD}DPU-TRD/prj/Vitis
$ cp ${Module_7}/kernel/src/* ${Vitis-AI/DPU-TRD}DPU-TRD/prj/Vitis
```

- Step3: Run the below commands to start IP integration and wait for it to complete.

```
$ cd ${Module_7}/kernel/src/* ${Vitis-AI/DPU-TRD}DPU-TRD/prj/Vitis
make -j
```

- Step4: Generate the system image

```
cp ${Module_7}/kernel/flash_sd_card.sh  ${Vitis-AI}/DPU-TRD/prj/Vitis/binary_container_1/sd_card
cd ${Module_7}/kernel/flash_sd_card.sh ${Vitis-AI}/DPU-TRD/prj/Vitis/binary_container_1/sd_card
sudo ./flash_sd_card.sh
```

- Step5: Use etcher or other tools to burn the image to 16GB sd card.
- Step6: Set the Mode to SD card

  ```
  Rev 1.0:  SW6[4:1] - off, off, off, on
  Rev D2:   SW6[4:1] - on, off, on, off
  ```

### Board development setting.

How to setup the board environment you could refer to [Module_3/README.md](../../Module_3/README.md)

## Setting Up the cross-compile environment and Build app.
---

- After cloning the project, use the following commands to compile the applications. Regard the sdk path to be ${SDK_PATH}.


```
$ source ${SDK_PATH}/environment-setup-aarch64-xilinx-linux
$ cd ${Section_3-Basic}/Module_7/
$ sh build_app.sh
```
Copy the executable file to board.

```
$ scp ${Section_3-Basic}/Module_7/build/usb_input_multi_threads_refinedet_drm root@$[IP_OF_BOARD]:/home/root
$ scp ${Section_3-Basic}/Module_7/build/myVHLS_V4l2s/libhls_v4l2s.so root@$[IP_OF_BOARD]:/usr/lib
```
Running zynqmp_dpu_optimize.sh on board to optimize the board setting.
```
#cd ~/dpu_sw_optimize/zynqmp/
#./zynqmp_dpu_optimize.sh
```
The script runs automatically after the board boots up with the official image
- Test the HLS kernel to see if it works properly with the test example. When the test example works, you will notice that two PNG files are generated in your execution directory.
```
# chmod +x test_hls_kernel
# ./test_hls_kernel
```


- Run the application.
 Before you run, it's needed to stop the weston service to use drm display mode for the application.
```
  ./usb_input_multi_threads_refinedet_hls_drm refinedet_pruned_0_8 0 -t 3
```
- DESCRIPTION
```
    - refinedet_pruned_0_8: The model used;
    - 0                   :  Camera input;
    - -t 3                :  thread number 3;

```

<p align="center"><sup>Copyright&copy; 2020 Xilinx</sup></p>
