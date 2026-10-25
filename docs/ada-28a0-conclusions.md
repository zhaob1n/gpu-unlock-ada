# RTX 4060 Laptop (`10de:28a0`) conclusions

One AC-powered host, one AD107M RTX 4060 Laptop (`10de:28a0`, chip
`0x197000a1`, VBIOS `95.07.21.00.24`) at `0000:01:00.0`. Kernel
`7.3.0-5-cachyos`, stock NVIDIA open driver `615.78.08`. No battery; the
only power supplies were AC and a USB-C source. This is not an AD107-family
allowlist and not a claim about other laptop firmware.

The code change only admits this exact ID and routes it through the existing
AD102 Booter, GSP and four-PLM payload. There is no laptop-specific
transaction. `gpu_unlock.ad107_experimental` defaults off. `targets=all` does
not select it.

## Measured issue rate

Same cubin (`33aa6db385aa9eb8a1f73bb2f74891aea84794326b7c40992bc642f93a3c614d`),
5000 iterations, three repetitions, 270336 cells checked per mode. Both boots
used `NVreg_DynamicPowerManagement=3`. Sampled SM clocks during the runs were
2520 MHz stock and 2505 MHz unlocked. The power-limit query returned `[N/A]`;
clocks were not locked. Zero mismatches in every mode.

| Mode | Stock | Unlocked | Ratio |
| --- | ---: | ---: | ---: |
| FP16×FP16 → FP32 | 27.64 TFLOPS | 55.07 TFLOPS | 1.99× |
| TF32×TF32 → FP32 | 15.48 TFLOPS | 30.65 TFLOPS | 1.98× |
| UINT4×UINT4 → INT32 | 246.2 TOPS | 244.8 TOPS | 0.99× |

The RM getter `0x20801230` changed only `fmla32`, `imla2` and `imla4` from
half (`1`) to full (`0`). `imla0`, `fmla16`, `dp`, `ffma`, `imla1` and
`imla3` were already full.

What that means:

- The doubled paths are tensor MMA that accumulates into FP32: FP16 and TF32,
  measured here. BF16→FP32 uses the same `fmla32` gate on other Ada reports;
  this host did not run a BF16 kernel.
- FP32 SIMT (`ffma`), FP64 (`dp`) and FP16→FP16 (`fmla16`) were already full.
  They are not an unlock result.
- Unsigned INT4 issue rate did not change, even though `imla2` and `imla4`
  read back as full. INT8 was not measured here; other Ada reports also saw
  no INT8 or INT4 speedup.
- These are register-resident instruction rates, not GEMM or application
  speedups.

The unlock boot logged marker `0x67b00001`, `SS0=0x88888888`, `SS1=0x8`,
post-FLR readout all zeros, and `gpu_unlock: unlocked`. The stock driver then
loaded. No NVRM Xid or fallen-off-bus message was seen on either comparison
boot. The stock comparison was a warm reboot, not an EC power removal.

## What is required

- Route `28a0` to the AD102 assets. Adding the PCI ID alone selects the GA102
  Booter, which halts on Ada.
- Run the payload until the marker and required permissions pass, then write
  the selectors. Those writes are the rate change.
- FLR after the writes. It is not the rate change. It clears WPR2 and leaves
  GSP halted so the stock driver can bind. On this card the selectors survived
  FLR. Skipping FLR was not tested; the module fails the handoff if WPR2 stays
  enabled.
- Load the module from the initramfs before `nvidia` binds, with one exact PCI
  address and `gpu_unlock.ad107_experimental=1`. The flag and the exact-address
  rule are admission gates, not extra silicon steps.

Minimal kernel command line for this host:

```text
gpu_unlock.targets=0000:01:00.0 gpu_unlock.ad107_experimental=1
```

Use the address printed by `lspci -D`. If `sdboot-manage` has
`OVERWRITE_EXISTING=yes`, put that line on a separate boot entry. The next
kernel update rewrites `linux-cachyos.conf` and drops options added to it.

## What this host did not need

`NVreg_DynamicPowerManagement=0x00` was not part of the working minimal set.
Both comparison boots used the driver default `3`, and the unlocked boot still
showed full rates and the 1.99× / 1.98× issue rates above.

That parameter disables Runtime D3. It matters where idle RTD3 cuts dGPU power
and drops the volatile selectors. This host has no battery, the GPU endpoint
advertises `D3cold-` and `NoSoftRst+`, and the GPU ACPI node has no `_PR0` or
`_PR3`. The upstream bridge does have `_PR3`. NVIDIA's own RTD3 requirement is
`_PR0` and `_PR3` on the GPU. An idle DPM=3 versus DPM=0 comparison was not
run, so this does not prove that every AC-powered 4060 Laptop can omit the
parameter. It does show that forcing `0x00` was not required for this boot or
for the matched measurement.

Also not required here: cutting AC power, a laptop-specific transaction, or
the observational FBPA permission (it stayed `0xffffff0f`; `required=0`).

## Limits

One host, one warm stock/unlocked pair. Suspend, resume, and a long idle with
Runtime D3 were not tested. No application or GEMM speedup is claimed.
Neighboring AD107 IDs are not admitted.
