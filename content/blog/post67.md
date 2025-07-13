+++
title = "Running Linux Kernel for Development on PocketBeagle 2"
description = "A tutorial on how to run custom kernels for development on PocketBeagle 2"
date = "2025-07-13T00:30:12+05:30"

[taxonomies]
tags = ["weekly-update", "beagleboard", "linux"]
+++

Hello everyone. In this post, I will go over the setup I use when doing Linux Kernel development. I will be using [PocketBeagle 2](https://www.beagleboard.org/boards/pocketbeagle-2) in this post, but the same instructions should work with most other BeagleBoard.org Boards.

# Setup

The first step is to fetch the kernel repo. I am going to use linux-next in this post, but the instructions stay the same for other kernel forks.

1. Clone the repo

```sh
git clone https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
cd linux-next
```

2. Since we are cross-compiling here, we need to set some environment variables. I personally use [direnv](https://direnv.net/), so I set them in the `.envrc` file:

```sh
export ARCH=arm64
# The CROSS_COMPILE binary might be different on Ubuntu based systems. I am using Fedora.
export CROSS_COMPILE=/usr/bin/aarch64-linux-gnu-
```

3. Create default config.

```sh
make -j$(nproc) defconfig
```

4. (Optional) Use menu config to enable more config options.

```sh
make -j$(nproc) menuconfig
```

# Building

Build the kernel.

```sh
make -j$(nproc)
```

# Copying to SD Card

The partitions on SD Card are mounted at `/run/media/ayush/`.

1. Copy Kernel

```sh
cp arch/arm64/boot/Image /run/media/ayush/BOOT/ImageDev
```

2. Install kernel modules into SD Card.

```sh
sudo make -j$(nproc) modules_install INSTALL_MOD_PATH=/run/media/ayush/rootfs
```

3. Add entry to extlinux.conf

```sh
cat <<'EOF' >> /run/media/ayush/BOOT/extlinux/extlinux.conf
label kernDev
    kernel /ImageDev
    append console=ttyS2,115200n8 earlycon=ns16550a,mmio32,0x02860000 root=/dev/mmcblk1p3 ro rootfstype=ext4 rootwait net.ifnames=0 modprobe.blacklist=pocktbeagle_connector
    fdtdir /
    # fdt /ti/k3-am6232-pocketbeagle2-dev.dtb
EOF
```

4. (Optional) It is possible to use a custom devicetree along with the kernel. Just copy the devicetree to `/run/media/ayush/BOOT/ti/k3-am6232-pocketbeagle2-dev.dtb` and uncomment the fdt line in the above entry.

Eject the SD Card.

# Booting into the new kernel

U-Boot will now show a menu to select the `kernDev` boot entry during boot. It is also possible to set the `kernDev` entry be the default boot entry in extlinux.conf

Once boot up is successful, the running kernel can be checked using `uname -r`.

# Ending Thoughts

That is all for this post. Hopefully, this helps people get an idea regarding running custom kernels without replacing the default one. There are also ways like network booting, which can be specially useful for development, but I will leave those for some other time.

Consider [supporting me](@/pages/about.md) if you like my work.

# Helpful links

- [BeagleBoard](https://www.beagleboard.org/)
