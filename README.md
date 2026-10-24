# GPU unlock 1.1.0

Unlock FP/BF16 instruction issue rate on Nvidia GA10x and AD103 consumer cards to their professional equivalents

This is a fork of [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock)
that adds AD103 support (RTX 4080 and 4080 SUPER) and a read-only preflight.
The upstream module, packaging and GA10x behaviour are unchanged; the AD103
work is additive.

This Linux kernel module combines the CMP unlocker's V67/PLM technique with the pre-driver timing of the UEFI approach. It runs the unlock, writes the speed selectors (SS0 `0x88888888`, which selects the full BF16 rate, and the 4-bit SS1 `0x8` IMLA4) and performs FLR before the unmodified NVIDIA driver binds the card. It is neither a UEFI application nor a patched NVIDIA driver. 

The benefit of this early kmod driver is that it does not require manual patching each time nvidia releases a new driver version.

The effect of this unlock is that it doubles the instruction issuing speed of FP/BF16 FMA into FP32 operations on GA102, GA103, GA104, GA106 and AD103 cards.

## Validated hardware

RTX 4080 SUPER (`10de:2702`), 32 GB modified card, VBIOS `95.03.44.40.74`,
driver `615.78.08`. Measured with `mmapeak`, one run per boot:

| Kernel | Before (TFLOPS) | After (TFLOPS) |
| --- | ---: | ---: |
| BF16→FP32 `32_8_16` | 101.2 | 198.4 |
| FP16→FP32 `32_8_16` | 101.5 | 202.7 |
| TF32 | 50.6 | 99.2 |
| FP8→FP32 | 201.8 | 376.1 |
| FP16→FP16 | 202.5 | 201.8 |

The full register, firmware and benchmark evidence is in
[docs/VALIDATION.md](docs/VALIDATION.md); the supported IDs and their status
are in [docs/HARDWARE.md](docs/HARDWARE.md).

[3090 Ti dense 8192³ GEMM at 450 W, five-sample medians with FP32 accumulation](https://github.com/Zagitta/gpu-unlock/issues/1#issue-5905622687):


| Input | Before       | After         |
| ----- | ------------ | ------------- |
| FP16  | 87.46 TFLOPS | 118.39 TFLOPS |
| BF16  | 87.50 TFLOPS | 157.41 TFLOPS |
| TF32  | 42.58 TFLOPS | 82.61 TFLOPS  |




## How to install

Clone this repository, then from its root run:

```sh
make install
```

Then append `gpu_unlock.targets=all` to your kernel command line, keeping
existing options:

- **GRUB (Debian/Ubuntu/Mint):** add it inside `GRUB_CMDLINE_LINUX_DEFAULT`
in `/etc/default/grub`, then run `sudo update-grub`.
- **GRUB (Arch/CachyOS):** same edit, then run
`sudo grub-mkconfig -o /boot/grub/grub.cfg`.
- **Fedora/RHEL:** run
`sudo grubby --update-kernel=ALL --args="gpu_unlock.targets=all"`.
- **systemd-boot:** append it to the `options` line in your boot entry
(`bootctl status` shows its location).
- **UKI / generated entries:** add it to `/etc/kernel/cmdline` and regenerate
your boot images/entries (e.g. `sudo mkinitcpio -P` for mkinitcpio UKIs).



## Local build

Every build downloads the pinned NVIDIA/CachyOS inputs over HTTPS, checks
their SHA-256, and generates firmware only in the build directory and the
binary package. All three formats install through the same `make install-*`
targets. Packages are built from the committed `HEAD`.

Build, install build dependencies, and install the packages in one step
(asks for your sudo password):

```sh
make install        # detect the distribution and select an installer
make install-arch   # makepkg -sif
make install-deb    # apt-get build-dep, dpkg-buildpackage, apt-get install
make install-rpm    # dnf builddep, rpmbuild, dnf install
```

`make install` uses `/etc/os-release` (`ID`, then `ID_LIKE`) to detect Arch,
Debian/Ubuntu, or Fedora/RHEL/CentOS families. The RPM installer requires DNF.
For other distributions, select a compatible explicit target above.

`make arch`, `make deb` and `make rpm` only build. Debian and RPM packages go
to `build/deb` and `build/rpm`. The NVIDIA inputs are cached in
`~/.cache/gpu-unlock/inputs` (`make inputs`; override with `INPUTS=`).
`dpkg-buildpackage` itself never uses the network, so a clean-chroot build (sbuild or
pbuilder) needs `GPU_UNLOCK_INPUTS` pointing at that cache.
The `make dist` archive, the Debian source package and the `.nosrc.rpm`
contain no NVIDIA archives or generated firmware.

## Boot flow and opt-in

The packages put `gpu_unlock` and its firmware in the initramfs and load it
there, before `nvidia` can bind a card:

- **mkinitcpio:** `/etc/mkinitcpio.conf.d/zz-gpu-unlock.conf` inserts
`gpu_unlock` right before `nvidia` in `MODULES`, so a display card is blank
for as short a time as possible. Without `nvidia` in the image, it goes
last and still runs before an `nvidia` loaded from the root filesystem.
Images are rebuilt on install. mkinitcpio ignores drop-ins for an image
built with `-c <config>` or a preset `_config`; put `gpu_unlock` right
before `nvidia` in that config's `MODULES` yourself.
- **initramfs-tools:** `/usr/share/initramfs-tools/modules.d/gpu-unlock.conf`.
- **dracut:** `/usr/lib/dracut/dracut.conf.d/gpu-unlock.conf`.

With initramfs-tools and dracut, the module is built for every installed
kernel that has headers, and those kernels' images are rebuilt on install
and again on removal. Kernels installed later get the module and a new
image from the kernel's own install hooks.

There is no `softdep nvidia pre:` rule: kmod honours only the first
`softdep nvidia` line it reads, so other packages' rules (`nvidia-utils`,
`gddr6_hwmon`) would silently override it, or it would override theirs.

The module does nothing unless the kernel command line opts in:

```
gpu_unlock.targets=all                          # every unbound GA102/GA103/GA104/GA106/AD103 card
gpu_unlock.targets=0000:21:00.0                 # only these cards
gpu_unlock.targets=0000:21:00.0,0000:22:00.0    # comma-separated
```

Addresses are full PCI names as `lspci -D` prints them (hex digits in either
case), independent of GPU numbering and display role. An entry that is not
a GA102/GA103/GA104/GA106/AD103 card, including `all` mixed with addresses, is logged and ignored.
The cards are unlocked concurrently, about a second in all however many
there are, and each on its own: a card that is already bound or powered
down is skipped. Bus mastering that the board firmware left on (Supermicro
X11 does this whatever its option ROM and bus master settings) is cleared
first; a card some driver already initialised is still refused. Then
`nvidia` loads and binds every card as usual.

AD103 (`0x2702` RTX 4080 SUPER, `0x2704` RTX 4080) is selected per card and
uses the AD102 Booter, GSP boot image, `.fwsignature_ad10x` section and
four-PLM payload instead of the GA102 ones. `0x2702` is validated with driver
615.78.08 (see [docs/VALIDATION.md](docs/VALIDATION.md)); `0x2704` uses the
same profile but has not been tested with this module. The four PLM values,
the payload marker `0x67b00001` and the SEC2 debug value `0xda550000` are
required after the Booter runs, and a mismatch fails the unlock before the
FLR. Use the read-only preflight on a card that has not been tested yet.

### Read-only preflight

`gpu_unlock.dry_run=1` makes the module claim each selected card, read and
log `PMC_BOOT_0`, the GFW/SEC2 state, `FEAT_OVR_PLM`, SS0/SS1, WPR2, the
layout inputs, and the PROM/FWSEC geometry, then hand the card back. It
writes no BAR0 register, starts no firmware, resets nothing and registers no
resume handler, so `nvidia` binds the card as usual afterwards. Combine it
with `targets` to check one card:

```
gpu_unlock.targets=0000:0a:00.0 gpu_unlock.dry_run=1
```

The `inspect:` lines in `journalctl -k -b` show the cold values the real
unlock would act on, and whether the card's firmware geometry parsed. Remove
`dry_run` to unlock.

The display card works too. Its firmware framebuffer (the boot console and
Plymouth) keeps running while the unlock's firmware steps run, and is
removed just before the card's reset, as `nvidia-drm` would do a moment
later anyway. Its screen is blank from that reset until `nvidia-drm` takes
over.

Loading the module by hand after boot does nothing to a card that `nvidia`
has bound.

With Secure Boot enabled the DKMS-signed module needs the local MOK key
enrolled once:

```sh
sudo mokutil --import /var/lib/dkms/mok.pub   # then complete it at the next boot
```

Without enrollment the module still loads, but the kernel is tainted. On a
system without Secure Boot the signature is not checked.

Suspend to RAM powers the cards off, which resets the speed selectors. After
resume the module unlocks every card it unlocked at boot again, before
`nvidia` resumes it: its suspend notifier runs just ahead of the one `nvidia`
resumes its cards from (`NVreg_UseKernelSuspendNotifiers=1`, the default in
current drivers; `nvidia-resume.service` runs later still). This adds about a
second to resume. A card that kept power, as in suspend to idle, is still
unlocked and is only read. A card that `nvidia` has already resumed, or that is
bound to another driver, is left as it is, and the log says why. Add
`gpu_unlock.resume=0` to the kernel command line, or write `0` to
`/sys/module/gpu_unlock/parameters/resume`, to turn this off. Hibernation is
not covered.

Check the result with:

```sh
journalctl -k -b | grep gpu_unlock
lspci -nnk -s 0000:21:00.0
```

To stop unlocking, remove `gpu_unlock.targets=` from the kernel command line;
for a one-off boot, delete it in the bootloader's edit screen. Removing the
packages also removes the module and the initramfs drop-ins.

If the firmware on a card never stops, the module logs it and keeps the DMA
buffers that firmware may still use until the next power off. Power off
before trying again.

The kernel module and local packaging are GPL-2.0-only. The generated payload
derives from the pinned CMP reference; NVIDIA firmware remains under its own
terms. See `firmware/NOTICE.md` and `licenses/` for attribution and notices.

# Upstream and attribution

The GA10x module, packaging and documentation come from
[Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock); support that work
through the links in the upstream repository. The AD102 payload table and the
AD103 geometry are adapted from
[TOTORONG/AD103_win_unlocker](https://huggingface.co/TOTORONG/AD103_win_unlocker).
See `firmware/NOTICE.md` for licensing and provenance.