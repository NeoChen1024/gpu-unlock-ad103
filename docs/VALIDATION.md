# Validation: RTX 4080 SUPER (AD103)

Measured on 2026-10-24 with the code that became module version 1.1.0. This
is the configuration the AD103 support in this fork was written and checked
against.

## Hardware and software

| Item | Value |
| --- | --- |
| GPU | NVIDIA GeForce RTX 4080 SUPER, `10de:2702`, PCI `0000:0a:00.0` |
| Memory | 32760 MiB (32 GB modified card; stock is 16 GB) |
| VBIOS | `95.03.44.40.74` |
| Chip ID (PMC_BOOT_0) | `0x193000a1` |
| Display role | single display GPU (`boot_vga=1`) |
| Kernel | `7.2.11-zen1-1-zen` |
| NVIDIA driver | `615.78.08` (open kernel modules) |
| Firmware assets | pinned `610.43.03` generated locally |

## Result

The card unlocked at boot and `nvidia` bound normally afterwards. The
selectors were written and survived the PCIe FLR:

```
selector read: SS0=0x36000134 expected_unlock=0x88888888
selector write: SS0=0x88888888
selector read: SS1=0x4 expected_unlock=0x8
selector write: SS1=0x8
pre-FLR selectors: ss0=0x88888888 expected=0x88888888 ss1=0x8 expected=0x8
PCIe FLR returned: result=0
post-FLR SEC2 reset released, GSP halted, BME=0 pending=0 WPR2=1ffffe00:00000000 SS0=88888888 SS1=00000008
gpu_unlock: unlocked
```

The stock driver requires `WPR2.HI.VAL=0` at probe; the stale low word after
FLR is expected and is not treated as enabled by the module or the driver.

### Cold state (read-only preflight)

| Register / value | Reading |
| --- | --- |
| `PMC_BOOT_0` | `0x193000a1` (matches the profile) |
| GFW PLM / BOOT / CPUCTL | `0x8b8f` / `0x3ff` / `0x10` (complete, halted) |
| `FEAT_OVR_PLM` (`0x00823804`) | `0xffffff8f` |
| SS0 / SS1 | `0x36000134` / `0x4` |
| WPR2 low / high | `0x1ffffe00` / `0x00000000` |
| `FB_MB` | `0x7ff8` (32760 MiB) |
| VGA workspace | `0x1` (fallback base, FB top minus 1 MiB) |
| MMU PLM / lock | `0x45cb8f` / `0x00100000`–`0x7cf7ff0` (not the FB top; ignored) |
| FWSEC image | `image=0x10500`, `imem=0xf700`, `dmem=0xd80` |
| FWSEC transfer | `0xe00` (rounded to 256-byte packets) |
| FWSEC mapper | `interface=0x1c`, `pkc=0xb24`, `engine=0x400`, `ucode=9`, `signatures=2`, `versions=0x3`, `fuse_version=1` |

### Payload verification

```
AD103 PLM: WPR_CFG=0xfffff0ff expected=0xfffff0ff matched=1 required=1
AD103 PLM: FBPA=0xffffff0f expected=0xffffffff matched=0 required=0
AD103 PLM: WPR=0xffffffff expected=0xffffffff matched=1 required=1
AD103 PLM: FEAT=0xffffffff expected=0xffffffff matched=1 required=1
AD103 boot result: marker=0x67b00001 debug=0xda550000 cpuctl=0x10 halted=1 plm_required_ok=1
```

The FBPA write is rejected by the hardware in this configuration. The
reference AD103 port observed the same value, so the module logs it and
continues; the other three PLMs, the completion marker, the debug value and
the SEC2 halt are required.

## Performance

`mmapeak` with the default 3.0 s target per kernel, one run on each boot: the
"before" run was a boot with only dry-run enabled (selectors untouched), the
"after" run was the unlocked boot. Clocks were not locked; power limit was the
default 320 W. Raw outputs are in `docs/benchmarks/`.

| Kernel | Before (TFLOPS) | After (TFLOPS) | Ratio |
| --- | ---: | ---: | ---: |
| `mma_f16f16f32_16_16_16` | 101.5 | 198.2 | 1.95x |
| `mma_f16f16f32_32_8_16` | 101.5 | 202.7 | 2.00x |
| `mma_bf16bf16f32_16_16_16` | 101.2 | 198.1 | 1.96x |
| `mma_bf16bf16f32_32_8_16` | 101.2 | 198.4 | 1.96x |
| `mma_tf32tf32f32_16_16_8` | 50.6 | 99.2 | 1.96x |
| `mma_f8f8f32_16_8_32` | 201.8 | 376.1 | 1.86x |
| `mma_f8f8f16_16_8_32` | 377.4 | 377.6 | 1.00x |
| `mma_f16f16f16_16_16_16` | 202.0 | 200.2 | 1.00x |
| `mma_s8s8s32_16_16_16` | 390.8 | 394.0 | 1.01x |
| `mma_s4s4s32_8_8_32` | 659.5 | 652.3 | 0.99x |
| `fma_fp32` (scalar) | 40.3 | 40.5 | 1.01x |
| `fma_fp64` (scalar) | 0.7 | 0.7 | 1.00x |

Only FP32-accumulate MMA, TF32 and FP8 with FP32 accumulation change.
FP16-accumulate MMA was already at the full rate, and integer, scalar and
FP64 throughput are unaffected, which is the expected selector-only effect.

## Reproducing

1. Build and install from this repository, then add the opt-in to the kernel
   command line:

   ```
   gpu_unlock.targets=0000:0a:00.0
   ```

2. Before the first real unlock, use the read-only preflight and compare the
   `inspect:` lines with the table above:

   ```
   gpu_unlock.targets=0000:0a:00.0 gpu_unlock.dry_run=1
   ```

3. Remove `gpu_unlock.dry_run=1` and reboot.

The card must be cold (no driver has used it since power on). If an unlock
fails after firmware has started, power off before trying again rather than
issuing a warm reboot.

## Caveats

- One card, one VBIOS and one driver branch. `0x2704` (RTX 4080) uses the same
  profile but has not been tested with this module.
- The benchmark is a single run per boot, not a median; small differences in
  the unchanged rows are run-to-run noise.
- The tested card has 32 GB. The memory modification should not affect the
  compute selectors, but it is part of the measured configuration.
- Suspend/resume re-unlocking has not been validated on AD103 yet.
- The module was unsigned/enrolled-MOK-less in this test, so the kernel was
  tainted. Secure Boot users need to enroll the DKMS MOK key.
