# gpu-unlock-blackwell

Research toward a pre-driver SM speed-select unlock for Blackwell Geforce cards,
starting with the RTX 5090 (GB202).

The [gpu-unlock](https://github.com/Zagitta/gpu-unlock) family removes a speed
restriction NVIDIA fuses into consumer silicon: the FP16/BF16 tensor FMA that
accumulates into FP32 issues at a fraction of the FP16-accumulate rate. On
Ampere the unlocked rate is double, on Ada it is double, and on a stock RTX 5090
it measured **0.68x** in this repository's own harness. So the restriction exists
on Blackwell.

Nobody has ported the unlock to Blackwell yet. There is no public GB20x booter
payload. This repository exists to establish that, record the measurements, and
lay out the port.

## Status

| Item | State |
| --- | --- |
| Fuse present on GB202 | **measured**, 0.68x and 0.66x on two separate modes |
| Fuse present on GB206 | **measured** on three RTX 5060 Ti, 0.547x to 0.565x across Linux and Windows, drivers 580 to 616 (#4, #6, `probe/5060ti-stock.json`) |
| Fuse present on GB205 | **reported** on an RTX 5070, 0.52x with an independent harness (#8) |
| Public Blackwell payload | does not exist |
| Public Blackwell debug booter | does not exist |
| Reference material for a port | complete, captured in `docs/` |
| Port itself | not started |

## The measurement

`probe/mma_rate.py` measures the ratio directly. It is the whole question: a
ratio near 1.0 means nothing to unlock, a ratio near 0.5 means a doubling sits
behind the fuse. Best of three, across five shapes, same card, same boot, same
power cap.

On an RTX 5090, driver 610.88, 575 W cap, WSL2:

| Mode | Peak | vs reference |
| --- | ---: | ---: |
| FP16 in, FP16 accumulate | 334.68 TFLOP/s | 1.000 |
| FP16 in, FP32 accumulate | 226.14 TFLOP/s | 0.676 |
| BF16 in, FP32 accumulate | 222.29 TFLOP/s | 0.664 |

Full receipt in `probe/5090-stock.json`, including every shape.

Read this as an issue-rate measurement, not an application speedup. It says the
restriction is present. It does not promise a matching number in any particular
kernel.

Run it on your own card:

```
python probe/mma_rate.py --json probe/mine-stock.json
```

## Why the port is hard

The unlock is a single 4-byte write performed from GPU firmware. Getting that
write requires all of the following, and only some exist for Blackwell.

1. **A decrypted debug booter to mine.** The signed debug SEC2 booter is
   encrypted with a known test key, which is why anyone can disassemble it and
   hunt for ROP gadgets. Ada and Ampere debug booters ship inside NVIDIA's open
   module as `IMAGE_DBG`. **There is no GB20x debug booter anywhere public.**
2. **A gadget chain.** The booter has a pristine stack overflow in its signature
   loader. A chain of gadgets turns it into one arbitrary write.
3. **The right MMIO target.** Clear `FEAT_OVR_PLM` at BAR0 `0x00823804`, which
   gates writes to the two speed-select registers.
4. **Correct instruction addresses.** The chain is built from absolute Falcon
   IMEM addresses, so every gadget address must be mined and re-verified per
   booter image. Addresses are not portable across chips.
5. **A kmod that runs before the driver.** Take the card, run the payload, PCIe
   FLR, hand the card back untouched to the stock driver.

Step 5 is written and reusable. Step 1 is the blocker.

## Why there is no GB20x booter

`docs/evidence/nvidia-booter-dispatch.txt` holds the verbatim dispatch. The
`kgspGetBinArchiveBooterLoadUcode` hal has explicit variants only for TU102,
TU116, GA100, GA102 and AD102. Every other chip, Blackwell included, lands in a
single shared fallback, `_80f438`.

In the open module that fallback is a **NULL stub**, because the object is
generated separately. So the GB20x booter image lives only inside NVIDIA's
closed `nv-kernel.o`, unlike the Ampere and Ada images which are published.

## What is reusable from the existing work

Enumerated from the Ada repository, since a Blackwell port is mostly an Ada port
with a new payload and a new target list.

| Piece | Reusable? |
| --- | --- |
| Payload container format and generator (`firmware/generate_v67.py`) | yes, format unchanged |
| Firmware asset extraction and the whole packaging/initramfs story | yes |
| `unlock_610.c` booter state machine, ~64 KB | partly, addresses and sizes differ |
| Free-standing layout, aperture and vbios modules | yes |
| Arch abstraction (`arch_610.h`, currently a two-value enum) | yes, the seam a new chip slots into |
| The chain itself | no, must be re-mined |

The arch seam is currently just `GU_BOOTER_GA10X` and `GU_BOOTER_AD10X` with a
device-ID table. A Blackwell port adds a third value and a third payload.

## The payload, concretely

A 64,000-byte image of one repeated fill dword, plus about 20 patched dwords at
fixed offsets that encode the chain. Two entries out of those 20 tell the chain
which address to write:

- `0/f9cc` holds nothing; the chain writes the address already in `0xf9cc`
  (`FEAT_OVR_PLM`, `0x00823804` on Ada) with the value in `0xf9d4`
  (`0xffffffff`).

So adapting a chain is: fix the funnel, re-mine the gadgets that move the value,
re-verify that the flow lands back in the clean path rather than the overflow
epilogue. TOTORONG's `docs/ANALYSIS.en.md` documents the per-booter function
addresses this depends on.

## Next step

The single highest-value action is to obtain a decrypted GB20x debug booter.
Options, roughly in order of preference, are laid out in `docs/port-plan.md`.

## Reading order

- `docs/port-plan.md` the port, phase by phase, with the prerequisites named
- `docs/feasibility.md` the full reasoning and the evidence behind every claim
- `docs/evidence/` raw captured artifacts, so nothing here is a summary of a summary

## Prior work this builds on

- [Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock), Ampere, and its `docs/How Ampere was unlocked.md`
- [JustAResearcher/gpu-unlock-ada](https://github.com/JustAResearcher/gpu-unlock-ada), Ada AD102/AD103
- [nlzy/turing-unlock](https://github.com/nlzy/turing-unlock), Turing
- [TOTORONG/AD103_win_unlocker](https://huggingface.co/TOTORONG/AD103_win_unlocker), the AD102 payload and booter analysis
- [amoghmunikote/cmpunlocker](https://github.com/amoghmunikote/cmpunlocker), the original CMP V67 technique
