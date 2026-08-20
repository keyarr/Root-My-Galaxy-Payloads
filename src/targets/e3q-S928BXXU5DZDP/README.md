# e3q-S928BXXU5DZDP compatibility status

This directory contains the hardware-tested SM-S928B (S928BXXU5DZDP) target.

`target.h` uses offsets recovered from the exact S928BXXU5DZDP ELF/BTF; the
DZF2 offsets were checked on this firmware and no offset differs.
`p0_fingerprint.h` in this directory was **regenerated from the S928B raw
Image** (`tools/generate_p0_fingerprint.pl`, probe `0x1f0000`, 256/256 source
qwords read back): the DZF2 header's row order was mirrored relative to the
S928B image, so the sister target's table must not be reused.

Input provenance, symbol audit, ABL handoff proof, BTF identity, and the
layout compatibility decision are recorded in:

```text
docs/SM-S928B-S928BXXU5DZDP.md
```

The sources pass host-Clang syntax checks. The Android release payload was
built twice with NDK r29 and reproduced byte-for-byte; its hash and the
matching KernelSU late-load artifacts are recorded in the target porting
record. The payload and KernelSU artifacts were then tested on hardware. The
exact S928B build loaded the module, entered `u:r:ksu:s0`, and remained stable
without a reboot.