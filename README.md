# ptxas 13.3 miscompiles `clmad` on Blackwell (emits no carry-less arithmetic)

**Toolchain:** `ptxas` **V13.3.73** (CUDA 13.3; `pip install nvidia-cuda-nvcc==13.3.73`), PTX ISA 9.3.
**Affects:** `sm_100a`, `sm_103a`, `sm_120a`, `sm_121a` — every Blackwell target tested.

## Summary
`clmad.lo.u64 d, a, b, c` — the carry-less multiply-add introduced in PTX ISA 9.3
(§9.7.1.5: *"performs a carryless multiplication of `a` and `b`, followed by a
carryless addition of `c`"*, all operands unsigned 64-bit) — **assembles without
error but generates no carry-less-multiply arithmetic**. The emitted SASS loads
the operands, computes nothing, and stores one of the inputs (`b`) as the result.

## Reproduce
```
ptxas -arch=sm_121a clmad_bug.ptx -o clmad_bug.cubin   # accepts, no diagnostic
cuobjdump -sass clmad_bug.cubin
```
`clmad_bug.ptx` computes `d = clmul(a,b) ^ c` (a=3, b=5, c=7 → expected 8) and
stores `d`.

## Observed SASS (sm_121a) — see `observed_sass.txt`
```
LDC.64 R2, c[0x0][0x390] ;   // R2 = param b
LDC.64 R4, c[0x0][0x380] ;   // R4 = output pointer
STG.E.64 desc[UR4][R4.64], R2 ;   // stores b, NOT clmul(a,b)^c
EXIT ;
```
No `XMAD`/`IMAD.WIDE`/`LOP3`/`PRMT`/`POPC` — zero carry-less arithmetic. Across all
four Blackwell targets: **0 arithmetic instructions emitted**, wrong value stored.

## Expected
A GF(2) polynomial multiply of `a` and `b` (low 64 bits) XOR `c`, lowered to real
hardware or an emulation sequence — `clmul(3,5)^7 == 8`. Instead the result is `b`.

## Files
- `clmad_bug.ptx` — minimal repro
- `observed_sass.txt` — SASS as emitted by ptxas 13.3.73
