# Advanced Guide: QEMU CPU Models for KVM Guests - Debunking the "host" CPU Model Performance Myth
> [!NOTE]
> Target audience: single-node homelab setups using AVX2-capable processors.

> [!TIP]
> Always install the latest CPU microcode for your CPU.  
> Details: https://pve.proxmox.com/pve-docs/pve-admin-guide.html#sysadmin_firmware_cpu

---

## Simple Windows Guests
For basic Windows guests without WSL2, Hyper-V, or VBS enabled, use named QEMU CPU models corresponding to your hardware. For example, if you have Skylake CPUs, use `Skylake-Client-v4`.

---

## Hyper-V, WSL2, and VBS Enabled Guests
To achieve the best performance with these features enabled:

### 1. Create Custom CPU Models
Create the file `/etc/pve/virtual-guest/cpu-models.conf` with the following content:

```
# Proxmox VE Custom CPU Models
cpu-model: amd-hide-vm-for-windows
    flags -hypervisor;+invtsc;+hv-frequencies;+hv-reenlightenment;+hv-emsr-bitmap;+hv-tlbflush-direct
    phys-bits host
    hidden 1
    hv-vendor-id amd
    reported-model host

cpu-model: intel-hide-vm-for-windows
    flags -hypervisor;+invtsc;+hv-frequencies;+hv-evmcs;+hv-reenlightenment;+hv-emsr-bitmap;+hv-tlbflush-direct
    phys-bits host
    hidden 1
    hv-vendor-id intel
    reported-model host
```

### 2. Apply the Custom Model
Select either `amd-hide-vm-for-windows` or `intel-hide-vm-for-windows` CPU model in the Proxmox web GUI, depending on your processor.

> [!TIP]
> To run Android emulators like [MEmu](https://www.memuplay.com/) smoothly, you need to have a GPU and enable Hyper-V:
> ```
> Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
> ```

---

## macOS Guests
### Performance Considerations
- Always use named QEMU CPU models instead of `host` passthrough
- The `host` passthrough can result in ~30% slower single-core and ~44% slower multi-core performance compared to named QEMU CPU models

### CPUID Model Compatibility
The macOS kernel checks CPUID model and vendor during boot. When using CPU models like `SapphireRapids` or `GraniteRapids` (which were never shipped in real Macs), you must override the CPUID model to a known Mac-compatible model. While this can be configured in OpenCore's `config.plist` (Kernel > Emulate Properties), QEMU-level configuration is recommended.

> [!TIP]
> **CPUID Models Used in Recent Intel Macs:**
> 
> | Generation | CPUID Model |
> |------------|-------------|
> | Kaby Lake / Coffee Lake | `158` |
> | Comet Lake | `165` |
> 
> **Example Configuration:**
> ```
> qm set <VMID> --args "-cpu SapphireRapids,vendor=GenuineIntel,model=165"
> ```

---

## Bonus: Advanced CPU Spoofing

<img width="1196" height="1012" alt="image" src="https://github.com/user-attachments/assets/90ebe0d4-c2be-4678-8e95-6d5102c780b5" />


### Customizing CPU Brand Name
When using named QEMU CPU models like `Skylake-Client-v4`, the guest OS will report the CPU as `Intel Core Processor (Skylake, IBRS, no TSX)`. You can customize this using the `model-id` parameter.

### Spoofing CPU Frequency
For modern CPUs, you can spoof the reported base speed using the `tsc-frequency` parameter.

### Example: Complete CPU Spoofing
To spoof a 10GHz CPU with the brand name "Intel Core i8-8800KS":

```
# Using global parameters (keeps CPU model from GUI)
qm set <VMID> --args "-global cpu.model-id='Intel Core i8-8800KS' -global cpu.tsc-frequency=10000000000"
```

### Additional Parameters
Many more CPU parameters are available, including:
- `family=` and `model=` - CPU family and model numbers
- `stepping=` and `level=` - CPU stepping and CPUID level
- `host-cache-info=on` - Pass through host cache information

For complete details, see: https://github.com/qemu/qemu/blob/master/target/i386/cpu.c
