+++
title = "GSoC23: Writing a Hello World Driver for BeaglePlay"
description = "A short introduction to getting started with BeaglePlay Driver development"
date = "2023-06-01T22:07:12+05:30"

[taxonomies]
tags = ["c", "gsoc23", "linux", "beagleboard"]
+++

Hello everyone. I will go over setting up BeaglePlay for Driver Development in this post. I will write a simple "Hello World" driver once BeaglePlay is set up to demonstrate this. Check out my [previous post](@/post21.md) for more information about my project.

<!-- more -->

# Setup BeaglePlay
## Update board with the latest software
1. Download the flasher image from [beagleboard.org distros](https://www.beagleboard.org/distros) page.
2. Flash this image onto an SD Card. I used [balenaEtcher](https://www.balena.io/etcher/) for this.
3. Insert SD Card and boot the board. The lights will start flashing in the following order: 0-1-2-3-2-1-0 in a loop. This signifies that image is being flashed into eMMC.
4. The board will power down once the flashing is over. Remove the SD Card now.

## Setup Networking
1. Connect the Board to Linux Host using USB.
2. SSH into the board. The default credentials are as follows:
    - **username:** debian
    - **password:** temppwd
    - **hostname:** beagleplay.local
3. Connect to WiFi using `wpa_cli`:
    1. Run `wpa_cli`.
    ```sh
    $ wpa_cli
    ```
    2. Use `scan` and `scan_results` to see available networks.
    ```sh
    > scan
    OK
    <3>CTRL-EVENT-SCAN-RESULTS
    > scan_results
    bssid / frequency / signal level / flags / ssid
    00:00:00:00:00:00 2462 -49 [WPA2-PSK-CCMP][ESS] MYSSID
    11:11:11:11:11:11 2437 -64 [WPA2-PSK-CCMP][ESS] ANOTHERSSID
    ```
    3. To associate with MYSSID, add the network, set the credentials, and enable it:
    ```sh
    > add_network
    0
    > set_network 0 ssid "MYSSID"
    > set_network 0 psk "passphrase"
    > enable_network 0
    <2>CTRL-EVENT-CONNECTED - Connection to 00:00:00:00:00:00 completed (reauth) [id=0 id_str=]
    ```
    4. If the SSID does not have password authentication, you must explicitly configure the network as keyless by replacing the command `set_network 0 psk "passphrase"` with `set_network 0 key_mgmt NONE`. 
    5. Finally, save this network in the configuration file and quit wpa_cli: 
    ```sh
    > save_config
    OK
    > quit
    ```

Now it should be possible to ssh into BeaglePlay without being connected using USB.

## Copy SSH key
Since we have to ssh into the BeaglePlay so many times, it is better to use ssh key instead of password.
1. Check if the ssh key is present:
```sh
ls -al ~/.ssh
```
2. [Generate new ssh key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) if not present.
3. Copy public key to BeaglePlay:
```sh
ssh-copy-id debian@beagleplay.local
```

You should now be able to ssh into BeaglePlay without a password.

# Build Kernel
While building the kernel module on BeaglePlay is possible, I prefer using my Linux host device for development. As such, I needed to have the BeaglePlay kernel source on my host.

1. Download the [beagle kernel] source using git:
```sh
git clone https://git.beagleboard.org/beagleboard/linux.git && cd linux
```
2. Check out the branch corresponding to BeaglePlay's Kernel. In my case, BeaglePlay was running `5.10.168`
```sh
git checkout v5.10.168-ti-arm64-r103
```
3. Copy the config from BeaglePlay.
```sh
scp debian@beagleplay.local:/boot/config-5.10.168-ti-arm64-r103 ./.config
```
3. Download dependencies. I am listing the fedora linux dependencies here:
```sh
sudo dnf install bison flex gcc-aarch64-linux-gnu -y
```
4. Generate config from the previous one:
```sh
make ARCH=arm64 CROSS_COMPILE=/usr/bin/aarch64-linux-gnu- oldconfig
```
5. Build the kernel
```sh
make ARCH=arm64 CROSS_COMPILE=/usr/bin/aarch64-linux-gnu- -j${nproc}
```
6. Generate `compiler_commands.json` to use LSP:
```sh
python scripts/clang-tools/gen_compile_commands.py
```

# Build Kernel Module
Now we will build a simple "Hello World" module for our BeaglePlay Kernel.
1. Create a new directory for our project:
```sh
mkdir hello-world
```
2. Write a `Makefile`:
```sh
obj-m += hello-world.o

KDIR ?= /lib/modules/$(shell uname -r)/build
PWD := $(CURDIR)

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```
3. Write `hello-world.c`:
```c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/printk.h>

static int __init bcf_greybus_init(void) {
  pr_info("Hello World 1.\n");
  return 0;
}

static void __exit bcf_greybus_exit(void) { pr_info("Goodbye World 1.\n"); }

module_init(bcf_greybus_init);
module_exit(bcf_greybus_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ayush Singh <ayushdevel1325@gmail.com>");
MODULE_DESCRIPTION("A simple Hello World Driver");
```
4. Build the Module:
```sh
make KDIR=${PATH_TO_BEAGLE_KERNEL} ARCH=arm64 CROSS_COMPILE=/usr/bin/aarch64-linux-gnu- -j12
```

# Run on BeaglePlay
1. Copy the module to BeaglePlay:
```sh
scp hello-world.ko debian@beagleplay.local:~/
```
2. Install Module in BeaglePlay:
```sh
sudo insmod hello-world.ko
```
3. Remove Module:
```sh
sudo rmmod hello_world
```
4. Check out the Kernel logs using `dmesg`:
```sh
dmesg -H
```

# Conclusion
The Kernel module I am writing for GSoC 23 can be found [here](https://git.beagleboard.org/ayush1325/bcf-greybus-driver). You can follow my GSoC23-related blog posts using this [feed](https://www.programmershideaway.xyz/tags/gsoc23/atom.xml).

Consider [supporting me](@/pages/supportme.md) if you like my work.
