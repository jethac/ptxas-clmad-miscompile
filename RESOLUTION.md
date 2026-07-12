# Resolution — NVONLINE #6425057: NOT a bug (old disassembler + no runtime check)

**Date:** 2026-07-12 · **Box:** Jetha-WS3 (RTX 5060 Ti, sm_120, driver 595.79) ·
**Verdict:** WITHDRAWN. `clmad` compiles and executes correctly on Blackwell. There
is no ptxas miscompile and no cuobjdump bug. NVIDIA/Yuki's diagnosis — an old
disassembler that couldn't decode the new instruction — was correct.

## Two mistakes in the original report
1. **Disassembly-only** — the probe harness never executes ("ptxas is the oracle …
   the answer a runtime test would have, for free" — results/FINDINGS_ptx93.md). The
   "result = 5" was *inferred* from a SASS listing, not measured.
2. **Old disassembler** — the listing came from cuobjdump V13.2.78.

## 1. Runtime ground truth — clmad is correct
`clmad_bug.ptx` assembled with ptxas 13.3.73 for sm_120a, launched on the RTX 5060 Ti
via the CUDA Driver API (`run_clmad.py`). `result = clmul(a,b) XOR c`, all correct:

| a | b | c | got | expected | store-b would give |
|---|---|---|-----|----------|--------------------|
| 3 | 5 | 7 | **8** | 8 | 5 |
| 0xF | 0xF | 0 | 0x55 | 0x55 | 0xF |
| 0x1234 | 0xABCD | 0x55 | 0x0bf62011 | 0x0bf62011 | 0xABCD |
| 0xFFFF…FFFF | 2 | 0 | 0xFFFF…FFFE | 0xFFFF…FFFE | 2 |
| **0** | 0xABCDEF | 0x99 | **0x99** | 0x99 | 0xABCDEF |

a=3,b=5,c=7 → 8, not the claimed 5. a=0 → c, not b. Definitively not "stores b."

## 2. The /*0060*/ gap = an undecoded instruction, not missing codegen
The 0x60 slot holds a real instruction (opcode 0x7c2c, dst R2). It writes R2 with
clmul(a,b)^c between the load of b at 0x10 and the store at 0x70, so the store is
correct. With matched tooling it disassembles as `CLMAD.LO R2, R2, UR6, R6`.

## 3. Mechanism: cuobjdump delegates decoding to nvdisasm (verified)
`cuobjdump -sass` shells out to `nvdisasm` for SASS decode. Controlled test on the
same cubin:

| disassembler behind cuobjdump | /*0060*/ |
|---|---|
| nvdisasm **V13.3.73** (matched) | `CLMAD.LO R2, R2, UR6, R6` ✅ |
| nvdisasm **V13.2.78** (report's) | **omitted, no diagnostic** ❌ |
| no nvdisasm present | **omitted, no diagnostic** ❌ |

So it is purely a version issue: nvdisasm ≥13.3 knows CLMAD; 13.2.78 does not, and
silently drops it. `pip install nvidia-cuda-nvcc==13.3.73` ships ptxas but not
nvdisasm/cuobjdump (separate packages), so the report's disassembly fell back to the
system CUDA 13.2 tools — exactly the old-disassembler case Yuki described.

**Earlier mis-step (corrected):** an intermediate check appeared to show cuobjdump
13.3.73 *itself* dropping CLMAD — that was an artifact of running it without a
co-located nvdisasm. With the matching nvdisasm present it decodes correctly. There
is **no cuobjdump bug**; do not report one.

## Toolchain
- ptxas: V13.3.73 (pip `nvidia-cuda-nvcc==13.3.73`) — correct codegen.
- Disassembler in the report: cuobjdump/nvdisasm V13.2.78 (system CUDA 13.2) — too old
  for CLMAD.
- Matched disassembler: nvdisasm V13.3.73 → `CLMAD.LO`.

## Disposition
- Reply to NVONLINE #6425057: concede, credit Yuki's diagnosis, attach the runtime +
  matched-tooling evidence, ask to close. Draft: `../../nvidia-6425057-reply.md`.
- Public repo `github.com/jethac/ptxas-clmad-miscompile`: README rewritten to a
  retraction (this finding). No NVIDIA bug to file — not in ptxas, not in cuobjdump.

Evidence: scratchpad/clmad_verify/ (wheels, cubins, run_clmad.py, isolation tests).
