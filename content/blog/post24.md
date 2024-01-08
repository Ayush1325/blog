+++
title = "GSoC23: Linux Serial Device Bus"
description = "An introduction to writing Linux Serial Device Bus Drivers"
date = "2023-06-15T16:05:12+05:30"

[taxonomies]
tags = ["c", "gsoc23", "linux"]
+++
Hello everyone. The Linux driver I am working on will be a Serial Device Bus Driver. So in this post, I will review the critical components of writing a functional Linux Serial Device Bus driver.

<!-- more -->
<br>

# Introduction
Here is the description of the Serial device Bus from the Mailing List:

> The serdev bus is designed for devices such as Bluetooth, WiFi, GPS and NFC connected to UARTs on host processors. Tradionally these have been handled with tty line disciplines, rfkill, and userspace glue such as hciattach. This approach has many drawbacks since it doesn't fit into the Linux driver model. Handling of sideband signals, power control and firmware loading are the main issues.

> This creates a serdev bus with controllers (i.e. host serial ports) and attached devices. Typically, these are point to point connections, but some devices have muxing protocols or a h/w mux is conceivable. Any muxing is not yet supported with the serdev bus.

The documentation about Serdev is sparse, so most of my knowledge comes from looking at other people's code.

<br>

# Device Tree
The "Open Firmware Device Tree", or simply Devicetree (DT), is a data structure and language for describing hardware. More specifically, it is a description of hardware that is readable by an operating system so that the operating system doesn't need to hard code details of the machine.

I will use a device tree to attach my driver to a specific UART (connecting AM62 and CC1352). I will be using the Device Tree overlay from [bcfserial](https://git.beagleboard.org/beagleplay/bcfserial):

```dts
/dts-v1/;
/plugin/;

/ {
		fragment@0 {
				target = <&uart4>;
				status = "okay";
				__overlay__ {
						bcfserial {
								compatible = "beagle,bcfserial";
								status = "okay";
						};
				};
		};
};
```

On the Driver source, we will limit the probing devices using the device table:
```c
static const struct of_device_id beagleplay_greybus_of_match[] = {
    {
        .compatible = "beagle,bcfserial",
    },
    {},
};
MODULE_DEVICE_TABLE(of, beagleplay_greybus_of_match);
```
<br>

# Greybus Driver
This struct defines our actual driver and stores the related data structures. It has the following members:
1. **serdev:** The serdev device we will use with this driver.
2. **tx_work:** The [workqueue](https://linux-kernel-labs.github.io/refs/heads/master/labs/deferred_work.html#workqueues) to execute writing to UART asynchronously.
3. **tx_producer_lock:** A spinlock for writing to the UART buffer.
4. **tx_consumer_lock:** A spinlock for reading from the UART buffer.
5. **tx_circ_buf:** A circular buffer that stores the UART data until it is written asynchronously.
```c
struct beagleplay_greybus {
  struct serdev_device *serdev;

  struct work_struct tx_work;
  spinlock_t tx_producer_lock;
  spinlock_t tx_consumer_lock;
  struct circ_buf tx_circ_buf;
};
```
<br>

# Device Driver
We first need to define the `serdev_device_driver` structure:
```c
static struct serdev_device_driver beagleplay_greybus_driver = {
    .probe = beagleplay_greybus_probe,
    .remove = beagleplay_greybus_remove,
    .driver =
        {
            .name = BEAGLEPLAY_GREYBUS_DRV_NAME,
            .of_match_table = of_match_ptr(beagleplay_greybus_of_match),
        },
};
```

It contains two functions:
1. `beagleplay_greybus_probe`
2. `beagleplay_greybus_remove`

## Probe
The probe function is called when a UART device is detected. In our case, it will only be called once since we have limited the UART device. It needs to initialize our `beagleplay_greybus` driver. This involves the following:
1. Allocate our driver struct.
2. Initialize the work queue for UART transmission.
3. Set up spin locks for producer and consumer.
4. Allocate a circular buffer for storing the data to write to UART.
5. Open serdev device.
```c
static int beagleplay_greybus_probe(struct serdev_device *serdev) {
  u32 speed = 115200;
  int ret = 0;

  struct beagleplay_greybus *beagleplay_greybus =
      devm_kmalloc(&serdev->dev, sizeof(struct beagleplay_greybus), GFP_KERNEL);

  beagleplay_greybus->serdev = serdev;

  INIT_WORK(&beagleplay_greybus->tx_work, beagleplay_greybus_uart_transmit);
  spin_lock_init(&beagleplay_greybus->tx_producer_lock);
  spin_lock_init(&beagleplay_greybus->tx_consumer_lock);
  beagleplay_greybus->tx_circ_buf.head = 0;
  beagleplay_greybus->tx_circ_buf.tail = 0;
  beagleplay_greybus->tx_circ_buf.buf =
      devm_kmalloc(&serdev->dev, TX_CIRC_BUF_SIZE, GFP_KERNEL);

  serdev_device_set_drvdata(serdev, beagleplay_greybus);
  serdev_device_set_client_ops(serdev, &beagleplay_greybus_ops);

  ret = serdev_device_open(serdev);
  if (ret) {
    dev_err(&beagleplay_greybus->serdev->dev, "Unable to Open Device");
    return ret;
  }

  speed = serdev_device_set_baudrate(serdev, speed);
  dev_dbg(&beagleplay_greybus->serdev->dev, "Using baudrate %u\n", speed);

  serdev_device_set_flow_control(serdev, false);

  dev_info(&beagleplay_greybus->serdev->dev, "Successful Probe\n");

  return 0;
}
```

## Remove
This function is called when the driver is unloaded, or the UART device is removed. We need to perform cleanup here:
1. Flush pending write work.
2. Close serdev device.
```c
static void beagleplay_greybus_remove(struct serdev_device *serdev) {
  struct beagleplay_greybus *beagleplay_greybus =
      serdev_device_get_drvdata(serdev);

  dev_info(&beagleplay_greybus->serdev->dev, "Remove Driver\n");

  flush_work(&beagleplay_greybus->tx_work);
  serdev_device_close(serdev);
}
```

## Writing to UART
The writing to UART part is performed asynchronously in the following steps:
1. The driver writes to `beagleplay_greybus->tx_circ_buf`.
2. The contents of `beagleplay_greybus->tx_circ_buf` are written to the UART using the work queue.
```c
static void
beagleplay_greybus_append(struct beagleplay_greybus *beagleplay_greybus,
                          u8 value) {
  // must be locked already
  int head = beagleplay_greybus->tx_circ_buf.head;

  while (true) {
    int tail = READ_ONCE(beagleplay_greybus->tx_circ_buf.tail);

    if (CIRC_SPACE(head, tail, TX_CIRC_BUF_SIZE) >= 1) {

      beagleplay_greybus->tx_circ_buf.buf[head] = value;

      smp_store_release(&(beagleplay_greybus->tx_circ_buf.head),
                        (head + 1) & (TX_CIRC_BUF_SIZE - 1));
      return;
    } else {
      dev_dbg(&beagleplay_greybus->serdev->dev, "Tx circ buf full\n");
      usleep_range(3000, 5000);
    }
  }
}

static void beagleplay_greybus_serdev_write_locked(
    struct beagleplay_greybus *beagleplay_greybus) {
  // must be locked already
  int head = smp_load_acquire(&beagleplay_greybus->tx_circ_buf.head);
  int tail = beagleplay_greybus->tx_circ_buf.tail;
  int count = CIRC_CNT_TO_END(head, tail, TX_CIRC_BUF_SIZE);
  int written;

  if (count >= 1) {
    written = serdev_device_write_buf(
        beagleplay_greybus->serdev, &beagleplay_greybus->tx_circ_buf.buf[tail],
        count);
    dev_info(&beagleplay_greybus->serdev->dev, "Written Data of Len: %u\n",
             written);

    smp_store_release(&(beagleplay_greybus->tx_circ_buf.tail),
                      (tail + written) & (TX_CIRC_BUF_SIZE - 1));
  }
}
```

## Workque callback
This work queue callback is called by the Kernel asynchronously. It then writes the contents of `beagleplay_greybus->tx_circ_buf` to UART.
```c
static void beagleplay_greybus_uart_transmit(struct work_struct *work) {
  struct beagleplay_greybus *beagleplay_greybus =
      container_of(work, struct beagleplay_greybus, tx_work);

  spin_lock_bh(&beagleplay_greybus->tx_consumer_lock);
  dev_info(&beagleplay_greybus->serdev->dev, "Write to tx buffer");
  beagleplay_greybus_serdev_write_locked(beagleplay_greybus);
  spin_unlock_bh(&beagleplay_greybus->tx_consumer_lock);
}
```
<br>

# Device Operations
The serdev device operations define the functions that handle asynchronous reading and writing to UART.
```c
static struct serdev_device_ops beagleplay_greybus_ops = {
    .receive_buf = beagleplay_greybus_tty_receive,
    .write_wakeup = beagleplay_greybus_tty_wakeup,
};
```

It has two main functions:
1. `beagleplay_greybus_tty_receive`
2. `beagleplay_greybus_tty_wakeup`

## TTY Recieve
This function is called when we receive data over UART. For now, I am just printing the data Kernel Logs.
```c
static int beagleplay_greybus_tty_receive(struct serdev_device *serdev,
                                          const unsigned char *data,
                                          size_t count) {
  struct beagleplay_greybus *beagleplay_greybus;

  beagleplay_greybus = serdev_device_get_drvdata(serdev);
  dev_info(&beagleplay_greybus->serdev->dev, "tty recieve\n");
  dev_info(&beagleplay_greybus->serdev->dev, "Data: %s\n", data);

  return count;
}
```

## TTY Wakeup
We call `schedule_work` when tty Wakeup is triggered by the Kernel. This adds the job to Kernel global work queue if it was not already queued and leaves it in the same position on the kernel-global work queue otherwise.
```c
static void beagleplay_greybus_tty_wakeup(struct serdev_device *serdev) {
  struct beagleplay_greybus *beagleplay_greybus;

  beagleplay_greybus = serdev_device_get_drvdata(serdev);
  dev_info(&beagleplay_greybus->serdev->dev, "tty wakeup\n");

  schedule_work(&beagleplay_greybus->tx_work);
}
```
<br>

# Send HelloWorld over UART
Here is a simple function to write "HelloWorld" over Serial from our new driver:
```c
static void hello_world(struct beagleplay_greybus *beagleplay_greybus) {
  const char msg[] = "HelloWorld\n\0";
  const ssize_t msg_len = strlen(msg);
  ssize_t i;

  spin_lock(&beagleplay_greybus->tx_producer_lock);
  for (i = 0; i < msg_len; ++i) {
    beagleplay_greybus_append(beagleplay_greybus, msg[i]);
  }
  spin_unlock(&beagleplay_greybus->tx_producer_lock);

  spin_lock(&beagleplay_greybus->tx_consumer_lock);
  beagleplay_greybus_serdev_write_locked(beagleplay_greybus);
  spin_unlock(&beagleplay_greybus->tx_consumer_lock);

  dev_info(&beagleplay_greybus->serdev->dev, "Written Hello World");
};
```
Now we can call this function from `beagleplay_greybus_probe` after driver initialization is complete.

**NOTE:** This function writes to UART synchronously. Generally, this should be avoided.

<br>

# Conclusion
[Here](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver/-/tree/develop) is the current working repository for my Linux Driver. Feel free to check out the code and open a PR if you are interested.

Consider [supporting me](@/pages/about.md) if you like my work.

<br>

# Helpful Links
- [Workqueue](https://linux-kernel-labs.github.io/refs/heads/master/labs/deferred_work.html#workqueues)
- [Serdev Driver](http://events17.linuxfoundation.org/sites/events/files/slides/serdev-elce-2017-2.pdf)
- [Device Tree](https://docs.kernel.org/devicetree/usage-model.html)
- [Spinlock](https://docs.kernel.org/locking/spinlocks.html#lesson-1-spin-locks)
