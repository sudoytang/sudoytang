---
title: "QEMU's Main Loop, vCPU Threads, BQL, and Bottom Halves"
tags: [qemu, concurrency, virtualization]
---

# QEMU's Main Loop, vCPU Threads, BQL, and Bottom Halves

A two-vCPU VM does not mean QEMU has two threads total, or that every device callback runs on one “main thread.” Separate the vCPU threads from QEMU's event loops.

## Threads and MMIO

With KVM, QEMU normally has one host thread per vCPU, plus one main-loop thread. Multi-threaded TCG also uses per-vCPU threads when supported; single-threaded TCG is an exception. Optional **IOThreads** add more event-loop threads, and devices may also create worker threads.

The main loop is the process's default event loop. Its `AioContext` handles file-descriptor readiness, timers, bottom halves (BHs), and other asynchronous work. An IOThread has a separate `AioContext`; it is not a vCPU.

For an ordinary emulated MMIO access, the callback usually runs on the vCPU thread that issued the access. With KVM, a trapped MMIO access exits to QEMU on that vCPU's path. Some devices or configurations route work elsewhere, for example through `ioeventfd` or an IOThread, so the exact path depends on the device and accelerator.

## BQL and BHs

The **Big QEMU Lock (BQL)** is a mutex that serializes access to legacy QEMU state that is not safe for concurrent use. The main loop and vCPU threads acquire it around work that requires this protection; it is not held continuously while the main loop waits. An IOThread callback does not automatically hold the BQL, so code there needs the synchronization required by its own `AioContext` and device.

A **bottom half (BH)** is a deferred callback attached to an `AioContext`. Scheduling a BH asks that event loop to run the callback soon. A BH is not a worker thread and does not make its callback parallel.

A common worker-to-INTx path is:

```text
device worker
  ├─ publish completion / interrupt status under a mutex or atomics
  └─ schedule a BH on the device's owning AioContext
        └─ event loop wakes and runs the BH
              └─ recompute the interrupt level and call pci_set_irq()
```

For traditional device-model code, the main-loop context is often the right place for that final PCI state change. Use the device's IOThread context if the device is assigned there, and do not assume an IOThread BH has the BQL. The worker and callback still need a safe way to share completion and interrupt state.

PCI INTx is **level-triggered**. Keep the line asserted while an enabled interrupt cause remains pending; deassert it when the device's status/acknowledgment logic clears that cause. An asynchronous completion changes when QEMU updates the line, not the level-triggered model.

## Crossing between QEMU instances

When one QEMU instance raises an interrupt in a device owned by another instance in the same process, a direct call does not change threads. `pci_set_irq()` propagates the PCI IRQ synchronously on the calling thread; the guest interrupt handler runs later on the x86 vCPU selected by interrupt routing. Thus a direct call from an embedded RISC-V vCPU would run the host-side x86 PCI/IRQ path on that RISC-V vCPU thread.

Prefer handing an IRQ event to the target instance: enqueue the desired IRQ state and schedule a BH on the target's owning `AioContext`. The target-side callback updates its device state and recomputes the INTx level under the synchronization required by that path. Use the same rule in the reverse direction. A BH scheduled on the source context does not transfer ownership, and a BH scheduled on a context that is not being polled will not run.

The BQL serializes only code using that same lock; acquiring it does not transfer execution to the target instance's thread. In an embedded or multiply linked setup, verify whether both QEMU cores actually share the same BQL rather than assuming there is one lock per VM or one process-wide lock. Avoid waiting for the other instance while holding source-side locks, and define a consistent lock order for any locks that must cross this boundary.

## BH ordering and starvation

Do not rely on a FIFO guarantee. QEMU's current implementation inserts pending BHs at the head of a singly linked list, so LIFO execution can be observed. That is an implementation detail, not an ordering contract for device code. For a reusable BH, scheduling it again while it is already pending is coalesced; it is not an event queue where every schedule call must produce a separate callback.

A finite batch of pending BHs is drained, so LIFO by itself does not mean an older callback disappears. But QEMU does not promise general fairness: a long callback, a BH that continually reschedules itself, or prolonged BQL contention can delay other work. Keep BHs short and let the event loop regain control between chunks of work. A vCPU issuing repeated MMIO can add contention, especially if callbacks hold the BQL for too long.

## What idle looks like

When there is no ready work, the main loop normally blocks in the host event poller until an fd, timer, BH notification, or other event wakes it. It is not designed to spin continuously while idle. Some IOThreads support busy polling to trade CPU time for lower wake-up latency; that is a separate configuration choice.

### References

- [Using Multiple IOThreads](https://www.qemu.org/docs/master/devel/multiple-iothreads.html)
- [Multi-threaded TCG](https://www.qemu.org/docs/master/devel/multi-thread-tcg.html)
- [QEMU Memory API](https://www.qemu.org/docs/master/devel/memory.html)
- [QEMU BH implementation](https://github.com/qemu/qemu/blob/master/util/async.c)
- [QEMU PCI interrupt implementation](https://github.com/qemu/qemu/blob/master/hw/pci/pci.c)
