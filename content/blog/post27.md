+++
title = "GSoC23: BeaglePlay Greybus Demo"
description = "A greybus demo with my driver and zephyr application"
date = "2023-07-26T23:40:12+05:30"

[taxonomies]
tags = ["c", "gsoc23", "zephyr", "linux"]
+++
Hello everyone. My project is now at a point where almost everything is working to a respectable level. Thus I wanted to do a demo. Check out my [GSoC 2023 Introduction post](@/blog/post21.md) for more information about my project.

<!-- more -->

# Setup
## BeaglePlay Linux Host
The BeaglePlay Linux host will be running [beagleplay-linux-driver](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver/-/tree/develop). The Linux Host serves as AP and communicates with SVC (CC1352) over HDLC UART.

I will also be running the following script on the Linux host. It repeatedly calls `iio_info`, which displays IIO devices and their attributes. The humidity and optical sensors in BeagleConnect Freedom show up here once the greybus connection is successfully established.
```sh
#!/bin/sh
while true
do
    echo "Probe IIO"
    echo
    iio_info
    sleep 1
    echo
done
```

## BeaglePlay CC1352
The BeaglePlay CC1352 will be running [cc1352-firmware](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware/-/tree/develop). This zephyr application performs the SVC and APBridge roles in greybus topology.

## BeagleConnect Freedom
The BeagleConnect Freedom will run [greybus-for-zephyr](https://git.beagleboard.org/ayush1325/greybus-for-zephyr/-/tree/final) sample application. This is the same as the one used in the GBridge setup.

# Video
Here is a video of the demo running on my hardware.
{{ youtube(id="O5coD55JvGU") }}

# Limitations
The current setup has three main limitations:
1. Updating CC1352 firmware requires restart. This makes the development process slower.
2. Beagleconnect greybus node disconnects if it is ideal for too long.
3. BeaglePlay CC1352 does not perform actual service discovery.

# Conclusion
I will now work on making a single device "just" work. This will involve optimization and ironing out bugs. Once that is done, I will move on to tackle the limitations.

Consider [supporting me](@/pages/about.md) if you like my work.

# Build Artifacts
Here are the CI build artifacts for anyone wanting to test this out:
1. [cc1352-firmware](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware/-/jobs/12674/artifacts/download?file_type=archive)
2. [beagleplay-greybus-driver](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver/-/jobs/12506/artifacts/download?file_type=archive)

# Helpful Links
- [beagleplay-linux-driver](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver/-/tree/develop)
- [cc1352-firmware](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware/-/tree/develop)
- [greybus-for-zephyr](https://git.beagleboard.org/ayush1325/greybus-for-zephyr/-/tree/final)
