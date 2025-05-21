# Updating JetPack (flashing the Jetson)

- Read all of this, especially all the linked docs, and be careful. All the data on the Jetson will be PERMANENTLY erased!
- Required components
  - Any NVIDIA SSO account
  - 2x ethernet cables
  - USB-C 3.x cable (plugged into the correct port, for AGX Orion 64GB the side with the GPIO pins)
  - Display port and monitor for Jetson
  - A computer running Docker
- The GUI tool makes changes to your host operating system, it is not recommended to use it
  - The Docker container is the best way. It has a fancy ncurses CLI, it's reasonably easy to use
  - Download the SDK docker image https://developer.nvidia.com/sdk-manager
  - Follow [this tutorial on the NVIDIA website](https://docs.nvidia.com/sdk-manager/docker-containers/index.html)
  - See commands under ["Additional Considerations"](https://docs.nvidia.com/sdk-manager/docker-containers/index.html#additional-considerations), copy-pasted to [Appendix A: Example commands to flash JetPack](#appendix-a-example-commands-to-flash-jetpack-62-onto-orion-agx-64gb)
- [This page](https://docs.nvidia.com/jetson/jetpack/install-setup/index.html) contains all the packages the JetPack should have installed. The SDK manager utility does not always finish doing this, so fill in the missing components under the command `jetson-release`
- It's required for your device to allow SSH passwords, as the Jetson SDK does most of its operations over SSH in this way
- It's not recommended to flash a Jetson over the university's network, as the JetPack SDK requires wake-on-LAN
- The SDK will try to put the Jetson into flashing mode, and sometimes this will fail. You may want to manually put it in flashing mode before you start

# Getting Intel RealSense cameras to work

Background of the issues and why this is here:
Discovered when looking to run 3 Intel RealSense D435i Depth Cameras on a Jetson AGX Orion 64GB running Jetpack 6.2.

when following the instruction in the README of the [realsense-ros repo](https://github.com/IntelRealSense/realsense-ros) (as of commit [1cbd81b](https://github.com/IntelRealSense/realsense-ros/commit/1cbd81be81e807eefb46f098e76381888ffc7001) on the 2nd Sept 2024) it is **Highly recommend** to build the wrapper package for ROS from source.

This is perhaps the **MOST IMPORTANT PART** to getting the working, install the Intel RealSense SDK 2.0 by building the `librealsense2` from source.  

This can be attempted a few way, but we have only tested the  methods below:

- clone the repo for `librealsense2`

```bash
git clone https://github.com/IntelRealSense/librealsense.git
```

- the simplest way that we found is to run `libuvc_installation.sh`, located in `librealsense/scripts/libuvc_installation.sh` of the librealsense repo
  > A copy for archival purposes can also be found in this repo [here](../scripts/realsense/libuvc_installation.sh)
  - During this you will be asked to remove any realsense cameras attached
  - This can take a while so maybe go get a coffee or something :wink:

- After the build and installation is done you can plug in the realsense camera and run `realsense-viewer` to check if you camera is detected

## RealSense D435i Specific

When trying to run a Realsense D435i we found that realsense viewer was failing to communicate with the camera.
To solve this found that flashing the camera with firmware **15.13.0.50** allowed the D435i to be detected.

A link to download **15.13.0.50** for the D435i can be found [here](https://www.intelrealsense.com/download/19295/?-545513842.1652797067)

# Known Issues
- Native ROS installs on AARCH do not come with gazebo. Docker is your friend


# Appendix A: Example commands to flash JetPack 6.2 onto Orion AGX 64GB
```bash
# To fix "Known Issues"
sudo apt install qemu-user-static binfmt-support
sudo update-binfmts --enable
sudo killall adb

# Download this on a GUI computer https://developer.download.nvidia.com/sdkmanager/redirects/sdkmanager-docker-image-ubuntu2204.html
# Fill in the gaps with these command
docker load -i ./sdkmanager-[version].[build#]-[base_OS]_docker.tar.gz
docker tag sdkmanager:[version].[build#] sdkmanager:latest

# Download everything we need
docker run -it --privileged           \
  -v /dev/bus/usb:/dev/bus/usb/       \
  -v /dev:/dev                        \
  -v /media/$USER:/media/nvidia:slave \
  --name JetPack_AGX_Orin_Devkit      \
  --network host                      \
  sdkmanager --cli                    \
    --action install                  \
    --login-type devzone              \
    --product Jetson                  \
    --target-os Linux                 \
    --version 6.2                     \
    --target JETSON_AGX_ORIN_TARGETS  \
    --flash                           \
    --license accept                  \
    --stay-logged-in true             \
    --collect-usage-data disable      \
    --exit-on-finish

docker commit JetPack_AGX_Orin_Devkit jetpack_agx_orin_devkit:6.2_flash
docker container rm JetPack_AGX_Orin_Devkit
docker run -it --rm --privileged -v /dev/bus/usb:/dev/bus/usb/ jetpack_agx_orin_devkit:6.2_flash
```
