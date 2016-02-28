---
categories:
- Linux
- Network programming
- select
- poll
- epoll
- kernel
- performance
draft: true
comments: true
date: '2015-07-24'
title: A Closer Look at select, poll, and epoll in Linux (Part 1)
url: /2015/07/24/a-closer-look-at-select-poll-and-epoll-in-linux-part-1
---

I am working on a project that requires creating wrappers around `select`, `poll`, and `epoll` system
calls in Linux. Since one of the goals of the project is keeping
it simple, I prefer to deal with only one of them at the moment. Thus, I did some
research to better understand the differences among these polling methods.

There are some online resources that discuss these methods ([here](http://www.ulduzsoft.com/2014/01/select-poll-epoll-practical-difference-for-system-architects/), [here](http://www.makelinux.net/ldd3/chp-6-sect-3), and [here](https://www.kernel.org/doc/ols/2004/ols2004v1-pages-215-226.pdf)), but none of them explain what exactly happens in the kernel space for each system call. The goal of this post is to summarize and simplify what happens in kernel when `select`, `poll` or `epoll` is called. I use Linux kernel v4.2rc5 source code for this post.

All three system calls allow a process to determine whether it can
read from or write to one or more open files without blocking. It is important to understand that
these system calls rely on device drivers that support polling. In
another word, the system polls related devices to see if they are ready for reading
or writing. Upon polling, the device driver performs two tasks:

* Uses `poll_wait` call to indicate that kernel is interested in
events from this device driver. To inform the kernel about future changes the device driver adds
its wait queues (`wait_queeus`) to the `poll_table` structure provided by the kernel. These wait queues
are triggered by driver events and can inform the kernel about the changes in the device.
* The device driver then checks the status of the device for any pending events and return the result as a bitmask to the kernel. Kernel then checks whether the triggered event is what it was polling for.

If there is one or more file descriptors ready, the process
returns them to the calling polling process (`select`, `poll`, or `epoll`).
Otherwise, the process blocks until one of the file
descriptors is ready and the device driver wakes the process up using wait queues.

It is important to keep in mind that setting up and removing the `poll_table`, and adding/removing wait queues to the table is expensive. To add or remove wait queues from the poll table, kernel has to grab a spinlock using `spin_lock_irqsave` which means disabling and restoring interrupts in addition to the potential contention for the spinlock. You can find more details on how polling device drivers work [here](http:/
/www.makelinux.net/ldd3/chp-6-sect-3).

In the following, I explain all three system calls in more details. I explain the data structures and exploit
pseudo code to describe what happens in user and kernel space upon polling. The psuedo code is based on a simple server, where the server polls a socket for incoming connections, and add each new connection to the polling list and
then polls all connections for read, and prints the received string on the screen.

## select

`select` uses `fd_set` to for polling devices. In this set each file descriptor is mapped to a bit in `fd_set`, and this bit set is used to pass the list of file descriptors to the kernel as is explained later. Here is how `fd_set` is defined in kernel:

```
/* The fd_set member is required to be an array of longs.  */
typedef long int __fd_mask;

/* It's easier to assume 8-bit bytes than to get CHAR_BIT.  */
#define __NFDBITS	(8 * (int) sizeof (__fd_mask))

/* fd_set for select and pselect.  */
typedef struct
  {
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
#define __FDS_BITS(set) ((set)->__fds_bits)
  } fd_set;

/* Maximum number of file descriptors in `fd_set'.  */
#define	FD_SETSIZE		__FD_SETSIZE
```

In another words `fd_set` contains a bit set with `__FD_SETSIZE` bits, and the size of the set is limited by `__FD_SETSIZE`. Since, `fd_set` is the only tool for users to inform kernel about the file descriptors they are interested in, and also the main vehicle for kernel to inform users about the potential changes, it is clear that `select` can only monitor up to `__FD_SETSIZE` number of file descriptors simultaneously (which is set to 1024 by default).

To perform polling using `select`, each time three separate `fd_set` should be setup to poll for `read`, `write`, and `exception` events on a specific file descriptor. `FD_ZERO` is used to unset all bits on a `fd_set`, and `FD_SET` is used to set the related bit to the file descriptor we are interested in. `select` requires to pass the maximum file descriptor number that is passed in the `fd_set`, thus it is user's responsibility to provide this information to kernel.

It is also important to note that since kernel uses the same `fd_set` structure to report the results back to the users space (which means it overrides the value of the bits in the original `fd_set`s passed from userspace), user code has reset the bits on these structures every time it performs a `select` system call. It is possible to make a copy of the original `fd_set` and use it again if there is no need to monitor a new file descriptor. However, if the file descriptors change frequently (e.g., new connections arrive and old connections disconnect), user has to iterate through all file descriptors, add them to the `fd_set`, and find the max file descriptor to pass to `select`. Thus, user have to keep track of all available connections so they can iterate through them before `select` is called.

As mentioned earlier, kernel return the results in form of `fd_set`, where the bit matching the file descriptor is set if the device is ready to perform the related task or an exception occurred (in the realted `fd_set`). Thus, user has to loop through all the bits (up to max file descriptor) and check whether the bit is set using `FD_ISSET`, and perform the proper task if the bit is set.
Lets use pseudo code to explain this process further. Green means the code is executing in user space, and red means the code is executing in the kernel space (due to the `select` system call). This pseudo code is based on a simple server that reads the data from each connected socket and echo it back. You can find a sample code [here](http://www.binarytides.com/multiple-socket-connections-fdset-select-linux/).

{{< >}}
<div class="pseudocode">
#### User Space

1. Declare and initialize server sockets, `fd_set readfds`, maximum file descriptor to poll (`max_sd`)
2. Infinite loop (need to call `select` again after kernel returns the results):
    1. Use `FD_ZERO` to reset `readfds`
    2. Use `FD_SET` to add server socket to `readfds`
    3. Loop through client sockets (we keep track of them in an array):
        1. Use `FD_SET` to set the related bit in `readfds`
        2. Check if this is the highest fd in the set: if `fd > max_sd`, then `max_sd = fd`.

    4. Do a `select` system call without write fd set, except fd set, and timeout: `select(max_sd+1, &readfds, NULL,
       NULL, NULL);`
<div class="kernel-space"><b>Kernel Space</b></span>
        1. Copy `timeout` from User-Space to Kernel-Space (if there is a timeout).
        2. Copy readfds, writefds, and exceptfds from User-Space to Kernel-Space. call `do_select`:
            1. Initialize `poll_table` and poll wait queues `poll_wqueues`
            2. Set `retval = 0` (this is the result returned to the user space when `select` returns)
            3. Infinite loop
                1. Loop through fdsets
                    1. Find the related device for the passed file descriptor
                    2. Call the `poll` function of the device and send the `poll_table`
                    3. The device driver adds its wait queues to the poll table and
                    checks whether any event is pending on the device so it can inform the
                    polling process about it. It returns a bitmask to the kernel
                    3. Kernel checks the returned bitmask to find out whether
                       the fd is ready for read or write, or if an exception happened. If so do
                       `retval++`, and set the related bit in readfds, writefds or
                       exceptfds.
                2. If `retval>0` or `timeout` occurred `break` the infinite loop (at least one fd is ready or timout happened).
                3. Else sleep until `timeout` is reached or one of the devices is ready to read, write or an exception happened (The device driver will wake the process up). Then simply continue the loop and poll devices again.
            4. Delete `poll_table` and `poll_wqueues` structures by looping through the
                entries and remove `poll_entry` and all wait queues
            5. return retval that indicates the total number of bits set
        3. Copy readfds, writefds, and exceptfds back to userspace (these will be copied in the same memory address as the original `fd_set` passed from the user space and override them).
        4. return `retval` to user space
    5. Check server socket with `FD_ISSET`, If the related bit is set; there are new connections:
        1.  `accept` the new connection
        2. add the new socket to the array of active connections (User has to keep track of current connections)
    6. Loop through the list of connections and check whether the related bit is set in `readfds`
       1. If the bit is set, read the data on the socket and print the result
       2. If connection is lost close the socket and remove it from the list of active connections
</div>
{{% %}}

Here are some of the points that requires to be highlighted:

* `fd_sets` passed to `select` will be modified and cannot be reused. Each time
  there is a call to `select`, these `fd_sets` should be recreated or restored
  from a back up. This means, user has to keep track of open connections and
  closed connections in user space and either create an `fd_set` based on that
  information or maintain a backup `fd_set` as new connections are established
  or closed.
* Since `select` is relying on `fd_sets` to communicate to the user level, it
  cannot maintain more than `FD_SETSIZE` connections simultaneously (default
  1024).
* However, `fd_sets` are extremely small in size and copying them from user
  space to kernel space and vice versa can be done very fast.
* Another overhead for the user is to calculate the largest descriptor number
  and pass it to `select`.
* The descriptor set cannot be modified from another threads when `select` is
  polling devices for readiness. The infinite loop inside the kernel shows that
  when select is
  polling device drivers, it sets up a `poll_table` and `wait_queues` and loop
  through descriptors to see if any of therm is ready, or the process sleeps
  until one of the descriptors becomes ready. During this time, there is no way
  for other threads to reach the kernel and ask for removal of a specific
  fd from the list, or stop waiting on the descriptor. Also, as __man select__
  specifies: "If a file descriptor being monitored by select() is closed in
  another thread, the result is unspecified". So there is no way to remove, add,
  or modify the `fd_sets` when `select` is polling in kernel space.
* Everytime `select` is called, the `poll_table`

Now lets do some calculations about the cost of `select` assuming we have _N_
total connections of which _M_ connections are active during the `select` call:

## poll

`poll` uses an array of `pollfd` structures for polling. `pollfd` structure is as follows:

```
/* Data structure describing a polling request.  */
struct pollfd
  {
    int fd;     /* File descriptor to poll.  */
    short int events;   /* Types of events poller cares about.  */
    short int revents;    /* Types of events that actually occurred.  */
  };

```
`fd` is the file descriptor, `events` mark the type of events we are interested in, and `revents` is returned(updated) by the kernel to show the events that actually occured. It is immediately clear that (unlike `select`) there is no limit on the number of file descriptors that can be monitored when using poll (not entierly true, the limit is RLIMIT_NOFILE). Also, it is possible to choose exactly what file descriptors we are interested in by creating a new `pollfd` and setting the `fd`. Keep in mind that with `select`, it is possbile to monitor many file descriptors using few bytes but with `poll` the array can grow big if the number of file descripors are large.

In addition, `pollfd` structure has separate fields for requested events and returned events. Thus, (unlinke `select`) the original requested events are not being changed and it's possible to recycle the struct and the array for the next poll request. If an item should be removed, the `fd` can be set to a negative value and kernel will ignore that file descriptor.

Here is the related pseudo code to create a server using `poll`:

{{< >}}
<div class="pseudocode">
#### User Space

1. Declare and initialize the server socket, bind, and listen. Then setup `struct pollfd poll_set[SIZE]`, maximum file descriptor to poll (`max_fd`). Add the server socket fd to `poll_set`
2. Infinite loop (need to call `poll` again after kernel returns the results):
    1. Do a `poll` system call without a timeout: `poll(poll_set, max_fd, NULL);`
<div class="kernel-space"><b>Kernel Space</b></span>
        1. Copy `pollfd` structe to `poll_list` an internal `pollfd` linked list to keep track of fds.
        2. Create and initialize a `poll_wqueues` structure.
        2. Call `do_poll` and pass the `poll_wqueues` structure:
            1. Initialize `poll_table` and poll wait queues `poll_wqueues`
            2. Set `count = 0` (this is the result returned to the user space when `poll` returns)
            3. Infinite loop
                1. Loop through `pollfd`s
                    1. Call `do_pollfd` and pass the `pollfd`
                        1. Find the related device for the passed file descriptor
                        2. Call the `poll` function of the device and send the `poll_table`
                        3. The device driver adds its wait queues to the poll table and checks whether any event is pending on the device so it can inform the polling process about it. It returns a bitmask to the kernel.
                        4. From the result of the device `poll` kernel checks whether the fd is ready for read or write, or if an exception happened. If so, it set the `revents` for the related `pollfd`.
                        5. return the bitmask returned from device polling.
                    2. If the returned result is not 0, increment the `count` value.
                2. If `count > 0` or `timeout` occured, break the infinite loop.
                3. Else sleep until `timeout` is reached or one of the devices is ready to read, write or an exception happened (The device driver will wake the process up). Then simply continue the loop and poll devices again.
                4. Return `count`.
            4. Delete `poll_table` and `poll_wqueues` structures by looping through the
                entries and remove `poll_entry` and all wait queues
            6. Copy all revents from `poll_list` to user space `pollfd`s
            7. Free `poll_list`
            5. Return `count` received from `do_poll` or error if an error
               happened.
        3. return `count` to user space
    3. Check for errors by check whether the returened result is less than 0
    4. loop through `poll_set`
        1. If `revents` is equal to 0, continue
        2. if the `fd` is equal to server socket `fd`
            1. `accept` the new connection
            2. add the new socket to `poll_set` and set the events to `POLLIN`
        3. else read the data on the socket and print the result and if connection is lost close the socket and set the related fd in `poll_set` to -1
</div>
{{% %}}


## epoll

<script type="text/javascript"
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML">
</script>