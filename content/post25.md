+++
title = "GSoC23: Concurrency in ZephyrRTOS"
description = "An Introduction to Writing Concurrent Application in Zephyr"
date = "2023-07-03T00:30:12+05:30"

[taxonomies]
tags = ["c", "gsoc23", "zephyr"]
+++
Hello everyone. I am working on a CC1352 firmware for Zephyr. This will be responsible for SVC and AP Bridge role in the Greybus topology that Gbridge currently handles. Zephyr provides a lot of abstractions to write efficient concurrent code. I am going to discuss some of them in this post.

<!-- more -->
<br>

# Introduction
The CC1352 firmware must perform many tasks, such as reading and writing to the greybus node over IEEE802.15.4, reading and writing to Linux Host over UART HDLC, etc. This requires efficient concurrent code. 

<br>

# Workqueue
According to the documentation

> A workqueue is a kernel object that uses a dedicated thread to process work items in a first in, first out manner. Each work item is processed by calling the function specified by the work item. A workqueue is typically used by an ISR or a high-priority thread to offload non-urgent processing to a lower-priority thread so it does not impact time-sensitive processing.

While this description makes a work item seem dynamic, I have had some trouble using dynamic work items, namely, the ownership of the work container. Thus, I mostly use a work queue with some form of buffer (ring buffer or message queue).

The kernel defines a work queue, the system work queue, available to any application or kernel code requiring work queue support. Additional work queues should only be defined when submitting new work items to the system work queue is impossible since each new work queue incurs a significant cost in memory footprint. A new work queue can be justified if it is not possible for its work items to co-exist with existing system work queue work items without an unacceptable impact; for example, if the new work items perform blocking operations that would delay other system work queue processing to an unacceptable degree.

## Example
This a simple example using a system work queue.
```c
void my_isr(void *arg)
{
    ...
    if (error detected) {
        k_work_submit(&work);
    }
    ...
}

void print_error(struct k_work *item)
{
    printk("Got error on device\n");
}

K_WORK_DEFINE(work, print_error);
```

<br>

# Threads
According to the documentation

> A thread is a kernel object that is used for application processing that is too lengthy or too complex to be performed by an ISR.

Zephyr threads are simple to use if you have used system threads for any OS. The stack size needs to be defined in advance. It is also possible to send arguments in the form of void pointers.

## Example
Here is a simple example to demonstrate using threads.
```c
#define MY_STACK_SIZE 1024
#define MY_PRIORITY 5

static void my_entry_point(void *p1, void *p2, void *p3) {
    while(1) {
        printk("Hello World");
        k_sleep(K_MSEC(500));
    }
}

K_THREAD_DEFINE(my_tid, MY_STACK_SIZE,
                my_entry_point, NULL, NULL, NULL,
                MY_PRIORITY, 0, 0);
```

<br>

# Messge Queue
According to the documentation

> A message queue is a kernel object that implements a simple message queue, allowing threads and ISRs to asynchronously send and receive fixed-size data items.

Message queues are thread-safe. This makes them quite attractive for message passing between threads. The API is also simple to understand and use.

## Example
Here is a simple example using a message queue.
```c
K_MSGQ_DEFINE(msgq, sizeof(char), 10, 4);

void producer(void) {
    char c = 'a';
    k_msgq_put(&msgq, &c, K_FOREVER);
}

void main(void) {
    char tx;
    while(k_msgq_get(&msgq, &tx, K_FOREVER) == 0) {
        printk("Got %c", tx);
    }
}
```

<br>

# Ring Buffer
According to the documentation

> A ring buffer is a circular buffer, whose contents are stored in first-in-first-out order.

Ring buffers are great for storing variable-length data (such as a stream of bytes). However, it is essential to note that it is not inherently thread-safe. Due to this and the API, I prefer using message queues for fixed-length data.

## Example
Here is a simple example using a ring buffer.
Note: This example is not thread-safe.
```c
#define RING_BUF_BYTES 1024

RING_BUF_DECLARE(ringbuf, RING_BUF_BYTES);

void producer(void) {
    uint8_t *data;
    uint32_t len;

    len = ring_buf_put_claim(&ringbuf, &data, RING_BUF_BYTES);
    data[0] = 'a';
    ring_buf_put_finish(&ringbuf, 1);
}

void consumer(void) {
    uint8_t *data;
    uint32_t len;

    len = ring_buf_get_claim(&ringbuf, &data, RING_BUF_BYTES);
    for(size_t i = 0; i < len; ++i) {
        printk("%c", data[i]);
    }
    printk("\n");
    ring_buf_get_finish(&ringbuf, len);
}
```

<br>

# Mutex
According to the documentation

> A mutex is a kernel object that implements a traditional reentrant mutex. A mutex allows multiple threads to safely share an associated hardware or software resource by ensuring mutually exclusive access to the resource.

Mutex is required when sharing data between multiple threads. One must be careful when dealing with nested mutex access since it can lead to deadlocks.

## Example
```c
K_MUTEX_DEFINE(my_mutex);

k_mutex_lock(&my_mutex, K_FOREVER);
// Do some mutually exclusive
k_mutex_unlock(&my_mutex);
```

<br>

# Doubly Linked List
Zephyr has a linked list implementation similar to Linux Kernel. A doubly linked list allows removing node in constant time. Thus it is quite useful for out-of-order processing of data. I am using this for managing in-flight greybus operations.

It is also possible to access the container using `SYS_DLIST_CONTAINER` macro. 

## Example
Here is a simple example about doubly linked list.
```c
static sys_dlist_t my_dlist = SYS_DLIST_STATIC_INIT(&my_dlist);

struct my_data {
    sys_dlist_t node;
    int data;
};

void push(int data) {
    struct my_data *temp = k_malloc(sizeof(struct my_data));
    temp->data = data;
    sys_dnode_init(&temp);
    sys_dlist_append(&my_dlist, temp->node);
}

void pop(struct my_data *temp) {
    sys_dlist_remove(temp->node);
    k_free(temp);
}

void print_data(sys_dnode_t *node) {
    struct my_data *data;
    data = SYS_DLIST_CONTAINER(&my_dlist, data, node);
    printk("Data: %d", data->data);
}
```

<br>

# Conclusion
I hope this post helps in making multi-threading more approachable. While CC1352 is a single-core, multi-threading code can still make the code a lot more readable and performant. 

Consider [supporting me](@/pages/supportme.md) if you like my work.

<br>

# Helpful Links
- [Workqueue](https://docs.zephyrproject.org/latest/kernel/services/threads/workqueue.html)
- [Threads](https://docs.zephyrproject.org/latest/kernel/services/threads/index.html)
- [Message Queue](https://docs.zephyrproject.org/latest/kernel/services/data_passing/message_queues.html)
- [Ringbuf](https://docs.zephyrproject.org/latest/kernel/data_structures/ring_buffers.html)
- [Mutex](https://docs.zephyrproject.org/latest/kernel/services/synchronization/mutexes.html)
- [Doubly Linked List](https://docs.zephyrproject.org/latest/kernel/data_structures/dlist.html)
