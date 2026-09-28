---
title: "QEMU's Main Loop, vCPU Threads, BQL, and Bottom Halves"
tags: [qemu, concurrency, virtualization]
---

# QEMU's Main Loop, vCPU Threads, BQL, and Bottom Halves

A two-vCPU VM does not mean QEMU has just two threads, or that device emulation happens on one universal “main thread.” It helps to separate the vCPU threads from QEMU's event loops.

## Threads and event loops

With KVM, QEMU normally has one host thread per vCPU, plus a separate main-loop thread. Multi-threaded TCG also runs vCPUs on separate threads when supported; single-threaded TCG is an exception. QEMU can create additional event-loop threads called **IOThreads** for devices assigned to them.

The main loop is one event loop for the QEMU process. It dispatches work such as file-descriptor readiness, timers, bottom halves (BHs), and management/UI activity. An IOThread is another event loop, not another vCPU.

For an ordinary emulated MMIO access, the device callback is usually reached from the vCPU's access path: the guest CPU that issued the load or store traps or exits, and QEMU handles that access. The callback is therefore often running on that vCPU's host thread, not on the main-loop thread. The exact path depends on the accelerator and device: mechanisms such as `ioeventfd` or an IOThread can route parts of the work elsewhere.

## BQL and BHs

The **Big QEMU Lock (BQL)** is a mutex used to serialize legacy QEMU code that is not safe to run concurrently. It is not a thread, and the main loop does not hold it continuously while sleeping. The main loop and vCPU threads acquire it around work that requires this serialization. IOThread code is generally designed to run outside the BQL and must use the appropriate `AioContext` APIs and its own synchronization.

A **bottom half (BH)** is a deferred callback attached to an `AioContext`. It is a way to ask that event loop to run a small piece of work soon; it does not create a worker thread or make the callback execute in parallel.

If a device worker thread completes an operation and needs to change QEMU device state or raise an interrupt, it should not assume it can safely call `pci_set_irq()` directly. A common pattern is to publish the completion/IRQ state with the required synchronization, then schedule a one-shot BH on the device's owning `AioContext`. The BH applies the state change in that event loop's synchronization domain. Use the main-loop context only if that is where the device operation belongs; an IOThread-owned device should use its IOThread context.

For legacy PCI INTx, the line is level-triggered: assert it while the device has a pending interrupt, and deassert it when the device's condition is cleared. Scheduling the transition asynchronously does not change that state model.

## Ordering, starvation, and idle time

Do not make correctness depend on BH callbacks being FIFO. QEMU's current implementation uses a lock-free singly linked list that inserts newly queued BHs at the head, so LIFO behavior can be observed. The public scheduling model is still not a general-purpose ordered work queue: concurrent scheduling and rescheduling can affect what is seen in each poll. A finite batch of pending BHs is drained, so LIFO by itself does not mean an older callback is automatically lost. The practical starvation risks are a callback that blocks for too long, or work that continually reschedules itself and keeps the event loop busy.

When there is no work to dispatch, the main loop normally waits in the host event poller until a file descriptor, timer, notification, or other event wakes it. It does not need to spin continuously. Busy polling can be enabled for some IOThreads and trades CPU time for lower wake-up latency.

### References

- [Using Multiple IOThreads](https://www.qemu.org/docs/master/devel/multiple-iothreads.html)
- [Multi-threaded TCG](https://www.qemu.org/docs/master/devel/multi-thread-tcg.html)
- [QEMU BH implementation](https://github.com/qemu/qemu/blob/master/util/async.c)
- [QEMU main-loop interface](https://github.com/qemu/qemu/blob/master/include/qemu/main-loop.h)
