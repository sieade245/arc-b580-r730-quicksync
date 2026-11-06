# Appendix B · PCI allocation and boot

[Main report](README.md) · [Methods and results](appendix-a-tests.md)

## What the firmware supplied

On stock Dell BIOS 2.18.1, the B580 endpoint advertised seven BAR2 sizes, from 256 MiB to 16 GiB. Firmware assigned the smallest aperture, with selector 8. A separate 16 MiB GPU BAR is outside this size sweep. The GPU was behind a card switch. The upstream switch used a separate 8 MiB resource in the same parent prefetchable window. The root port's assigned 64-bit prefetchable window was 264 MiB. A wider host resource envelope was visible to the OS, but each bridge and its children still needed a coherent assignment.

The hardware gave us three separate things to verify on every resize. The Resizable BAR selector has to represent the requested size. The assigned BAR start and length must agree with the selector. All ancestor windows must cover that assigned interval without overlaps or collateral changes. A selector by itself cannot establish a working aperture.

## Three Linux corrections

| Gate | Observed failure | Correction and verification |
| --- | --- | --- |
| Small-aperture media allocation, WR-0011/0012 | The original Intel media userspace did not reliably complete the small-BAR media path. | Backport Intel media-driver commit c3e1867 to the tested iHD build. It requests CPU-visible VRAM for the relevant Xe GEM allocations and has a fallback. This is a media allocation change and leaves the BAR unchanged. Hardware uploads and encodes then passed the rolling-pool tests at 256 MiB. [Intel patch](https://github.com/intel/media-driver/commit/c3e1867d6de236102951ee40a75319dbf54beb79) |
| Bridge capacity, WR-0012/0013 | With Xe unloaded, a 512 MiB endpoint resize returned ENOSPC under the original 264 MiB window. | A narrowly scoped Linux helper extended the root port's prefetchable window downwards with the upper limit fixed. It reserved the extra prefix with the kernel resource tree, checked parent and sibling conflicts and read back bridge configuration. The helper left already-assigned resources in place at this step. Linux then assigned the new BAR2 location through its PCI resource resize path. Xe rebind and hardware upload/readback passed at 512 MiB. |
| Media device discovery, WR-0013/0014 | The GPU came back on renderD129 after rebind. The runtime returned `MFX_ERR_NOT_FOUND` (−9) before any frames. Local fixtures reproduced a discovery defect when lower render-node attributes were absent. The full historical device inventory was unavailable. | Backport Intel GPU runtime commit 5376a299. It changes two scan exits to continue past missing entries. The next hardware sweep found the correct device on renderD128 through renderD134 and passed both modes at all seven sizes. [Intel runtime patch](https://github.com/intel/vpl-gpu-rt/commit/5376a299a84b8a477c4225229e2ce12759c619db) |

These changes solve different layers. A media fix can make 256 MiB useful. An enlarged bridge can make a larger BAR assignable. A media runtime can still fail to discover a GPU that has been resized and rebound successfully.

In the Linux sweep, root window expansion grew the parent from 264 MiB to 32,520 MiB inside the existing 256 GiB host envelope. The subsequent GPU resize may relocate BAR2 and adjust the lower switch windows. The tested helper constrained the action to the known R730 root port and card branch, checked its endpoint IDs and current resources, required an unbound target, checked conflicts and stopped on uncertainty. The USB tool recorded its intent before mutating the resource and compared readback afterwards. No global PCI rescan, firmware flash, or changes to unrelated storage controllers were needed for the verified path. [Linux PCI sysfs ABI](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci) · [Linux PCI driver API](https://docs.kernel.org/driver-api/pci/pci.html)

## Making the 16 GiB Debian boot repeatable

The persistent Debian 13 image used a package containing the guarded policy, a DKMS build of the root-window helper, an initramfs hook, and a systemd retry path. This is a design description for review and porting. The draft supplies measurements and the upstream references. The boot image and the private implementation remain local.

1. Identify the exact board, root port, card switch, endpoint and removable boot volume. Preserve a normal boot entry and a bypass entry. Fail closed on an unexpected topology, active driver, incompatible module, outside-resource change or unknown boot medium.
2. Build the helper for each installed kernel and include the matching module and the policy in that kernel's initramfs. Check the module's kernel version and the archive contents after installing headers and building the initramfs.
3. Gate automatic Xe binding while the validated resource change is pending. Once the correct boot volume is available, extend the root port window if it is still at the verified default, request BAR2 size 16 GiB using Linux PCI resize, and verify the selector, assigned interval and ancestor coverage. Load Xe and check the effective resource again.
4. On the first physical USB boot, the early hook correctly refused before USB partitions appeared. The final image allowed one guarded retry after the enrolled USB root was mounted. The retry used the same checks and only performed the change while the GPU remained unbound.
5. Check Xe binding, actual BAR2 size, the media render device and a real encode with verified pixels. Keep a recovery kernel and repeat these checks after updating. The physical update verified automatic reapplication and a 96-frame encode per mode. The 90-second test was run on the prior kernel.

Read-only checks to locate the GPU and inspect its assigned BAR on a similar Linux machine are below. Replace the example address with the device found on the machine. Resource line 3 is BAR2, as the file starts at BAR0. The lspci bridge tree is a starting point for checking ancestor coverage.

```sh
lspci -Dnn -d 8086:e20b
lspci -Dtv
gpu=0000:05:00.0
lspci -D -s "$gpu" -vv
sed -n '3p' "/sys/bus/pci/devices/$gpu/resource"
readlink "/sys/bus/pci/devices/$gpu/driver"
```

## Why the Windows UEFI path stopped

WR-0021 verified a USB application calling UEFI LoadImage and StartImage on the stock Microsoft boot manager. Windows Server 2022 came up normally. Its ConfigMgr resource descriptor still showed the 256 MiB B580 BAR and a 64 MiB D3D11 copy test returned zero errors. This qualifies a no-change handoff and driver health check. There was no enlarged allocation or encode qualification in that run.

| Layer | Finding on this system |
| --- | --- |
| ACPI host resource | PCI0's prefetchable address envelope was 256 GiB. It describes available host address space at that layer. It does not grant an endpoint or a child bridge ownership of all of it. |
| UEFI resource descriptor | The reported prefetchable range was 265 MiB, distinct from the live root bridge's 264 MiB prefetchable window. Inspect configuration registers and child resources as well as protocol descriptors. |
| In-place growth | The original GPU interval could not grow to 512 MiB at its assigned base while retaining the observed neighbouring allocations. |
| Offline placement | A different aligned base inside the host envelope could fit 512 MiB in the saved resource map, with a 776 MiB root window. An aligned 16 GiB candidate also fit the offline envelope. Neither candidate established ownership or a live allocation. |
| Reconnect timing | Reference EDK II code distinguishes initial host allocation from a light path for a previously enumerated host. A bounded offline review of 336 extracted PE32 bodies from the Dell BIOS 2.18.1 distribution found allocator-related interfaces and no validated late, branch-local allocation route. The live provider identity was not verified. Re-entering the full provider sequence appeared host-scoped. |
| Stock hot-plug option | A per-port setup form exists. The recorded port did not advertise the required hot-plug state, and the inspected provider supplies 16 MiB of prefetchable padding. It gave no verified 512 MiB path. |
| Windows allocation | WDDM can renegotiate BARs on supported platforms. The case found no supported fixed-512 MiB control for this branch and stack. A post-Windows resource check is necessary for any future experiment. |

An early PCI allocation hook and a USB application launched after allocation have different opportunities. ReBarUEFI hooks the PCI resource collection phase when loaded in time. The tested late application could chainload Windows but did not trigger a new isolated allocation. The limitation applies to the tested stock firmware and route. The project left host-wide reallocation, firmware modification and an alternate custom Windows allocator untested. Linux's post-boot reallocation is independent of the Windows handoff. [UEFI PI PCI host bridge allocation](https://uefi.org/specs/PI/1.9/V5_PCI_HostBridge.html) · [EDK II PCI enumerator](https://github.com/tianocore/edk2/blob/496fd85f26d40871c97c5835bf5ed849b6eb8a70/MdeModulePkg/Bus/Pci/PciBusDxe/PciBus.c) · [Microsoft WDDM Resizable BAR](https://learn.microsoft.com/en-us/windows-hardware/drivers/display/resizable-bar-support)

## Limits and operational status

The validated Debian environment was USB-booted. The tests did not qualify installing to the production SSD, media array migration, every kernel update, gaming performance, or other Dell boards. QEMU verified boot/package and recovery behaviour with virtual storage. The GPU measurements and kernel-update smoke test were on physical hardware. The 16 GiB aperture describes address-space geometry, while the physical B580 still has 12 GiB VRAM.
