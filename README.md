 Project: Windows Gaming Interoperability on Linux

## 1. Objective

Create a software and hardware architecture that allows running games designed for Windows from a Linux system with the highest possible compatibility, performance, and transparency.

The primary goal is NOT to replace Linux with Windows or rely on Windows as the main desktop environment. The objective is to bridge the compatibility gap so that Windows games can be utilized from Linux with an experience close to native execution.

## 2. Project Philosophy

Principles:

- Prioritize real-world compatibility and performance over claims of "invisibility."
- Measure each optimization with reproducible benchmarks.
- Prefer standard and maintainable technologies.
- Avoid unnecessary kernel or firmware modifications.
- Keep experimental components separate from stable parts.
- Document issues, workarounds, and regressions.

## 3. Proposed Architecture

### Main Route

Linux
→ Wine/Proton
→ DXVK/VKD3D-Proton
→ Windows Game

This must be the primary route because it allows running the game without a VM whenever compatible.

### Advanced Compatibility Route

Linux
→ KVM/QEMU
→ Windows
→ VFIO
→ Dedicated Physical GPU
→ Dedicated Storage
→ Physical USB Devices

This route is utilized for games or software that require a more complete Windows environment.

## 4. KVM/QEMU

Investigate and optimize:

- CPU passthrough / `host-passthrough`
- Correct CPU topology
- CPU pinning
- CPU isolation when appropriate
- Hugepages
- Timer configurations
- Invariant TSC
- VirtIO when the virtual device is appropriate
- I/O overhead reduction
- QEMU/libvirt configuration
- Measuring VM exits and overhead when useful

Do not assume that eliminating a VM exit means zero latency. All claims must be verified through measurements.

## 5. VFIO / PCI Passthrough

The goal is to pass dedicated physical devices directly to the guest VM when necessary.

Possible devices:

- Secondary GPU
- Dedicated NVMe controller
- Dedicated NIC
- Dedicated USB controllers

Requirements to investigate:

- IOMMU/VT-d/AMD-Vi
- IOMMU groups
- Correct binding to `vfio-pci`
- Device reset mechanics
- Dependencies between devices
- Firmware/BIOS
- GPU compatibility with reset and passthrough

Never assume concrete PCI IDs; they must be detected on the actual hardware.

## 6. CPU and Timing

Investigate:

- Invariant TSC
- `host-passthrough`
- `invtsc` when appropriate
- Guest clock synchronization
- CPUID/RDTSC behavior
- Effects of VM exits
- CPU scheduling
- Isolation/pinning

Avoid solutions based on artificially modifying the guest time to hide overhead. A temporal compensation can produce inconsistencies.

## 7. Memory

Evaluate:

- Hugepages
- Transparent Huge Pages depending on the case
- NUMA
- Memory affinity
- Memory allocation to QEMU
- Swapping impact
- Latency and bandwidth

Do not confuse `/proc/sys/vm/compact_memory` with memory isolation. Memory compaction does not make QEMU invisible or create process isolation.

## 8. ACPI and SMBIOS

SMBIOS can be configured to provide coherent information to a guest system when a legitimate compatibility reason exists.

Do not use a DSDT copy from another computer as a general solution. ACPI tables depend on the specific hardware and firmware and can introduce incompatibilities.

Do not assume that changing ACPI turns QEMU into a specific physical chipset like Z790 or X670.

## 9. Hyper-V and Windows

Investigate Hyper-V features exposed by KVM/QEMU and Windows compatibility with nested virtualization when necessary.

Separate:

- CPU vendor identification.
- Hyper-V enlightenments.
- VBS.
- Hyper-V nested virtualization.
- KVM features.

Do not assume that a few XML flags reproduce a physical Hyper-V system bit by bit.

## 10. GPU and Graphics

Priority:

1. GPU passthrough for Windows when necessary.
2. Official GPU driver inside Windows.
3. Performance measurement against bare-metal Windows.
4. Comparison of FPS and frametime.
5. Verification of VRR, HDR, resolution, and refresh rate.

## 11. Input

Prioritize standard physical USB devices or USB passthrough when appropriate.

The project must not depend on modifying commercial firmware or creating devices designed to hide their origin.

Measurable objectives:

- Input latency
- Polling rate
- Stability
- Event loss
- Keyboard/mouse behavior

## 12. Capture and Interface

As an alternative architecture, the following can be investigated:

Windows GPU
→ HDMI/DisplayPort output
→ Capture card
→ Linux

This would allow visualizing Windows from Linux without Linux having to read the game process memory directly.

Do not use "zero latency" as an assumption. Measure the full capture and presentation latency.

## 13. Network

Investigate:

- NIC passthrough
- VirtIO networking
- Direct Ethernet
- 10 GbE when hardware justifies it
- Latency
- Jitter
- Throughput

Choose the solution based on the actual needs of the game.

## 14. Security and Anti-Cheat

The project's objective is compatibility and performance.

It should not be assumed that a configuration exists capable of guaranteeing a VM is indistinguishable from physical hardware against any anti-cheat system.

Virtualization technologies must be evaluated by:

- Stability
- Compatibility
- Performance
- Security
- Maintainability

## 15. Measurement

Each version must be compared against a baseline.

Metrics:

- Average FPS
- 1% low
- 0.1% low
- Frametime
- Input latency
- CPU usage
- GPU usage
- VRAM
- RAM
- Temperature
- Power consumption
- Stability
- Loading times
- Network performance

Recommended comparisons:

A. Bare-metal Windows
B. Windows VM without optimizations
C. Windows VM + optimizations
D. Wine/Proton

## 16. Development Plan

### Phase 0 — Inventory

Obtain:

- CPU
- GPU(s)
- RAM
- Motherboard
- Storage
- NIC
- USB controllers
- IOMMU support
- Linux kernel
- QEMU/libvirt version

### Phase 1 — VM Prototype

Create a functional Windows VM using KVM/QEMU.

### Phase 2 — GPU Passthrough

Assign a dedicated GPU to Windows via VFIO.

### Phase 3 — Optimization

Add and measure:

- CPU passthrough
- Pinning
- Hugepages
- Timers
- I/O
- Network
- USB

### Phase 4 — Benchmark

Compare VM, Proton, and bare-metal Windows.

### Phase 5 — Integration

Create an orchestrator that automates the environment preparation and execution.

### Phase 6 — Compatibility

Create a database of known games and issues.

## 17. Future Orchestrator

The orchestrator should:

1. Detect hardware.
2. Verify IOMMU status.
3. Detect IOMMU groups.
4. Check available devices.
5. Prepare VFIO bindings.
6. Create/validate VM configuration.
7. Start Windows.
8. Apply CPU/memory/I/O configurations.
9. Log metrics.
10. Restore devices to the host upon termination.

Never execute destructive actions on devices without first verifying their function.

## 18. Previous Experimental Code

The four original documents form part of the design history and must be preserved as a conceptual reference.

### Document 1

Explored:

- KVM
- TSC
- SMBIOS
- Hyper-V
- KVM hiding
- SEV/TDX
- Timers

Identified issues:

- Some parameters cannot be modified on the fly.
- SEV/SEV-ES cannot be activated simply by writing to `/sys/module`.
- SMBIOS does not make a VM physically indistinguishable.
- `hyperv vendor_id` does not turn a VM into real Hyper-V.

### Document 2

Explored:

- Invariant TSC
- `MSR_IA32_TSC_ADJUST`
- DSDT/ACPI
- Nested Hyper-V
- VBS
- External HID devices

Identified issues:

- A DSDT from another machine is not a generic replacement for a chipset.
- Nested Hyper-V adds complexity and overhead.
- Eliminating VM exits does not imply zero latency.
- External hardware does not turn a VM into physical hardware.

### Document 3

Explored:

- Bare-metal Windows
- Physical hardware
- Dedicated network
- VBS
- Secure Boot
- TPM
- External capture
- Physical HID

Advantage:

Conceptually coherent as a dedicated Windows architecture controlled from Linux.

Limitation:

No longer a solution for running Windows inside Linux; it is a hybrid architecture between two physical systems/environments.

### Document 4

Explored:

- KVM patching
- Memory isolation
- VFIO
- Physical GPU/NVMe/NIC
- Hyper-V
- Timers

Leveragable elements:

- VFIO/passthrough
- Dedicated GPU
- Dedicated NVMe
- Dedicated NIC
- CPU/TSC investigation
- KVM/QEMU optimization

Identified issues:

- `compact_memory` is not memory isolation.
- `ignore_msrs` is not a performance solution.
- `GenuineIntel` is not a Hyper-V signature.
- PCI IDs should not be assumed.
- Patching `vmx.c`/`svm.c` should not be the first step.

## 19. Engineering Principle

Before modifying the kernel:

1. Reproduce the problem.
2. Measure it.
3. Determine the root cause.
4. Look for a standard configuration.
5. Implement the minimal change.
6. Re-measure.
7. Only consider a kernel patch if there is a demonstrable reason.

## 20. Expected Outcome

The ideal final result is a Linux platform where:

- Most Windows games run via Proton/Wine.
- Problematic games utilize a KVM/QEMU route.
- VMs can leverage physical hardware via VFIO.
- Performance is measured objectively.
- The configuration is reproducible.
- The user does not need to become an expert QEMU administrator to use it.

## 21. Next Step

Do not implement all components yet.

First, create a minimal prototype:

Linux
→ KVM/QEMU
→ Windows
→ GPU Passthrough

Afterwards, measure performance and expand the architecture only when data indicates that an optimization is necessary.
