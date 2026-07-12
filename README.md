# RETRACTED — `clmad` is NOT miscompiled on Blackwell

**This report was wrong. There is no ptxas bug (and no cuobjdump bug).** `clmad`
assembles and executes correctly on Blackwell. The original claim below was based on
a disassembly produced by an **old disassembler** that could not decode the new
instruction, plus the mistake of never actually running the kernel. Retained as a
public correction. — 2026-07-12

## What actually happened
1. **The analysis was disassembly-only** — the kernel was never executed. The
   "result = 5, not 8" was *inferred* from a SASS listing, not measured.
2. **The disassembler was old.** `cuobjdump` hands SASS decoding to `nvdisasm`.
   `nvdisasm` **V13.2.78** does not know the PTX 9.3 `clmad` opcode, so the
   instruction at `/*0060*/` was dropped from the listing **with no diagnostic**.
   Its absence made the store of the (correctly computed) result register look like
   it was storing input operand `b`.

## Ground truth
**Runtime** — `clmad_bug.ptx` built with ptxas 13.3.73 for `sm_120a`, executed on an
RTX 5060 Ti (driver 595.79); `result = clmul(a,b) XOR c` for every input:

| a | b | c | result | expected |
|---|---|---|--------|----------|
| 3 | 5 | 7 | **8** | 8 |
| 0 | 0xABCDEF | 0x99 | **0x99** | 0x99 (= c, not b) |
| 0xFFFFFFFFFFFFFFFF | 2 | 0 | 0xFFFFFFFFFFFFFFFE | 0xFFFFFFFFFFFFFFFE |

**Matched tooling** — with `nvdisasm` **V13.3.73** the instruction decodes correctly
(see `matched_tooling_sass.txt`):

```
/*0060*/  CLMAD.LO R2, R2, UR6, R6 ;
```

vs. the original old-disassembler listing (`observed_sass.txt`), where `/*0060*/` is
simply missing.

## Reproduce
```
pip download --only-binary=:all: --no-deps \
  nvidia-cuda-nvcc==13.3.73 nvidia-cuda-nvdisasm==13.3.73
ptxas    -arch=sm_120a clmad_bug.ptx -o clmad_bug.cubin   # assembles, no diagnostic
nvdisasm -c            clmad_bug.cubin | grep 0060        # /*0060*/ CLMAD.LO ...
```
Then launch it (any Driver-API/runtime harness) with a=3, b=5, c=7 → returns **8**.

## Takeaway
Don't infer a miscompile from disassembly alone — run it. And match your
`nvdisasm`/`cuobjdump` version to the toolkit that produced the cubin; an older
disassembler silently omits instructions it doesn't recognize.

*NVIDIA incident #6425057 — reported 2026-07-07, withdrawn 2026-07-12. Thanks to the
NVIDIA engineer who correctly flagged the disassembler version.*

## Files
- `clmad_bug.ptx` — the minimal kernel (loads a,b,c; `clmad.lo.u64`; stores result)
- `observed_sass.txt` — original listing (old disassembler; `/*0060*/` missing)
- `matched_tooling_sass.txt` — nvdisasm 13.3.73 (`/*0060*/ CLMAD.LO ...`)
- `RESOLUTION.md` — full investigation notes
