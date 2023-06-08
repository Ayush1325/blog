+++
title = "GSoC23: Getting started with Zephyr for BeaglePlay CC1352"
description = "A short introduction to getting started with BeaglePlay CC1352 firmware development"
date = "2023-06-07T20:05:12+05:30"

[taxonomies]
tags = ["c", "gsoc23", "zephyr", "beagleboard"]
+++
Hello everyone. In this post, I will go over setting up BeaglePlay and my Linux host for BeaglePlay CC1352 firmware development. Check out my [GSoC 2023 Introduction post](@/post21.md) for more information about my project.

<!-- more -->

# Setup Linux host for Zephyr
While it is possible to do Zephyr development on BeaglePlay itself, I will use my Linux PC to program and build the Zephyr application. This is because my Linux PC is much more powerful than BeaglePlay and thus plays well with Neovim + Clangd LSP. 

So, I will now go over setting up the Linux host for development.

## Setup Zephyr SDK
1. Install [host dependencies](https://docs.zephyrproject.org/latest/develop/getting_started/installation_linux.html#installation-linux).
2. Download Zephyr SDK Bundle.
```sh
cd ~
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/zephyr-sdk-0.16.1_linux-x86_64.tar.xz
wget -O - https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/sha256.sum | shasum --check --ignore-missing
```
3. Extract the Zephyr SDK bundle archive. I extracted the bundle to `~/.local/opt/zephyr-sdk-0.16.1`
```sh
tar xvf zephyr-sdk-0.16.1_linux-x86_64.tar.xz
```
4. Run the Zephyr SDK bundle setup script:
```sh
~/.local/opt/zephyr-sdk-0.16.1/setup.sh
```

Since I will not use my Linux host for flashing, I did not need the udev rules.

## Setup Zephyr Builddir
I will be using a Zephyr fork with support for [beagleplay-cc1352](https://git.beagleboard.org/beagleconnect/zephyr/zephyr).

1. Create a new Python virtual environment.
```sh
python -m venv ~/zephyrproject/.venv && cd ~/zephyrproject
```
2. Activate virtualenv. Since I am using [zsh-autoswitch-virtualenv](https://github.com/MichaelAquilina/zsh-autoswitch-virtualenv), this automatically happens when I enter the directory.
3. Install west:
```sh
pip install west
```
4. Get the Zephyr source code:
```sh
west init -m https://git.beagleboard.org/beagleconnect/zephyr/zephyr ~/zephyrproject
cd ~/zephyrproject
west update
```
5. Export a Zephyr CMake package. This allows CMake to automatically load boilerplate code required for building Zephyr applications.
```sh
west zephyr-export
```
6. Zephyr’s scripts/requirements.txt file declares additional Python dependencies. Install them with pip.
```sh
pip install -r ~/zephyrproject/zephyr/scripts/requirements.txt
```

# Build a sample Zephyr application
I will be using the [`samples/net/dns_resolve`](https://git.beagleboard.org/beagleconnect/zephyr/zephyr/-/tree/sdk/samples/net/dns_resolve).
1. Build the Zephyr Application.
```sh
west build -b beagleplay_cc1352 -p always ~/zephyrproject/samples/net/dns_resolve
```
2. Copy the binary to BeaglePlay
```sh
scp ~/zephyrproject/build/zephyr/zephyr.bin debian@beagleplay.local:~/
```

# Run the application on BeaglePlay CC1352
1. Enable `k3-am625-beagleplay-bcfserial-no-firmware.dts` overlay. This is needed to ensure the bcfserial driver isn’t blocking the serial port.
```sh
echo "    fdtoverlays /overlays/k3-am625-beagleplay-bcfserial-no-firmware.dtbo" | sudo tee -a /boot/firmware/extlinux/extlinux.conf
sudo shutdown -r now
```
2. Clone [cc1352-flasher](https://git.beagleboard.org/beagleconnect/cc1352-flasher)
```sh
git clone git@git.beagleboard.org:beagleconnect/cc1352-flasher.git && cd cc1352-flasher
```
3. We need to checkout the `from-20230521` branch.
```sh
git checkout from-20230521
```
4. Flash the application
```sh
python cc1352-flasher --beagleplay ~/zephyr.bin
```
5. Try out the application:
```sh
tio /dev/ttyACM0
```

# Conclusion
The BeaglePlay CC1352 Firmware I am working on can be found [here](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware). You can follow my GSoC23-related blog posts using this [feed](https://www.programmershideaway.xyz/tags/gsoc23/atom.xml).

Consider [supporting me](@/pages/supportme.md) if you like my work.
