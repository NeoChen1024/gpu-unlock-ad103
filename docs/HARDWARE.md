# Supported cards and validation status

The module has two firmware profiles. One serves every GA10x card; the other
serves the two desktop AD103 parts. The profile is selected from the PCI
device ID, and each profile pins its own Booter, GSP boot image, firmware
signature section and V67 payload.

## GA102 profile

| Family | Device IDs | Status |
| --- | --- | --- |
| GA102 | `0x2200` … `0x222f` (see `kernel/gpu_unlock/main.c`) | supported, upstream path |
| GA103 | `0x2414` | supported, untested upstream |
| GA104 | `0x2480` … `0x24e0` | supported, upstream path |
| GA106 | `0x2501` … `0x2544` | supported; upstream validated on RTX 3060 12GB (`0x2504`) |

## AD103 profile

| Device | Model | Status |
| --- | --- | --- |
| `0x2702` | RTX 4080 SUPER | **validated by this fork**; see `docs/VALIDATION.md` |
| `0x2704` | RTX 4080 | profile included; the reference Windows port validated this part, not this module |
| other AD103 parts | mobile and professional | not enabled |

The AD103 profile requires the AD102 Booter, `gsp_boot_ad102.bin`,
`.fwsignature_ad10x` and the four-PLM payload. A different AD103 SKU can have
a different VBIOS; the loader parses the FWSEC descriptor from the board ROM, so
a different VBIOS is not automatically a problem, but the pinout of the payload
and the PLM/selector behaviour have to be confirmed.

## Adding a card

1. Boot with `gpu_unlock.targets=<bdf> gpu_unlock.dry_run=1` and keep the
   `inspect:` lines. The preflight writes no BAR0 register and starts no
   firmware.
2. Compare `PMC_BOOT_0`, `FEAT_PLM`, `MMU_PLM`, `FB_MB`, `SS0`/`SS1`, WPR2 and
   the FWSEC geometry with an already validated card of the same chip.
3. If the FWSEC geometry is the same family, add the PCI ID to `supported_ids`
   in `kernel/gpu_unlock/main.c`. Only widen the profile after a successful
   cold boot, payload marker, required PLM readbacks, both selector readbacks
   and the post-FLR state.

The module refuses a card that some driver has already initialised since power
on, so a failed attempt needs a power cycle before the next one.
