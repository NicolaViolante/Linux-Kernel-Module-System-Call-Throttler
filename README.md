# Linux Kernel Module: System Call Throttler

<p align="left">
  <img src="https://img.shields.io/badge/C-00599C?style=for-the-badge&logo=c&logoColor=white" />
  <img src="https://img.shields.io/badge/Linux%20Kernel-FCC624?style=for-the-badge&logo=linux&logoColor=black" />
  <img src="https://img.shields.io/badge/Makefile-31A8FF?style=for-the-badge&logo=gnu&logoColor=white" />
</p>

## 📝 Overview
This repository contains a Loadable Kernel Module (LKM) developed for Linux (kernel version 6.17.12) that implements a robust mechanism for **system call throttling**. The module allows a system administrator to dynamically monitor specific processes, User IDs (UIDs), and x86-64 system calls. 

When a monitored system call is invoked by a targeted process or UID, it passes through a custom monitor. The monitor enforces a configurable threshold, limiting the maximum aggregate number of invocations permitted within a one-second window based on a lock-free token-bucket algorithm. If the threshold is exceeded, the invoking threads are temporarily blocked and placed in interruptible wait queues. 

## ⚙️ System Architecture & Mechanics
The module is designed with a strong emphasis on extremely low overhead, cache-friendly data structures, and safe concurrency management.

- **Syscall Hooking & Trampoline Injection:**
  - The module dynamically locates the hidden system call table by scanning kernel memory for `sys_ni_syscall` patterns. 
  - To bypass Read-Only kernel protections, it disables preemption and modifies the `cr0` and `cr4` control registers. 
  - Since the global table is mostly a legacy artifact, the module performs inline hooking directly on the `x64_sys_call` function, redirecting execution to a custom assembly trampoline that includes the `endbr64` instruction for Intel CET compatibility.
- **Lock-Free Synchronization & RCU:**
  - Tracked UIDs and program names are organized into 256-bucket hash tables.
  - Read operations inside the wrapper are completely lock-free, orchestrated by the Read-Copy-Update (RCU) mechanism to prevent execution bottlenecks. Write operations are protected by spinlocks for rapid mutual exclusion.
- **Token-Bucket Throttling Engine:**
  - Tokens are stored in an atomic variable, allowing fully concurrent lock-free subtraction. 
  - Tokens are replenished either passively (when a thread detects a one-second transition) or actively via a dynamically scheduled system timer if threads are queued.
- **Per-CPU Statistics Collection:**
  - To prevent lock contention and cache invalidation on multiprocessor systems, operational metrics (such as peak blocked threads and maximum delays) are collected using Per-CPU variables. These local copies are aggregated on-demand only when requested by the userspace driver.
- **Userspace Interface & IOCTL:**
  - A virtual character device acts as the bridge to userspace. The driver implements a centralized `ioctl()` dispatcher, securely transferring configuration rules and aggregate metrics using `copy_from_user()` and `copy_to_user()` while strictly enforcing root privileges.

## 🚀 Quick Start
To build and load the kernel module, run the following commands in sequence in your terminal.

**1. Compile the module and userspace tools:** `make all`

**2. Insert the module into the kernel:** `sudo insmod throttler.ko`

**3. Verify the module is loaded:** `lsmod | grep throttler`

**4. Interact with the module via the provided userspace controller:** `./controller [args]`

## 🧹 Teardown
To safely shut down the monitor, unhook the system calls, and remove the module:

`sudo rmmod throttler`

> **Note:** The module implements a safe unload sequence. It guarantees that all in-flight threads have completely exited the wrapper (using per-CPU counters) before restoring the original memory bytes and initiating the `rcu_barrier()` cleanup.

---
*Developed by Nicola Violante as an academic project for the Advanced Operating Systems course at the University of Rome Tor Vergata.*
