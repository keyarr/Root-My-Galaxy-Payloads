# SM-S928B / S928BXXU5DZDP porting record

This file records the evidence-driven port of the Galaxy S24 Ultra
international firmware. Values from another device or firmware must not be
reused. Each stage is added only after its inputs and results have been
verified.

## Stage 1: freeze and verify the input evidence

Status: **COMPLETE**

### Firmware identity

| Field | Verified value |
| --- | --- |
| Package model | `SM-S928B` |
| PDA/AP package | `S928BXXU5DZDP` |
| Android build ID | `BP4A.251205.006` |
| Build fingerprint | `samsung/e3qxxx/e3q:16/BP4A.251205.006/S928BXXU5DZDP:user/release-keys` |
| Device codename | `e3q` |
| Product name | `e3qxxx` |
| Kernel banner | `6.1.145-android14-11-33419968-abS928BXXU5DZDP` |

### Verified AP/kernel chain

| Object | Size (bytes) | SHA-256 |
| --- | ---: | --- |
| Decompressed `boot.img` | 100,663,296 | `4638165368c37b11b3efadff84cac7aef3cf1ea37a44b56708cea91ce656e259` |
| Decompressed `recovery.img` | 110,034,944 | `0ad78d7e2969c8b5668a800441da70f2085de352b6dc5da5f5ba96377ea303fc` |
| Raw kernel payload, `boot.img` | 38,005,248 | `ac6e652bc27af83b3327a274b7f5586865fd8da7efa6f830d6389228c111aa96` |
| Raw kernel payload, `recovery.img` | 38,005,248 | identical to boot |
| Recovered `vmlinux.elf` | 43,070,883 | `776912ae01d96df28f9be97a7bb7312def932af6bb27d74ee623f3729cd3b678` |
| Extracted `vmlinux.btf` | 5,981,643 | `8415104c012e18942b18bcb52f401075cb6b92df837b9552a8c11070d65efe56` |

The `boot.img` and `recovery.img` kernels are byte-identical (same SHA). The
Image header reports `text_offset = 0x0`, `image_size = 0x26f0000`,
`flags = 0xa`, `magic = ARMd`, and the banner confirms the DZDP release
string. `vmlinux.elf` was recovered from the raw payload with
`vmlinux-to-elf`; BTF was cut from the same `vmlinux.elf` binary.

## Stage 2: BTF identity

Status: **COMPLETE**

The S928B `vmlinux.btf` SHA-256 equals the S928U1 DZF2 published BTF hash
recorded in `docs/SM-S928U1-S928U1UES6DZF2.md` (byte-identical data), so all
structure layouts in `target.h` are guaranteed on this build without
re-derivation.

The bootloader image examined for the handoff part was the decompressed
`abl.elf` (ELF32 ARM container, 2,441,528 bytes, SHA-256
`b6aa3f4382ee6b6e12e67f5adeb635e511a3b08ac94c1f95778cb3031c6c4ab2`).

## ABL kernel handoff analysis

Status: **COMPLETE**

The `abl.elf` (ELF32 ARM) contains a vendor firmware volume at offset `0x1000`
whose FFS decodes the `ee4e5898-3914-4259-9d6e-dc7bd79403cf` volume; the
stripped `LinuxLoader` PE+ (FFS GUID `f536d559-459f-48fa-8bbc-43b554ecae8d`)
carries the load-address constants for this build:

```text
0x00080000  AArch64 kernel text offset (kernel header text_offset for this
            build is 0x0, so the ABL adds the fixed 0x80000 to the DT)
0x05600000  AArch64 kernel region size
0x00800000 / 0x03c00000  ARM32 alternative pair (not used)
```

The same reserved-first-region anchor appears in all 22 `vendor_boot` DTBs
(`gunyah_hyp_region@80000000`), establishing the lowest RAM base:

```c
#define P0_PHYS_OFFSET      0x80000000ULL
#define P0_KERNEL_PHYS_LOAD 0x80080000ULL
```

The constant cluster seen in the EL2 boot image for S928B `abl` matches the
S928U1 documented derivation: kernel is loaded at `0x80080000`
(`0x80000000 + 0x00080000 + text_offset(0)`).

## Slide constants and P0 fingerprint

The S928B raw Image was scanned with `tools/generate_p0_fingerprint.pl` at
probe `0x1f0000`:

```text
verified 32 rows and 256 source qwords at probe 0x1f0000
```

Important: the raw S928B slide pattern is the SAME 256-qword table as the
S928U1 DZF2 image but in mirrored row order:

| Image | table row at probe `0x1f0000` |
| --- | --- |
| S928U1 (DZF2) | row `slide` |
| S928B (DZDP) | row `0x1f0000 - slide` (mirrored) |

The pipe oracle returns the winning row label, so the DZF2 header cannot be
reused. The checked-in regenerated header:

```text
src/targets/e3q-S928BXXU5DZDP/p0_fingerprint.h
SHA-256 949089878b76b80247a116d569d1961eb41e661a944e832f0033a75b8deee72b
```

Verified: with `APP_REQUIRE_FRESH_P0_SESSION 1` and
`APP_P0_FINGERPRINT_INVERSE_SLIDE 1` set in `target.h`, the preprocessed
`slide_app.c` contains `p0 fingerprint inverse source_offset` (the
`P0_ORACLE_PROBE_OFFSET - source_offset` slide math) and the fresh-page
`reclaim miss fresh=%d/%d` path — the flat non-fresh fallback is not
compiled.

The S928B slide constants (identical to DZF2 after re-derivation):

```c
#define SLIDE_TRACEFS_EVENT_ID 106
#define SLIDE_TRACEFS_WORKER_CALLER_OFF 0x000db1a0ULL
#define SLIDE_PSELECT_WORD_SHIFT 3
```

Image qword proofs for the log/bootid slide anchors:

```text
Image[0x016a61b8] = 0x6e696c74656e666e   ('nfnetlink' string)
nfulnl_logger object +0x00 = 0xffffffc0096a61b8  (data ptr at Image 0x016a61b8)
Image[0x023762f0] = 0xffffffc00a6046e8   (== kernel addr of sysctl_bootid)
```

## KernelSU late-load artifacts

Status: **COMPLETE for static build; hardware test pending**

- KernelSU tag `v3.2.5` commit `b0bc817b4e966aa6aa830834eaf6ef765d821d40`
  plus the Samsung KDP/RKP/DEFEX patch.
- Built in the validated DDK image `ghcr.io/ylarod/ddk-min:android14-6.1-20260313`.
- Built-in `UTS_RELEASE` (`6.1.166-dirty`) replaced with the exact target
  release before generation, so the module's vermagic line reads:

```text
vermagic=6.1.145-android14-11-33419968-abS928BXXU5DZDP SMP preempt mod_unload modversions aarch64
```

Symbol-audit report against the recovered S928B `vmlinux.elf`:

```text
undefined imports:               209
missing from target symtab:        0
symbols resolved via kallsyms:    73
undefined intentional (no CRC):  209
target CRC mismatches:             0
check_symbol exit code:           0
```

The `__versions` section is present (zero entries after strip — intentional,
per the Samsung late-load contract).

| Artifact | Size (bytes) | SHA-256 |
| --- | ---: | --- |
| `kernelsu/android14-6.1_kernelsu-e3q-S928BXXU5DZDP-kdp.ko` | 400,152 | `81ba5ba5175522437f978f79d0c5a5446d0c0054a96c446d18101bd7e3058b6d` |
| `kernelsu/ksud-e3q-S928BXXU5DZDP-kdp` | 3,799,648 | `148bf4f53fedc45cd955ecab51147242e62bee90f5ee49b0866bfa77e31577c0` |

## Payload build

Status: **COMPLETE — offline, reproducible**

```sh
make TARGET=e3q-S928BXXU5DZDP ANDROID_NDK_HOME=/path/to/android-ndk-r29 release
```

Repeated (forced) builds produce the identical artifact. The size is
truncated to the fixed `APP_RELEASE_SIZE`, so the SHA is stable across
builds:

| Artifact | Size (bytes) | SHA-256 |
| --- | ---: | --- |
| `artifacts/e3q-S928BXXU5DZDP/cve-2026-43499-app.so` | 104,128 | `85b8dea78a238bc7459a32cf374d81d831c0f446096f1f5f0cefcab2d65f95d4` |

## Completion boundary

All offline gates verified (firmware hashes, identical BTF, handoff
constants, 256-qword fingerprint regenerated from S928B, reproducible
104,128-byte release, exact-vermagic late-load module, clean import audit).
**No hardware execution has been attempted**: slide-accurate runtime offsets
and the p0 allocation path need on-device validation. Target state:
`experimental` until a device report is filed.

See also `docs/PORTING.md` and `src/targets/e3q-S928BXXU5DZDP/README.md`.
