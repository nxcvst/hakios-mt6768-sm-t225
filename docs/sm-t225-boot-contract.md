# SM-T225 offline boot contract and memory map

This is an offline description of what is known from this repository and the supplied Linux DTS at `../gta7lite.dts`. The DTS identifies the model as `MT8768WT` and the root compatible as `mediatek,MT6765`; it is useful hardware documentation, but it is not an Apple Device Tree and does not prove what Samsung's production boot chain leaves active. The JSON alongside this document is the machine-readable inventory of addresses declared by that DTS.

## Existing donor boot path

The current repository is for the Infinix Hot 20 X6826B. Its execution path is:

1. The Android bootloader loads an Android boot image that has been repacked by `tools/chainboot.py`. The tool expects an Android boot image v2 and an arm64 Linux `Image`; it prepends the bare-metal loader, pads to its fixed chain offset, and retains the original Android payload and boot metadata. This establishes the donor's packaging assumption, not compatibility with Samsung's boot image format or bootloader policy.
2. The loader's Linux arm64 Image entry is `_start` in `mtkloader/start.S`. Its source comments and code assume the loader is entered at `0x40080000` by LK, with x0 pointing to LK's DTB and MMU/data/instruction caches off. `_start` masks exceptions, installs a local exception vector at the current exception level, creates a stack just below the image, clears BSS, and calls `mtkloader_main`. This address and incoming state are donor assumptions; the SM-T225 loader entry address and state are **unknown**.
3. The loader can either return to the Android Image through its chain stub or continue to XNU. Its C code handles donor-specific platform setup, image loading, device-tree generation, boot arguments, and handoff. It assumes the vendor boot stage has already initialized several devices (notably the eMMC controller/card state used by the current loader), while it performs additional donor setup itself. Which state Samsung BL/LK/ABL provides, and which MT8768T setup would be needed, is **unknown**.
4. For XNU, `run_xnu` reads the payload's Mach-O, lays out segments using the configured slide, obtains the entry from its Unix thread state, constructs the custom Apple-format flattened device tree and `boot_args`, then calls `jump_to_kernel(entry_pa, ba_pa)`. `mtkloader/start.S` passes the boot-args physical pointer in x0 and branches to the physical entry. From EL2 it sets HCR_EL2.RW, prepares an EL1h return with DAIF masked, and `eret`s. From EL1 it disables the MMU and data/instruction caches before branching. It does not establish a target-specific EL2, timer, GIC, PSCI, or clock contract for this tablet.

### Existing XNU handoff requirements versus donor assumptions

| Handoff item | Verified in existing code | SM-T225 status |
| --- | --- | --- |
| CPU architecture | AArch64 XNU entry; physical entry address derived from Mach-O entry and loader slide | ISA family is expected from target platform, but boot image/EL handoff must be checked on device |
| Entry registers | x0 is the physical pointer to the custom `boot_args`; branch/ERET goes to physical entry | Must retain the XNU boot-args ABI; target entry and kernel placement unassigned |
| Execution level/state | Loader supports entry to XNU from EL2 or EL1 and masks DAIF; EL1 path disables SCTLR.M/C/I | Incoming level and required translation/cache state for Samsung boot stages are unverified |
| Memory ABI | `boot_args` includes `virtBase`, `physBase`, `memSize`, `topOfKernelData`, video fields, device-tree pointer/length and command line | Target values and RAM available to XNU are unassigned; Linux's `memory.reg` is not enough to define free RAM |
| Apple Device Tree | Loader emits a custom Apple flattened tree, and `boot_args.deviceTreeP` is set to its slid address | Target tree schema/content and a safe physical placement are not established; Linux DTS cannot be passed through as an Apple tree |
| Boot arguments | Donor defaults include `rd=md0 emmc_root=1`, CPU count, and loader return timing; build flags can extend the command line | No SM-T225 arguments are assigned. Root and userspace are outside this milestone |
| CPU topology | Donor CPU ordering, core count and PSCI CPU_ON MPIDR assumptions are encoded in the loader | DTS declares eight Cortex-A53 CPUs with MPIDRs `0..3` and `0x100..0x103`, PSCI, and `method = "smc"`; runtime PSCI behavior and MPIDR affinity values still require confirmation |
| Interrupts/timer | Donor loader has a GIC-related XNU patch/initialization path and timer assumptions | Linux DTS declares GICv3 ranges and the architectural timer, but distributor/redistributor state, security groups, routing, timer frequency and EL access at XNU entry remain unverified |
| Clocks and platform initialization | Loader has donor-specific PMIC/clock/touch and subsystem setup; some subsystems are gated/configured in code | No target initialization sequence is inferred from the Linux DTS; must be independently verified |

The XNU ABI-level obligations above are requirements of the existing implementation. Hard-coded load addresses, physical/virtual bases, core ordering, CPU counts, device identities, GIC workarounds, clocks, panel/framebuffer handling, and watchdog/reset behavior are implementation-specific assumptions and cannot be carried over without target evidence.

## SM-T225 physical memory evidence

### Declared RAM window

The supplied Linux DTS `memory.reg` is `<0 0x40000000 0 0xc0000000>`, which denotes physical RAM `[0x40000000, 0x100000000)` (3 GiB). This is the best current target range, but it is only a DTS declaration. The Android kernel's final memory map, bootloader carveouts, secure-world assignments, and the actual bank layout must be checked before XNU claims any portion as available.

### Fixed reservations

The complete fixed reservation inventory, including each DTS `no-map` property, is in `configs/sm-t225-memory-map.json`. Its important classes are:

- Secure/firmware-labelled regions: GZ, TEE, ATF, SSPM and DRAM-controller reservations, plus secure logging/diagnostic ranges.
- Boot/diagnostic working areas: preloader arguments, LK ramdump, DTB ramdump, RAM console, pstore, mini dump and log storage.
- A framebuffer reservation at `0x79da0000` of `0xe60000` bytes. Its presence in the DTS does not prove that it remains the active framebuffer or that XNU may reuse it.

All fixed ranges fall within the DTS RAM window and the static intervals do not overlap each other. This only validates arithmetic over the DTS declarations. In the absence of a verified handoff snapshot, treat every listed range as unavailable to the loader, kernel, Apple Device Tree and boot arguments. `no-map` describes the Linux reservation's mapping policy; it does not tell us whether Samsung firmware has released or repurposed that memory by the time this loader runs.

### Dynamic reservations

The DTS also has allocation policies, rather than physical base addresses, for connectivity, ION, SCP shared memory, SSPM shared memory, SSMR/CMA and Wi-Fi. Their allocation windows and requested sizes are recorded in the JSON. No exact physical range can be assigned from these policies alone. In particular, `ssmr-reserved-cma_memory` declares a size of `0x18000000` (384 MiB) but an allocation window of `0x10000000` (256 MiB); a standard single-window allocation cannot fit. This may reflect vendor-specific semantics or an inconsistent dump. It is unresolved and is not converted into a physical carveout here. The movable-CMA node has no size or address.

### XNU and loader memory assignments

No target physical address is presently assigned for the loader, XNU kernel, its reserved growth/data area, small ramdisk, Apple Device Tree, or `boot_args`. The donor's loader entry at `0x40080000`, XNU physical base `0xc0000000`, system image base offset, memory size, and 16 KiB placement alignment cannot be adopted: they conflict with or lack proof against the target's reservations and actual boot packaging. The machine-readable configuration intentionally represents these fields as `null`.

There is consequently no defensible “free RAM” total yet. Computing one requires knowing which dynamic allocations are instantiated, the bootloader's final reserved-memory/usable-memory description, and the memory occupied by the Android image and loader at handoff. A guessed XNU range could overwrite secure or live firmware memory.

## Target-specific facts from the Linux DTS

The DTS declares eight Cortex-A53 CPUs, `enable-method = "psci"`, and a PSCI node with `method = "smc"`. It declares `arm,gic-v3` with register windows `0x0c000000+0x40000`, `0x0c100000+0x200000`, and a MediaTek sysirq window at `0x10200a80+0x50`; it also declares `arm,armv8-timer`. These are useful candidate interfaces for XNU bring-up. They do not establish that secure firmware services are available to XNU, which SMC functions are implemented, whether the GIC is left enabled/configured, or the timer's effective frequency/access permissions.

The supplied DTS identifies MediaTek MT8768WT in its model string while its root compatible says MT6765. The user identifies the platform family as MT8768T/MT8768WT. This mismatch is preserved as evidence, not resolved by guessing a more specific silicon identity.

## Static validation and limits

The address ranges and boundary arithmetic in the JSON can be checked offline. No loader/kernel source has been changed, so no compiled component changes as part of this record. The donor loader build cannot be reproduced from this standalone checkout: `mtkloader/build.sh` expects `xnuloader/identity.h` and `xnu-deps/host/zig/zig`, while the repository instructions describe the shared XNU tree and an IPSW-derived identity header as external prerequisites. No build script, package installer, platform-tools utility, or device interface is needed for this static milestone.

This work does not claim that the target boot contract has been proven on hardware. In particular, it does not authorize or perform a reboot, flash, partition operation, or bootloader change.

## First-XNU-handoff blockers

The first handoff remains blocked until there is target evidence for: (1) boot image format and loader entry address/register/EL/cache state; (2) final physical memory banks and live reserved ranges, including dynamic MediaTek allocations and secure-world/ATF/TEE ownership; (3) a target-safe loader/kernel/RAM-disk/Apple-DT/boot-args placement plan; (4) PSCI conduit/function support and CPU affinity identifiers; and (5) GIC/timer state and any required target-specific initialization. The platform-tools directory in the workspace was not used; any later inspection of it requires asking the user first.

## Next milestone

The smallest next milestone is to verify the Samsung boot-stage handoff and final memory reservations from trusted target documentation or user-approved read-only device evidence, then fill only the currently unresolved boot and placement fields. Keep loader changes deferred until those addresses and entry conditions are evidence-backed.
