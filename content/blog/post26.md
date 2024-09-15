+++
title = "GSoC23: Project Status update"
description = "A blog post to summarise everything that is currently working"
date = "2023-07-08T00:30:12+05:30"

[taxonomies]
tags = ["c", "gsoc23", "zephyr", "linux", "beagleboard"]
+++
Hello everyone. It will soon be time for the Mid-term evaluation of Google Summer of Code 2023. Thus, I decided to write a post to summarise everything I have been working on. I will also go over how it can be replicated by anyone interested.

<!-- more -->

# Introduction
This project has two main parts
- BeaglePlay CC1352 Application
- BeaglePlay Linux Driver

CC1352 and Linux driver communicate over UART using High-Level Data Link Protocol (HDLC). The BeaglePlay CC1352 application also communicates with the BeagleConnect node running [greybus-for-zephyr](https://git.beagleboard.org/ayush1325/greybus-for-zephyr/-/tree/latest-zephyr). I have left greybus-for-zephyr largely unmodified. The changes in my fork were to make it run with the latest Zephyr.

I will now review the current functionality with the working logs, which should be reproducible.

# BeaglePlay Linux Driver
## Description
The [beagleplay Linux driver](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver/-/tree/develop) is responsible for HDLC communication over UART. I am using three hdlc addresses:
1. DBG: For logs from BeaglePlay CC1352. These frames are only ever received.
2. Greybus: For greybus payload. All greybus related communication should use this. Currently does not do much.
3. MCUmgr: For MCUmgr tty communication. These frames are sent when data is written to `ttyMCU0`. Any MCUmgr frame sent from CC1352 is also streamed to the `ttyMCU0`.

The Linux driver complies against [v5.10.168-ti-arm64-r103](https://git.beagleboard.org/beagleboard/linux/-/tree/v5.10.168-ti-arm64-r103), which is running on BeaglePlay.

## Implementation
### UART Transmission
The UART transmission is done using a workqueue. Producers write the data (HDLC block) in a circular buffer. The workqueue handler then reads the circular buffer to send the block over HDLC.

### UART Receiving
The UART receiving is done using serdev client ops. The data is buffered until a complete HDLC frame is received. Once the frame is complete, it is handled appropriately depending on the frame address field. Finally, the receive buffer is cleared to prepare for a new frame.

### MCUmgr Sending
Any data written to `ttyMCU0` is buffered until `\x0a` is encountered. Then the data is queued to be sent as a single MCUmgr frame. This part works somewhat since my Zephyr application successfully processes the MCUmgr fragment. However, the MCUmgr side of things is still a work in progress.

### MCUmgr Receiving
If an MCUmgr HDLC frame is received, the data is directly streamed to `ttyMCU0`. This is untested since I have not yet received a response from BeaglePlay CC1352 MCUmgr.

# BeaglePlay CC1352 Application
The BeaglePlay CC1352 Application currently has the following responsibilities:
1. Handle HDLC UART Communication.
2. Discover Any new nodes in the network.
3. Maintain a table of all active nodes and their cports.
4. Maintains a list of all in-flight greybus operations.
5. Handle Greybus communication with Nodes over TCP sockets.
6. Logging over HDLC UART with a custom backend.
7. Cleanup finished greybus operations along will calling associated callbacks

## Implementation
### HDLC Uart Communication
An interrupt is used to handle receiving data over UART. This data is then written to a ring buffer. The actual processing of this data is done in the system workqueue. The workqueue buffers the input data until a complete HDLC frame is received. Then the frame is processed depending on its address.

The writing of a block is also done asynchronously using the System workqueue. A [First In First Out Queue](https://docs.zephyrproject.org/latest/kernel/services/data_passing/fifos.html) of blocks to transmit is maintained. The workqueue handler sends all pending blocks when it is called.

### Node Discovery
A thread is constantly running that performs node discovery after a set interval (which is configurable). The Node IPv6 address is currently static since Zephyr DNS Resolver does not support DNS Discovery. However, it will be easy to add dynamic node discovery by modifying [get_all_nodes function](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware/-/blob/develop/src/main.c#L133).

After querying for all greybus nodes, it checks against the active nodes table if the node is already present. In case the node is absent, it is added to the table, and the node is submitted for setup.

Node setup runs on the System Workqueue. Its job is to create a TCP socket with the specified cport (in this case, Cport0). It then adds the cport socket to the nodes table. Finally, in the case of CPort0, the GetManifestSize control request is sent to the node.

If the `GetManifestSize` response is successfully received, a GetManifest control request is sent using `gb_operations->callback`. The response of this request is parsed to get all available Cports in the Node. All the cports other than Cport0 are then queued for setup.

The setup is similar to Cport0. However, in the case of these Cports, a simple Ping SVC request is sent.

Everything is logged to the Linux host using the custom HDLC logging backend.

### Nodes Table
The nodes table is maintained as a static array of node_table_items:
```c
struct node_table_item {
  struct in6_addr addr;
  int cport0;
  size_t num_cports; /* Number of cports. This does not include Cport0 */
  int *cports;
};
```
The size of the node_table array is configurable. The Cport0 or Control port is treated differently since a node without CPort0 is mostly cleaned up. A new node can be added with just the IPv6 address. However, it is essential to initialize the Cport0 (which is done on node discovery).

The Cports pointer is supposed to be dynamically initialized after parsing the greybus manifest. The current assumption is that cports will be initialized only once. Thus it does not implement copying to resized cports array yet.

The Cports can be removed by passing the socket. This also closes the socket. If CPort0 is removed, the whole node is also removed with all sockets closed.

It is also possible to remove a node by IPv6 address. It also closes any open sockets.

### Greybus Operations
A [Double-linked list](https://docs.zephyrproject.org/latest/kernel/data_structures/dlist.html) of in-flight greybus operations is maintained. This is because the greybus operations can be sent/received out of order, making a regular queue unfit. The `gb_operaitions` struct is as follows:
```c
/*
 * Struct to represent a greybus operation.
 *
 * @param sock: socket to perform this operation on.
 * @param operation_id: the unique id for this operation.
 * @param request_sent: flag to check if the request has been sent.
 * @param request: pointer to greybus request message.
 * @param response: pointer to greybus response message.
 * @param callback: callback function called when operation is completed.
 * @param node: operation dlist node.
 */
struct gb_operation {
  int sock;
  uint16_t operation_id;
  bool request_sent;
  bool response_received;
  struct gb_message *request;
  struct gb_message *response;
  greybus_operation_callback_t callback;
  sys_dnode_t node;
};
```
The sock parameter will probably be removed soon since I must also send requests to AP (Linux Driver) over UART.

The operation id is set using an atomic if the operation is not one-shot. For one-shot operations, the operation id is 0. 

The request and response are not allocated when allocating the operation. Similarly, the operation is not queued unless `gb_operation_queue` is called. This is important since the cleanup is the caller's responsibility until the operation is queued.

An optional callback can be provided, which is called in the System Workqueue. For one-shot requests, this is called once the request is sent, while for normal operations, it is called on receiving a response. The callback can essentially serve as a way to chain dependent greybus operations, or do something with the response, etc. Once the operation concludes, it is removed from the in-flight operations dlist and moved to the callback processing dlist. There is no reason to use a dlist for callbacks since they are called in a FIFO order, but the `sys_dnode_t` is already present, so you might as well use it. Once the callback is complete, the operation is deallocated.

The `gb_message` structure essentially stores a greybus message with a header and payload. It also contains a pointer to the `gb_operation`, if there is any.
```c
/*
 * Struct to represent greybus message
 *
 * @param operation: greybus operation this message is associated with. Can be
 * NULL in case of message received.
 * @param header: greybus msg header.
 * @param payload: heap allocated payload.
 * @param payload_size: size of payload in bytes
 */
struct gb_message {
  struct gb_operation *operation;
  struct gb_operation_msg_hdr header;
  void *payload;
  size_t payload_size;
};
```

### Communication with nodes
The communication with greybus nodes happens over TCP. Each Cport in the Node exposes a Port in incrementing order. We establish a connection to these CPorts from the node setup.

A Reader thread is continuously running, which checks for any sockets with pending data using `k_poll`. If a valid greybus message is received, we first check if is associated with any in-flight greybus operations. If it is valid, we add the response to the operation, which then processes the callback. In the case of stand-alone messages, we do nothing since it is probably intended for the AP.

A Writer thread is also continuously running, checking if any socket is available for writing using `k_poll`. If the socket can be written to, we check if any greybus operations are present for that socket. In case of a match, we send the message and mark the operation request as sent.

### Logging over HDLC
I also wrote a custom logging backend that sends all logging data as an HDCL frame with a DBG address. This data is then printed to standard Linux logs. Currently, the logging backend does not do much message processing, like setting the log level, but it can be done in the future.

# Demo
Here are all the components for this demo:
1. [greybus-for-zephyr](https://git.beagleboard.org/ayush1325/greybus-for-zephyr/-/tree/latest-zephyr): 57cf1c1b1ee3388d1ad9971ed77f548ae7abf63a
2. [Zephyr](https://git.beagleboard.org/beagleconnect/zephyr/zephyr/-/tree/sdk-next): 520fb22555402360e5eba798f6834771254198af
3. [beagleplay-greybus-driver](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver/-/tree/develop): 40ed0fbc6cbe3c150d7eec74331f53a5e4fc351b
4. [cc1352-firmware](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware/-/tree/develop): d68e300440affc502209b7fa8f39e57bc0476346

The instructions for building beagleplay-greybus-driver can be found in my [linux driver post](@/blog/post22.md). The instructions for building cc1352-firmware are similar to my [zephyr application post](@/blog/post23.md). It is more tricky to compile greybus-for-zephyr, but my fork (with some changes to the project config) should work.

For flashing, I am still using [cc1352-flasher](https://git.beagleboard.org/beagleconnect/cc1352-flasher/-/tree/from-20230521) as shown in my [previous post](@/blog/post23.md).

After installing the Linux driver, I used a simple Python script to reset cc1352 to get the full logs:
```python
import time
import gpiod

CC1352_RESET_GPIO=14

if __name__ == "__main__":
    gpiochip = gpiod.Chip("gpiochip2", gpiod.Chip.OPEN_BY_NAME)
    cc1352_reset_line = gpiochip.get_line(CC1352_RESET_GPIO)
    cc1352_reset_line.request(consumer="reset", type=gpiod.LINE_REQ_DIR_OUT)
    cc1352_reset_line.set_value(0)
    time.sleep(0.2)
    print('Setting RESET high')
    cc1352_reset_line.set_value(1)
    cc1352_reset_line.set_direction_input()
    cc1352_reset_line.release()
```

## BeagleConnect Node Logs
```sh
*** Booting Zephyr OS build bcf-sdk-0.2.1-3359-ga3641861aff2 ***
[00:00:00.009,735] <dbg> greybus_service: greybus_service_init: Greybus initializing..
[00:00:00.009,765] <dbg> greybus_manifest: identify_descriptor: cport_id = 0
[00:00:00.009,826] <dbg> greybus_manifest: identify_descriptor: cport_id = 1
[00:00:00.009,857] <dbg> greybus_manifest: identify_descriptor: cport_id = 2
[00:00:00.009,887] <dbg> greybus_transport_tcpip: gb_transport_backend_init: Greybus TCP/IP Transport initializing..
[00:00:00.010,131] <inf> greybus_transport_tcpip: CPort 0 mapped to TCP/IP port 4242
[00:00:00.014,709] <inf> greybus_transport_tcpip: CPort 1 mapped to TCP/IP port 4243
[00:00:00.014,953] <inf> greybus_transport_tcpip: CPort 2 mapped to TCP/IP port 4244
[00:00:00.015,075] <inf> greybus_transport_tcpip: Greybus TCP/IP Transport initialized
[00:00:00.015,136] <inf> greybus_manifest: Registering CONTROL greybus driver.
[00:00:00.015,167] <dbg> greybus: _gb_register_driver: Registering Greybus driver on CP0
[00:00:00.015,380] <inf> greybus_manifest: Registering GPIO greybus driver.
[00:00:00.015,411] <dbg> greybus: _gb_register_driver: Registering Greybus driver on CP1
[00:00:00.015,594] <inf> greybus_manifest: Registering I2C greybus driver.
[00:00:00.015,625] <dbg> greybus: _gb_register_driver: Registering Greybus driver on CP2
[00:00:00.015,747] <inf> greybus_service: Greybus is active
[00:00:08.432,830] <dbg> greybus_transport_tcpip: accept_new_connection: cport 0 accepted connection from [2001:db8::2]:37546 as fd 3
[00:00:08.716,644] <dbg> greybus: gb_process_request: gb_control_get_manifest_size: 0
[00:00:08.716,796] <wrn> cbprintf_package: (unsigned) char * used for %p argument. It's recommended to cast it to void * because it may cause misbehavior in certain configurations. String:"%s: gb: send(%d, %p, %zu, 0)" argument:2
[00:00:08.716,827] <dbg> greybus_transport_tcpip: sendMessage: gb: send(3, 0x20013678, 10, 0)
[00:00:09.232,543] <dbg> greybus: gb_process_request: gb_control_get_manifest: 0
[00:00:09.232,696] <wrn> cbprintf_package: (unsigned) char * used for %p argument. It's recommended to cast it to void * because it may cause misbehavior in certain configurations. String:"%s: gb: send(%d, %p, %zu, 0)" argument:2
[00:00:09.232,727] <dbg> greybus_transport_tcpip: sendMessage: gb: send(3, 0x20013700, 200, 0)
[00:00:22.429,962] <dbg> greybus_transport_tcpip: accept_new_connection: cport 1 accepted connection from [2001:db8::2]:46805 as fd 4
[00:00:22.445,892] <dbg> greybus_transport_tcpip: accept_new_connection: cport 2 accepted connection from [2001:db8::2]:35818 as fd 5
[00:00:22.738,494] <dbg> greybus: gb_process_request: gb_gpio_protocol_version: 0
[00:00:22.738,677] <wrn> cbprintf_package: (unsigned) char * used for %p argument. It's recommended to cast it to void * because it may cause misbehavior in certain configurations. String:"%s: gb: send(%d, %p, %zu, 0)" argument:2
[00:00:22.738,708] <dbg> greybus_transport_tcpip: sendMessage: gb: send(4, 0x200136a8, 10, 0)
[00:00:22.996,429] <dbg> greybus: gb_process_request: gb_i2c_protocol_version: 0
[00:00:22.996,612] <wrn> cbprintf_package: (unsigned) char * used for %p argument. It's recommended to cast it to void * because it may cause misbehavior in certain configurations. String:"%s: gb: send(%d, %p, %zu, 0)" argument:2
[00:00:22.996,643] <dbg> greybus_transport_tcpip: sendMessage: gb: send(5, 0x200136a8, 10, 0)
```

## BeaglePlay Linux Logs
```sh
...skipping...
[ +19.288401] bcfgreybus serial1-0: DBG Frame: *** Booting Zephyr OS build bcf-sdk-0.2.1-3359-ga3641861aff2 ***
[  +0.013175] bcfgreybus serial1-0: DBG Frame: [00:00:00.019,805] <inf> net_config: Initializing network
[  +0.013263] bcfgreybus serial1-0: DBG Frame: [00:00:00.019,927] <inf> net_config: IPv4 address: 192.0.2.2
[  +0.013646] bcfgreybus serial1-0: DBG Frame: [00:00:00.127,868] <inf> net_config: IPv6 address: 2001:db8::2
[  +0.014292] bcfgreybus serial1-0: DBG Frame: [00:00:00.129,119] <inf> cc1352_greybus: Starting BeaglePlay Greybus
[  +0.019938] bcfgreybus serial1-0: DBG Frame: [00:00:00.129,547] <dbg> net_sock: zsock_socket_internal: (0x20002638): socket: ctx=0x20002ab0, fd=0
[  +0.016514] bcfgreybus serial1-0: DBG Frame: [00:00:00.129,577] <inf> cc1352_greybus: Trying to connect to node from socket 0
[  +0.017943] bcfgreybus serial1-0: DBG Frame: [00:00:00.130,706] <inf> cc1352_greybus: Added Greybus Node
[  +0.013700] bcfgreybus serial1-0: DBG Frame: [00:00:00.360,137] <inf> cc1352_greybus: Connected to Greybus Node
[  +0.014855] bcfgreybus serial1-0: DBG Frame: [00:00:00.360,168] <dbg> cc1352_greybus: node_setup: Added Cport 0
[  +0.019548] bcfgreybus serial1-0: DBG Frame: [00:00:00.360,260] <dbg> cc1352_greybus: node_setup: Sent get manifest size request
[  +0.951923] bcfgreybus serial1-0: DBG Frame: [00:00:00.636,199] <dbg> cc1352_greybus: gb_operation_send_request_all: Request 1 sent
[  +0.018590] bcfgreybus serial1-0: DBG Frame: [00:00:00.651,702] <dbg> net_sock: zsock_received_cb: (0x20001ee8): ctx=0x20002ab0, pkt=0x2000ed88, st=0, user_data=(nil)
[  +0.025380] bcfgreybus serial1-0: DBG Frame: [00:00:00.651,916] <dbg> cc1352_greybus: gb_operation_set_response: Operation with ID 1 completed
[  +0.020478] bcfgreybus serial1-0: DBG Frame: [00:00:00.651,977] <dbg> cc1352_greybus: gb_control_get_manifest_size_callback: Manifest Size: 192 bytes
[  +0.021837] bcfgreybus serial1-0: DBG Frame: [00:00:00.652,038] <dbg> cc1352_greybus: gb_control_get_manifest_size_callback: Sent control get manifest request
[  +0.023736] bcfgreybus serial1-0: DBG Frame: [00:00:01.141,662] <dbg> cc1352_greybus: gb_operation_send_request_all: Request 2 sent
[  +0.018560] bcfgreybus serial1-0: DBG Frame: [00:00:01.172,302] <dbg> net_sock: zsock_received_cb: (0x20001ee8): ctx=0x20002ab0, pkt=0x2000ed40, st=0, user_data=(nil)
[  +0.025278] bcfgreybus serial1-0: DBG Frame: [00:00:01.183,258] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018329] bcfgreybus serial1-0: DBG Frame: [00:00:01.186,645] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018254] bcfgreybus serial1-0: DBG Frame: [00:00:01.464,111] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018234] bcfgreybus serial1-0: DBG Frame: [00:00:01.473,876] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +1.207098] bcfgreybus serial1-0: DBG Frame: [00:00:01.879,943] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018212] bcfgreybus serial1-0: DBG Frame: [00:00:01.883,300] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018252] bcfgreybus serial1-0: DBG Frame: [00:00:02.498,687] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018308] bcfgreybus serial1-0: DBG Frame: [00:00:02.502,044] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +1.486659] bcfgreybus serial1-0: DBG Frame: [00:00:03.421,478] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018322] bcfgreybus serial1-0: DBG Frame: [00:00:03.424,865] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +1.360565] bcfgreybus serial1-0: DBG Frame: [00:00:04.800,262] <err> net_ieee802154_6lo_fragment: Could not get a cache entry 
[  +0.018317] bcfgreybus serial1-0: DBG Frame: [00:00:04.803,619] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +2.067861] bcfgreybus serial1-0: DBG Frame: [00:00:06.886,322] <dbg> net_sock: zsock_received_cb: (0x20001ee8): ctx=0x20002ab0, pkt=0x2000ed40, st=0, user_data=(nil)
[  +0.025061] bcfgreybus serial1-0: DBG Frame: [00:00:07.168,426] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018434] bcfgreybus serial1-0: DBG Frame: [00:00:07.171,783] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018327] bcfgreybus serial1-0: DBG Frame: [00:00:07.584,228] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018134] bcfgreybus serial1-0: DBG Frame: [00:00:07.587,615] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +1.236824] bcfgreybus serial1-0: DBG Frame: [00:00:08.203,033] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018201] bcfgreybus serial1-0: DBG Frame: [00:00:08.206,390] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018145] bcfgreybus serial1-0: DBG Frame: [00:00:09.125,823] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018315] bcfgreybus serial1-0: DBG Frame: [00:00:09.129,211] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +2.246804] bcfgreybus serial1-0: DBG Frame: [00:00:10.504,608] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +0.018327] bcfgreybus serial1-0: DBG Frame: [00:00:10.507,995] <err> net_ieee802154_6lo_fragment: Could not get a cache entry
[  +1.109356] bcfgreybus serial1-0: DBG Frame: [00:00:12.590,698] <dbg> net_sock: zsock_received_cb: (0x20001ee8): ctx=0x20002ab0, pkt=0x2000ed40, st=0, user_data=(nil)
[  +0.011856] bcfgreybus serial1-0: DBG Frame: [00:00:12.600,769] <dbg> net_sock: zsock_received_cb: (0x20001ee8): ctx=0x20002ab0, pkt=0x2000ed40, st=0, user_data=(nil)
[  +0.011169] bcfgreybus serial1-0: DBG Frame: [00:00:12.600,891] <dbg> cc1352_greybus: gb_operation_set_response: Operation with 2 completed
[  +0.024784] bcfgreybus serial1-0: DBG Frame: [00:00:12.600,952] <dbg> cc1352_greybus: gb_manifest_get_cports: Manifest Size: 192
[  +0.018105] bcfgreybus serial1-0: DBG Frame: [00:00:12.600,982] <dbg> cc1352_greybus: gb_manifest_get_cports: Manifest version: 0.1
[  +0.018678] bcfgreybus serial1-0: DBG Frame: [00:00:12.601,074] <dbg> cc1352_greybus: gb_control_get_manifest_callback: CPort: ID 0, Protocol: 0, Bundle: 0
[  +0.023162] bcfgreybus serial1-0: DBG Frame: [00:00:12.601,074] <dbg> cc1352_greybus: gb_control_get_manifest_callback: CPort: ID 1, Protocol: 2, Bundle: 1
[  +0.023106] bcfgreybus serial1-0: DBG Frame: [00:00:12.601,135] <dbg> cc1352_greybus: gb_control_get_manifest_callback: CPort: ID 2, Protocol: 3, Bundle: 1
[  +0.023225] bcfgreybus serial1-0: DBG Frame: [00:00:12.601,562] <dbg> net_sock: zsock_socket_internal: (0x20002638): socket: ctx=0x20002b58, fd=1
[  +0.016492] bcfgreybus serial1-0: DBG Frame: [00:00:12.601,593] <inf> cc1352_greybus: Trying to connect to node from socket 1
[  +0.017856] bcfgreybus serial1-0: DBG Frame: [00:00:12.616,973] <inf> cc1352_greybus: Connected to Greybus Node
[  +0.014851] bcfgreybus serial1-0: DBG Frame: [00:00:12.617,004] <dbg> cc1352_greybus: node_setup: Added Cport 1
[  +0.014935] bcfgreybus serial1-0: DBG Frame: [00:00:12.617,065] <dbg> cc1352_greybus: node_setup: Sent ping request
[  +0.019069] bcfgreybus serial1-0: DBG Frame: [00:00:12.617,431] <dbg> net_sock: zsock_socket_internal: (0x20002638): socket: ctx=0x20002c00, fd=2
[  +0.019547] bcfgreybus serial1-0: DBG Frame: [00:00:12.617,431] <inf> cc1352_greybus: Trying to connect to node from socket 2
[  +0.017108] bcfgreybus serial1-0: DBG Frame: [00:00:12.632,873] <inf> cc1352_greybus: Connected to Greybus Node
[  +0.017678] bcfgreybus serial1-0: DBG Frame: [00:00:12.632,904] <dbg> cc1352_greybus: node_setup: Added Cport 2
[  +0.014957] bcfgreybus serial1-0: DBG Frame: [00:00:12.632,995] <dbg> cc1352_greybus: node_setup: Sent ping request
[  +0.020266] bcfgreybus serial1-0: DBG Frame: [00:00:12.669,097] <dbg> cc1352_greybus: gb_operation_send_request_all: Request 3 sent
[  +0.018670] bcfgreybus serial1-0: DBG Frame: [00:00:12.674,255] <dbg> cc1352_greybus: gb_operation_send_request_all: Request 4 sent
[  +0.018679] bcfgreybus serial1-0: DBG Frame: [00:00:12.925,140] <dbg> net_sock: zsock_received_cb: (0x20001ee8): ctx=0x20002b58, pkt=0x2000ecb0, st=0, user_data=(nil)
[  +1.106579] bcfgreybus serial1-0: DBG Frame: [00:00:13.102,874] <dbg> cc1352_greybus: gb_operation_set_response: Operation with 3 completed
[  +0.016056] bcfgreybus serial1-0: DBG Frame: [00:00:13.102,935] <dbg> cc1352_greybus: svc_ping_callback: Received Pong
[  +0.020780] bcfgreybus serial1-0: DBG Frame: [00:00:13.173,858] <dbg> net_sock: zsock_received_cb: (0x20001ee8): ctx=0x20002c00, pkt=0x2000ecb0, st=0, user_data=(nil)
[  +0.025525] bcfgreybus serial1-0: DBG Frame: [00:00:13.603,363] <dbg> cc1352_greybus: gb_operation_set_response: Operation with 4 completed
[  +0.016142] bcfgreybus serial1-0: DBG Frame: [00:00:13.603,424] <dbg> cc1352_greybus: svc_ping_callback: Received Pong
```

# Conclusion
I hope this sheds light on the current status of my project. You can follow my GSoC23-related blog posts using this [feed](https://www.programmershideaway.xyz/tags/gsoc23/atom.xml).

Consider [supporting me](@/pages/about.md) if you like my work.

# Build Artifacts
Here are the CI build artifacts for anyone wanting to test this out:
1. [cc1352-firmware](https://git.beagleboard.org/gsoc/greybus/cc1352-firmware/-/jobs/12140/artifacts/download?file_type=archive)
2. [beagleplay-greybus-driver](https://git.beagleboard.org/gsoc/greybus/beagleplay-greybus-driver/-/jobs/12129/artifacts/download?file_type=archive)
