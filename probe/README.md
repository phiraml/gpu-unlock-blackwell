# probe

`mma_rate.py` measures the thing the unlock changes, the issue-rate gap between
FP16 tensor FMA into FP32 and into FP16. Everything in this repository's
feasibility claim rests on this number, so it is a script rather than a
paragraph.

Torch only. Run it before and after any unlock attempt and diff the receipts.

```
python mma_rate.py                    # table and verdict
python mma_rate.py --json mine.json   # also write a receipt to diff
```

`5090-stock.json` is the baseline taken on an RTX 5090, driver 610.88, 575 W,
under WSL2 on 2026-10-24.

`5060ti-stock.json` is the baseline taken on an RTX 5060 Ti (GB206, device id
0x2D04), driver 580.178.04, 180 W, native Linux, on 2026-11-04. FP16 into FP32
issues at 0.565x the FP16-accumulate rate, BF16 into FP32 at 0.567x.
It agrees with the other stock 5060 Ti receipts reported in #4 (0.555x) and #6
(0.547x).
