# Adding B0 (NSIB) and MCUboot to an nRF54LM20A Project

## Introduction

This article walks through building a two-stage secure boot chain on the nRF54LM20A: NSIB (Nordic's Secure Immutable Bootloader, also called B0) sitting in front of MCUboot. The starting point is a blank project — neither bootloader existed beforehand. Both get set up together, with B0 as the immutable root of trust and MCUboot as the upgradable second stage underneath it.

The end state is a three-stage chain: B0 verifies and starts MCUboot (from S0 or S1, explained below), and MCUboot verifies and starts the application.

This covers the partition layout, sysbuild configuration, Kconfig, image signing, and key provisioning needed to get there. The example project keeps the application's secondary slot in internal flash (no external flash) to keep the layout simple, and ships with mcumgr enabled over UART so MCUboot and application updates can be demonstrated without extra hardware. A companion article covers adding BLE as a second transport for updating MCUboot's S0/S1 slots specifically.

---

## What B0 (NSIB) and MCUboot Actually Do

NSIB, also called B0, is a first-stage bootloader built to be an immutable root of trust that never gets updated in the field. Like any bootloader, it verifies and boots the next stage by checking that image's signature against public key hashes provisioned onto the device separately from the NSIB binary itself. On the nRF54L Series, that key lives in the hardware Key Management Unit (KMU) rather than being compiled into the NSIB image.

NSIB has two slots for storing the next stage, S0 and S1. Updating that next stage means transferring a signed S0 or signed S1 image onto the device — never both at once, and more on why below.

MCUboot is the second-stage bootloader. It stores and verifies a primary and a secondary application image. The primary app is what runs after successful verification; the secondary stores the update image. Updating the application goes through MCUboot's swap mechanism, the same as in any MCUboot-only setup.

The reason for stacking both: it makes MCUboot itself upgradeable. A new MCUboot build gets written into whichever of S0/S1 isn't currently active. NSIB stays completely immutable, MCUboot can be upgraded to a newer version, and MCUboot keeps providing the update mechanism for the application underneath it. The result is an upgradeable second-stage bootloader sitting on top of an immutable root of trust.

This also matters because NSIB on its own doesn't support firmware updates over SMP/mcumgr transport at all — not for itself, not for an application running directly under it. If BLE or UART-based DFU is a requirement anywhere in the chain, MCUboot has to be there regardless of whether B0 gets added.

---

## Boot Architecture on the nRF54L

On power-up, B0 runs first. It checks the public key hash stored in the KMU against whichever of S0 or S1 currently has the higher firmware version, verifies that image's signature, and boots it. That image is MCUboot. MCUboot then runs its own verification against the application's primary slot — using its own embedded or KMU-stored key — and either boots the primary app directly or performs the primary/secondary swap if an update is pending, exactly as it would without B0 in the picture at all.

Two things are specific to the nRF54L Series compared to older nRF52/53/91 parts:

- **Key storage uses the KMU, not UICR/OTP.** Older parts provision public key hashes into a one-time-programmable UICR region. The nRF54L Series instead stores key material in dedicated KMU slots backed by RRAM, accessed through the CRACEN security subsystem. This changes how provisioning works in practice (see Key Provisioning below).
- **B0 picks between S0 and S1 by firmware version, using a monotonic counter.** Whichever of the two has the higher `CONFIG_FW_INFO_FIRMWARE_VERSION` value wins at boot. This is what makes MCUboot field-upgradable even though B0 itself is not.

Underneath MCUboot, the existing primary/secondary slot mechanism for the application is unaffected. B0 only ever looks at S0/S1, never at the application slots.

---

## Project Setup and Sysbuild Configuration

Enabling B0 as a child image is a sysbuild-level decision. In `sysbuild.conf`:

```ini
SB_CONFIG_SECURE_BOOT_APPCORE=y
SB_CONFIG_SECURE_BOOT_SIGNING_KEY_FILE="path to private key"

SB_CONFIG_BOOTLOADER_MCUBOOT=y
SB_CONFIG_BOOT_SIGNATURE_KEY_FILE="path to private key"

# key provisioning needs to be done on the first flash, since B0 fails
# validating keys if this is not enabled on first flash
SB_CONFIG_SECURE_BOOT_GENERATE_DEFAULT_KMU_KEYFILE=y
```

`SB_CONFIG_SECURE_BOOT_APPCORE` pulls B0 in as a sysbuild child image for the application core, the same way MCUboot gets pulled in via `SB_CONFIG_BOOTLOADER_MCUBOOT`. *[added]* Enabling it also takes care of building the S1 variant of that image automatically — it's not something you need to ask for separately.

`SB_CONFIG_SECURE_BOOT_SIGNING_KEY_FILE` (B0) and `SB_CONFIG_BOOT_SIGNATURE_KEY_FILE` (MCUboot) point at the actual signing keys. In this example both B0 and MCUboot are signed with the same key (`ed25519_2_private.pem`), though they don't have to be. Each bootloader is built with its own signature key, so using different keys for B0 and MCUboot won't break anything as long as the key type is supported by both.

Worth being precise about: `SB_CONFIG_SECURE_BOOT_GENERATE_DEFAULT_KMU_KEYFILE` does not mean "use a throwaway Nordic-generated key." Combined with `SB_CONFIG_SECURE_BOOT_SIGNING_KEY_FILE` pointing at a real key, it means the build system automatically generates and provisions the KMU keyfile for that key as part of flashing. Without it, the key still gets used for signing, but provisioning into the KMU has to be done as a separate manual step before the device will boot (see Key Provisioning below).

The B0 child image has its own Kconfig fragment, `sysbuild/b0.conf`:

```ini
CONFIG_IS_SECURE_BOOTLOADER=y
```

This marks the image being built as the immutable bootloader image, as opposed to a regular application image linked at the same address.

A failed attempt worth recording: running MCUboot in single-application mode (no secondary slot) by adding `SB_CONFIG_MCUBOOT_MODE_SINGLE_APP=y`. This doesn't work once B0 is in the chain. B0's partition layout still assumes MCUboot has both a primary and secondary slot to manage the application swap, and the build doesn't accommodate dropping the secondary slot. If single-slot operation is a hard requirement, resolve that before introducing B0, not after.

---

## Flash Layout and Partitions

Adding B0 means the flash layout below MCUboot has to be split into two interchangeable MCUboot images, S0 and S1, instead of one. Every address downstream of B0 shifts.

**Before**, with just MCUboot:

```
mcuboot              0x00000 – 0x14000   (80 KiB)
mcuboot_pad          0x14000 – 0x14800
mcuboot_primary_app  0x14800 – 0x1FD000  (app, ~1.95 MiB)
mcuboot_secondary    same size as mcuboot_primary
```

**After**, with B0 added. This is the actual `pm_static.yml` from the worked example:

```yaml
b0_container:
  address: 0x0
  region: flash_primary
  size: 0x9000
  span: [b0]
b0:
  address: 0x0
  region: flash_primary
  size: 0x9000

s0:
  address: 0x9000
  region: flash_primary
  size: 0x11000
  span: [s0_pad, mcuboot]
s0_pad:
  address: 0x9000
  region: flash_primary
  size: 0x800
s0_image:
  address: 0x9800
  region: flash_primary
  size: 0x10800
  span: [mcuboot]
mcuboot:
  address: 0x9800
  region: flash_primary
  size: 0x10800

s1:
  address: 0x1a000
  region: flash_primary
  size: 0x11000
  span: [s1_pad, s1_image]
s1_pad:
  address: 0x1a000
  region: flash_primary
  size: 0x800
s1_image:
  address: 0x1a800
  region: flash_primary
  size: 0x10800

mcuboot_primary:
  address: 0x2b000
  region: flash_primary
  size: 0xdd000
  span: [mcuboot_pad, app]
mcuboot_pad:
  address: 0x2b000
  region: flash_primary
  size: 0x800
app:
  address: 0x2b800
  region: flash_primary
  size: 0xdc800
mcuboot_primary_app:
  address: 0x2b800
  region: flash_primary
  size: 0xdc800
  span: [app]

mcuboot_secondary:
  address: 0x108000
  region: flash_primary
  size: 0xdd000

bootconf:
  address: 0xffd080
  region: bootconf
  size: 0x4
```

You can copy the syntax from this example (`span: [app]`) or type out addresses and sizes manually. Just make sure the addresses match and there's no overlap or gaps.

What changed, address by address:

- `b0` occupies the first 36 KiB (`0x9000`) of internal flash and is the only partition that's truly fixed. Everything else is positioned relative to it. (36 KiB rather than a round 32 KiB here. Size it to whatever your actual B0 build needs, check the build output for the real image size, and pad up to an erase-block boundary.)
- `s0` and `s1` are each `0x11000` (68 KiB): a `0x800` pad plus a `0x10800` (66 KiB) MCUboot image region. S0 and S1 must be the same size, pad and image — a mismatch here is a build error.
- `mcuboot` itself is defined at `0x9800`, inside `s0`. There's no separate `mcuboot` partition definition for S1 in this static file. The S1 variant is generated automatically as `s1_image` from the same MCUboot build, just relocated, when `SB_CONFIG_SECURE_BOOT_APPCORE` is enabled.
- `app` shrank to make room for B0 and the second MCUboot copy. B0 (36 KiB) plus the second `s1` copy of MCUboot (68 KiB) is real flash the application no longer has, on top of the application slot's own pad and headers.
- `mcuboot_secondary` stays the same size as `mcuboot_primary`. That invariant doesn't change just because B0 is now in front of everything.

---

## Devicetree Partition Labels

If the board already has a devicetree overlay with hand-written partition nodes (a custom board target, for instance), those nodes need to stay in sync with whatever `pm_static.yml` says. In the worked example here, on the standard `nrf54lm20dk` board target, this step wasn't needed at all — there was no pre-existing custom partition overlay to conflict with, so Partition Manager's static layout was sufficient on its own.

If you do have an overlay defining partitions, it needs to mirror `pm_static.yml` exactly, address for address. Example:

```dts
boot_partition: partition@0 {
    label = "b0";
    reg = <0x0 0x8000>;
};

s0_partition: partition@8800 {
    label = "s0_image";
    reg = <0x8800 0x14000>;
};

s1_partition: partition@1d000 {
    label = "s1_image";
    reg = <0x1d000 0x14000>;
};

slot0_partition: partition@31000 {
    label = "image-0";
    reg = <0x31000 0x1cc000>;
};
```

(These addresses are from a different, larger layout than the worked example above, shown to illustrate the shape of the override. Match yours to your own `pm_static.yml`.) Get the devicetree partition nodes out of sync with the static layout file, and the build will link images against one address while B0/MCUboot look for them at another. See the debugging notes below for what that failure actually looks like.

---

## Key Provisioning

NSIB's job is to verify the next stage, and that depends entirely on a public key hash that has to already be sitting on the device before B0 ever runs. On the nRF54L Series this hash lives in the KMU, not in OTP/UICR, and is provisioned through a different path than on older nRF52/53/91-class parts.

In this example, `SB_CONFIG_SECURE_BOOT_GENERATE_DEFAULT_KMU_KEYFILE=y` automates that. The build system generates a `keyfile.json` derived from the signing key set in `SB_CONFIG_SECURE_BOOT_SIGNING_KEY_FILE`, and that gets pushed into the KMU automatically the next time the device is flashed with `west flash --recover`. This is convenient for bench work, and it's also the realistic path once a real, project-specific key is used in `SB_CONFIG_SECURE_BOOT_SIGNING_KEY_FILE` rather than a generated one. The word "default" in the Kconfig name refers to the automatic-provisioning-during-flash mechanism, not to the key material itself.

For the manual equivalent, or to understand what that automation is doing under the hood, Nordic's KMU provisioning guide is the reference. The short version, manually:

```bash
west ncs-provision upload -k <public_key>.pem \
    --keyname BL_PUBKEY
```

followed by `west flash --recover`. A few things that matter in practice:

- Provisioning has to happen on the first flash. Skip it and B0 will fail to validate the next stage and refuse to boot.
- If the key changes after the device has already been provisioned once, it has to be re-provisioned. The old key's hash doesn't automatically get replaced; you flash a new keyfile/hash for the new key.
- `west flash --recover` is the flag combination that works for a project containing NSIB. The programming file carries UICR/KMU provisioning data alongside the application image, and a plain `--erase` alone doesn't reliably get a clean re-provisioning when keys are involved.

---

## Build and Flash Process

The build and flash process doesn't change shape once B0 is added:

```bash
west build -p -b nrf54lm20dk/nrf54lm20a/cpuapp
west flash --recover
```

One MCUboot-side setting worth flagging from `sysbuild/mcuboot.conf`:

```ini
CONFIG_BOOT_MAX_IMG_SECTORS=256
```

This caps how many flash sectors MCUboot's swap algorithm tracks per image. If the application slot is sized up and this isn't increased to match, MCUboot will fail to manage the full slot.

---

## Updating MCUboot's S0/S1 Slots

This is the part B0 actually changes the shape of: updating MCUboot now means uploading a new S0 or S1 image through mcumgr/SMP, not flashing over SWD/JTAG.

The mechanism itself is transport-agnostic — the same upload-confirm-reset flow works whether mcumgr is talking over UART or BLE. What matters more than the transport is picking the right image: you always upload whichever of S0/S1 is *not* currently active, and that has to be checked fresh every time, since active and inactive flip with each successful update. Upload an S0 image while S0 is already running, and nothing useful happens.

### Enabling mcumgr over UART

`prj.conf`:

```ini
CONFIG_MCUMGR=y
CONFIG_MCUMGR_GRP_OS=y
CONFIG_MCUMGR_GRP_IMG=y

# UART transport
CONFIG_MCUMGR_TRANSPORT_UART=y

# dependencies for MCUMGR / IMG_MANAGER
CONFIG_NET_BUF=y
CONFIG_ZCBOR=y
CONFIG_CRC=y
CONFIG_FLASH=y
CONFIG_IMG_MANAGER=y
CONFIG_STREAM_FLASH=y
CONFIG_FLASH_MAP=y
CONFIG_BASE64=y

# needed to read the monotonic counter from the application
CONFIG_SECURE_BOOT_STORAGE=y
```

`CONFIG_MCUMGR_TRANSPORT_UART=y` exposes the SMP server directly over the console UART, so the `mcumgr` CLI can talk to it with no extra application-side code. `CONFIG_SECURE_BOOT_STORAGE` is included specifically so application code can later read the monotonic counter B0 maintains — that's the hook for any logic that needs to know which slot, S0 or S1, is currently active (see the open item at the end of this article).

These same dependencies, minus the transport line, are also what the BLE setup in the companion article builds on top of.

### Actually performing the update

A GUI tool like AuTerm (https://github.com/thedjnK/AuTerm/) makes this easier than the raw CLI, though the CLI works fine for scripting.

Once the project is built, there will be two files: `signed_by_mcuboot_and_b0_s1_image.bin` and `signed_by_mcuboot_and_b0_mcuboot.bin`. The `*_b0_mcuboot.bin` file is the image for S0; `*_s1_image.bin` is the image for S1. Both images carry metadata about where they're meant to live, so one can't be substituted for the other — upload whichever one corresponds to the slot that's *not* currently booted.

```bash
mcumgr --conntype serial --connstring dev=/dev/ttyACM0,baud=115200 image upload build/<s0 or s1 signed image>
```

followed by:

```bash
mcumgr --conntype serial --connstring dev=/dev/ttyACM0,baud=115200 image confirm
mcumgr --conntype serial --connstring dev=/dev/ttyACM0,baud=115200 image reset
```

The same commands work over BLE once the BLE transport and advertising are wired up — that's covered in full in the companion article, **Adding BLE Support for B0/MCUboot S0 and S1 Updates**.

---

## Common Issues and Debugging Notes

**B0 fails to start with a verification-related error on first boot.** This almost always traces back to key provisioning, not the image itself. If a build using `SB_CONFIG_SECURE_BOOT_GENERATE_DEFAULT_KMU_KEYFILE=y` boots fine but a hand-provisioned key doesn't, check the provisioning step first.

**MCUboot updates fine the first time, then fails on the second update.** If the application boots, and the first MCUboot upgrade (uploading, say, S1 while S0 is active) works, but a second upgrade attempt fails, check `pm_static.yml` for stale or incorrect addresses before anything else. If `EMPTY_0`-style placeholder partitions show up in the generated layout, that's a sign the static layout isn't fully pinned down. Delete them and make sure B0 explicitly starts at `0x0` rather than being left to float. If a devicetree overlay is involved, check there too.

**Trying single-slot MCUboot with B0 enabled doesn't build or doesn't behave as expected.** Already covered above. `SB_CONFIG_MCUBOOT_MODE_SINGLE_APP` and B0 don't currently coexist cleanly in this setup. Don't spend time debugging this combination; figure out which one is actually required for the project before going further.

**Partition size mismatches between S0 and S1 fail the build outright.**

---

## Final Result and Recommendations

The chain works end to end on the bench. B0 verifies and boots whichever of S0/S1 has the higher firmware version, that MCUboot instance verifies and boots the application, and the application's own primary/secondary DFU flow is unaffected by any of this. MCUboot itself can be updated over UART (or BLE — see the companion article) using ordinary mcumgr image-upload commands, pointed at an S0 or S1 build instead of an application build.

**Open item, unsolved here:** there is currently no automatic way for the device, or for whatever is driving the OTA flow, to know which of S0/S1 is active and therefore which one is the valid upload target. Uploading an S0 image while S0 is already running does nothing useful. The early shape of a fix is visible in this project's `main.c`, a guarded block reading the active slot via `PM_S0_ADDRESS`/`PM_S1_ADDRESS` and a `read_s0_active()` helper. The intended approach is to use `CONFIG_SECURE_BOOT_STORAGE` to read B0's monotonic counter / active-slot state at runtime, and use that to decide which image variant to request or accept over OTA. *[added]* Worth knowing going in: the counter backing that S0/S1 selection has a finite number of update slots (`CONFIG_SB_NUM_VER_COUNTER_SLOTS`), which is a constraint on how this gets designed, not something to dig into here since it's not implemented yet. That work isn't done and is the next real piece of this project, not something this article can claim is solved.

**Recommendations for repeating this elsewhere:**

- Decide whether single-slot MCUboot is a hard requirement before adding B0. The two didn't coexist in this setup, and that's a layout decision, not a quick fix.
- Treat the static partition file and any custom devicetree partition overlay as one document. Most of the debugging time in this kind of change goes into the two drifting out of sync with each other.
- Don't ship `SB_CONFIG_SECURE_BOOT_GENERATE_DEFAULT_KMU_KEYFILE=y` pointed at a key that isn't actually yours.
- Get S0/S1 active-slot detection working before building any unattended OTA pipeline around MCUboot updates. Right now that decision has to be made by a human who knows which slot is currently booted.

---

## References

- nRF Secure Immutable Bootloader (NSIB) overview, Nordic Connect SDK documentation, `nrf/samples/bootloader`
- Secure bootloader chain (NSIB + MCUboot), Nordic Connect SDK documentation, `nrf/app_dev/bootloaders_dfu/mcuboot_nsib`
- Enabling a bootloader chain using sysbuild, Nordic Connect SDK documentation, `nrf/app_dev/bootloaders_dfu/mcuboot_nsib/bootloader_adding_sysbuild`
- Performing KMU provisioning (nRF54L), Nordic Connect SDK documentation, `nrf/app_dev/device_guides/nrf54l/kmu_provision`
- Nordic Developer Academy, "Bootloaders and DFU / FOTA" lesson, exercises on DFU over UART/BLE and custom signing keys
- Companion article: **Adding BLE Support for B0/MCUboot S0 and S1 Updates**
