# Weenix Kernel

A Unix-like kernel implemented in C based on the Weenix project from Brown University. Boots stably to a userland shell on QEMU (x86-64).

> Note: Source code is kept private to respect academic integrity guidelines. This repo documents what was implemented.

## What I Implemented

### K1 — Process & Thread Management

- Process lifecycle — creation, destruction, parent-child relationships, orphan reparenting to init
- Threads — kernel thread creation, cancellation, clean exit
- Cooperative scheduler — FIFO run queue; threads run until they voluntarily yield or sleep
- Wait queues — threads sleep on events and wake on signal
- Mutexes — lock/unlock with wait queue-based sleeping; direct handoff to next waiter on unlock
- Interrupt masking — used IPL_HIGH instead of mutex to protect run queue from interrupt handlers (interrupt handlers can't sleep, so they can't acquire a mutex)
- Clean shutdown — thread cancellation, proc_kill_all, graceful kernel halt

### K2 — Virtual File System

- VFS abstraction — vnode operations dispatched to filesystem-specific implementations
- Path resolution — tokenize path, walk directory tree component by component
- Vnode reference counting — prevents use-after-free when multiple processes share the same file
- Ordered vnode locking — always lock by inode number to prevent deadlocks
- Syscalls — read, write, close, dup, dup2, mkdir, rmdir, unlink, link, rename, stat, chdir, lseek, getdent, mknod

### K3 — Virtual Memory

- Per-process address spaces — each process has its own page table
- vmmap/vmarea — data structures tracking mapped regions per process
- Anonymous memory objects — zero-filled pages on demand for heap and stack
- Shadow objects — implement copy-on-write; intercept writes and store private page copies per process
- Copy-on-write fork — parent and child share physical pages marked read-only; page fault triggers private copy of only the written page
- TLB flush after fork — invalidate stale cached translations after marking pages read-only
- vmmap_read/write — kernel-to-user memory access primitives

## Architecture

```
Userland shell (/sbin/init)
        ↓
Syscall interface
        ↓
┌─────────────────────────────────────┐
│ K2: Virtual File System             │
│  vnode → vnode_ops → filesystem     │
│  path resolution, reference counting│
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ K3: Virtual Memory                  │
│  vmmap, shadow objects, COW fork    │
│  per-process page tables            │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ K1: Process & Thread Management     │
│  scheduler, mutexes, wait queues    │
│  interrupt masking                  │
└─────────────────────────────────────┘
        ↓
Hardware (x86-64, QEMU)
```

## Key Design Decisions

**Cooperative scheduling** — no timer interrupts, threads switch only when they yield or block.

**Interrupt masking over mutex for run queue** — interrupt handlers modify the run queue but can't sleep, so they can't acquire a mutex. IPL_HIGH blocks all interrupts during critical sections instead.

**Ordered vnode locking** — when two vnodes must be locked simultaneously, always lock in inode number order to prevent circular wait deadlock.

**Copy-on-write shadow chain** — each shadow object tracks only its private pages. On write fault, copy the page into the shadow's private list. Parent and child never interfere.

## Build & Run

```bash
make clean && make
./weenix -n
```

## Result

```
init: starting shell on tty0
weenix ->
```

Full boot with DRIVERS=1, VFS=1, S5FS=1, VM=1 enabled.

## Tech Stack

C, x86-64, QEMU, GDB, GNU Make
