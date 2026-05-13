# CCSDS-123.0-B-2 On-Board Compression Architecture Analysis

**Executive Summary:** This document analyses the architecture path from the current verified NZ=1 single-band compressor to a multi-band (NZ=50) CCSDS-123.0-B-2 implementation for FF (ZU7EG) and JTYU (ZU17EG). The recommended approach is **Option A** (C-slow predictor + sample-adaptive encoder under BIP ordering) — best compression, no GSS Rust decompressor changes needed, reuses existing encoder structure. Option C (block-adaptive encoder) offers easier timing closure at higher clock rates but requires a new GSS decode path. All frequency and throughput estimates in the FUTURE columns are **pre-synthesis** and subject to Vivado implementation results.

## Mission & Standard Context

| | Firefly (FF) | JTYU |
|---|---|---|
| **Target Standard** | CCSDS-123.0-B-2 (mandated) | CCSDS-123.0-B-2 (same core) |
| **Compression Mode — Phase 1** | Lossless (PAVI-CORE-COMPRESSION-001) | Lossless |
| **Compression Mode — Phase 2** | Near-Lossless (PAVI-CORE-COMPRESSION-004) | TBD |
| **Bands (NZ)** | 50–64 (hyperspectral) | 2–8 (pan + multispectral) |
| **Chip** | ZU7EG | ZU17EG |

> **Note on Tsigkanos et al. (3.3 Gbps, IEEE TETC 2021):** This paper implements CCSDS-123.0-**B-1** on a Virtex-5QV space-grade FPGA.
> It is used here as the **architectural reference** for C-slow retiming and BIP ordering on the predictor.
> The B-1 and B-2 predictor weight-update feedback loop is structurally identical — C-slow retiming, the
> spectral slice buffer, and the BIP task-level parallelism argument all transfer directly to our B-2 predictor.
> B-2 adds near-lossless mode and a hybrid entropy coder; neither changes the feedback loop that C-slow targets.
> Our RTL target is B-2. The paper is a proof point, not the standard we are implementing.

---
***

## CCSDS-123 Architecture: TODAY vs FUTURE

> **Current RTL scope:** The verified baseline operates at **NZ=1, P_eff=0, 1 band, 1 tile, NX=2048, NY=512** only.
> All multi-band paths (spectral prediction, per-band weight vectors, BIP reorder, C-slow register files)
> require **new RTL modules, modifications to existing modules, and full re-verification** against CCSDS
> multi-band golden reference data. The FUTURE column describes the target architecture, not existing RTL.

| **TODAY** (NZ=1, P=0, VERIFIED) | **FUTURE** (NZ=50, P=3, C-slow, BIP) |
|:---|:---|
| SSD (BIL, NZ=1 = trivially raster) | SSD (BIL — 50 bands × 2048 pixels per line) |
| ↓ | ↓ |
| AXI DMA (128-bit AXI-Stream) | AXI DMA (128-bit AXI-Stream) |
| ↓ | ↓ |
| **╔═ FID REORDER (TB only) ═╗** ← NZ=1: FID collapses to raster, trivial | **╔═ bil\_to\_bip\_reorder.v ← NEW MODULE ═╗** |
| collapses to simple raster order, 0 bits of BRAM | 🟢 **PING only (recommended):** 50×2048×16b = 1,600 Kbits, 89 BRAM18s. Phase A: BIL→PING (8,534 clks). Phase B: PING→BIP (102,400 clks). Total: 110,934 clks |
| **╚═══════════════════════════╝** | 🔴 **PONG bank (not recommended):** +1,600 Kbits, +89 BRAM18s. +7.7% faster but radio cap (500 Mbps FF / 1 Gbps JTYU) absorbs all gain → zero benefit. Save 89 BRAM18s for packetizer FIFOs |
| ↓ *s\_raw: 1 sample/clk (raster)* | **╚════════════════════════════════════╝** |
| ↓ | ↓ *s\_raw: BIP — clk0=band0,x0,y0 · clk1=band1,x0,y0 … clk50=band0,x1,y0 …* |
| | ↓ |
| **╔═ PREDICTOR ═╗** | **╔═ PREDICTOR (C-slow, SAME datapath) ═╗** |
| **Weights:** 6×7b = 42 bits (regs) | **Weights\[z\]\[0:5\]:** 50×6×7b = 2,100 bits (LUTRAM) |
| **sigma, gamma, k\_z:** 3 single regs = 37 bits | **sigma\[z\], gamma\[z\], k\_z\[z\]:** σ:50×24b + γ:50×9b + k:50×4b = 1,850 bits (LUTRAM) |
| **central\_diff\_fid\_buffer:** (what a correct FID impl. *would* need) P\_eff × NX × NY × 19b | **diff\_local\[z\]\[0:P-1\] ← REPLACES FID BUFFER** — Tsigkanos BIP local-diff-vector method |
| NZ=1: P\_eff=0 → **0 bits** 🟢 passthrough (current RTL: `mem[NZ][NZ]` degenerates to no-op) | In BIP+C-slow: d(z-1) computed exactly NZ=50 clks ago → in P-entry rolling window. **No NX×NY BRAM needed.** |
| NZ=2: P\_eff=1 → **19 Mbit** 🔴 ZU7EG=11M · 🔴 ZU17EG BRAM=28M — fits but leaves nothing | 50×3×19b = **2,850 bits (LUTRAM)** — 350,000× reduction vs 19 Mbit (NZ=2) |
| NZ=10: P\_eff=3 → **57 Mbit** 🔴 ZU7EG · 🔴 ZU17EG BRAM+URAM=57M exactly 100% | **Spectral Slice Buffer (N/W/NW/NE):** 4-FIFO: (NX+2×NZ)×D = (2048+100)×16 = 34,368 bits = 2 BRAM18s |
| NZ=50: P\_eff=3 → **57 Mbit** 🔴 same · 🔴 no Zynq chip has headroom at NZ≥10 | |
| **Blocker:** BSQ/FID needs d(z-1,y,x) for any pixel on demand → entire prev-band diff image must be pre-stored. Blocker hits at NZ=2. **Note:** Current RTL `central_difference_fid_buffer` uses `mem[0:NZ-1][0:NZ-1]` (NZ²×19b shift register) — this is a **placeholder that only works at NZ=1 P_eff=0**. A correct FID/BSQ implementation for NZ≥2 would require the full NX×NY×P×19b storage shown above. The Tsigkanos BIP method eliminates this requirement entirely. | |
| **Row Buffer (N/W/NW/NE):** 3×2048×16 = 96 Kbits = 6 BRAM18s ← today's BRAM cost | |
| PREDICTOR BRAM: 96 Kbits (0.85% ZU7EG) for NZ=1 · 19 Mbit NZ=2 P=1 · 57 Mbit NZ=10,50 P=3 | PREDICTOR BRAM: 33.6 Kbits (0.30% ZU7EG) · PING: 1,600 Kbits (14.2%) 🟢 |
| **╚═══════════════════════════╝** | **╚════════════════════════════════════╝** |
| ↓ *delta\_z: 1 mapped residual/clk* | ↓ *delta\_z: 1 mapped residual/clk (BIP order)* |
| ↓ | ↓ |
| **╔═ ENCODER ═╗** | **╔═ ENCODER (C-slow, SAME datapath) ═╗** |
| sigma, gamma, k\_z: 3 single regs = 37 bits | sigma\[z\], gamma\[z\], k\_z\[z\]: 50×24b+50×9b+50×4b = 1,850 bits (LUTRAM) |
| ENCODER BRAM: 0 | ENCODER BRAM: 0 |
| **╚═══════════════════════════╝** | **╚════════════════════════════════════╝** |
| | |
| **TODAY TOTAL BRAM: 96 Kbits (0.85% ZU7EG)** | FUTURE BRAM 🟢 PING only: **1,634 Kbits (14.5% ZU7EG)** |
| | FUTURE BRAM 🔴 PING+PONG (not recommended): **3,234 Kbits (28.7% ZU7EG)** |

***

### Chip BRAM Capacity

| Chip | BRAM | URAM | BRAM+URAM | Verdict for central_diff_buffer |
|:---|:---|:---|:---|:---|
| ZU7EG | 11 Mbit | none | 11 Mbit | NZ=1 🟢 · NZ≥2 🔴 |
| ZU17EG | 28 Mbit | 29 Mbit | 57 Mbit | NZ=2 🟡 (19 Mbit fits, but leaves only ~9 Mbit BRAM headroom before accounting for the rest of the design) · NZ≥10 🔴 (57 Mbit = 100% of chip) |
| ZU19EG | 35 Mbit | 36 Mbit | 71 Mbit | NZ=2 🟢 (fits with margin) · NZ≥10 🔴 (57 Mbit = 80% of chip) |

### No Chip Solves NZ≥10 with BSQ/FID

| NZ | P_eff | central_diff | ZU7EG | ZU17EG | ZU19EG |
|:---|:---|:---|:---|:---|:---|
| 1 | 0 | 0 bits | 🟢 | 🟢 | 🟢 |
| 2 | 1 | 19 Mbit | 🔴 | 🟡 fits, but little headroom for the rest of the design | 🟢 |
| 10 | 3 | 57 Mbit | 🔴 | 🔴 saturates chip resources | 🔴 80% of chip used |
| 50 | 3 | 57 Mbit | 🔴 | 🔴 | 🔴 |

> **Tsigkanos Fix:** `diff_local = P×NZ×19b = 3×50×19 = 2,850 bits (LUTRAM)` — **350,000× reduction.** Works on any chip including ZU7EG. No NX×NY BRAM needed.

> **Ping-Pong Decision:** +7.7% speed, but radio cap absorbs all gain. Save 89 BRAM18s for packetizer FIFOs, housekeeping, science metadata.

***
`PARAMETERS: NZ=50, P=3, D=16, NX=2048, NY=512, OMEGA=4, DIFF_WIDTH=D+3=19b`

--------------------

***

## Full Resource Table — ZU7EG, NZ=1→50

> **Note:** All clock frequency and throughput figures for the FUTURE columns are **estimated, pre-synthesis**.
> Actual achievable frequencies depend on Vivado implementation and will be confirmed during synthesis.
> Compression ratio estimates for NZ=50 are based on CCSDS Green Book test corpus data, not measured on mission imagery.

| Resource | NZ=1 (Current, Verified) | NZ=50 Option A (C-slow, Sample-Adaptive) | NZ=50 Option C (C-slow, BIP demux → per-band Block-Adaptive BSQ) | Tsigkanos Paper NZ=224 AVIRIS, Option A (Virtex-5QV) |
|---|---|---|---|---|
| **P_eff (spectral prediction)** | P_eff = 0 (NZ=1, no prior band exists) | P=3 (full spectral prediction) | P=3 (full spectral prediction) | P=3 (fixed in paper) |
| **Encoding order into encoder** | BIP (NZ=1 so BIP=BSQ, no difference) | BIP - encoder consumes per-band σ/γ/kz naturally | BIP from predictor - demux FIFO - per-band BSQ into block-adaptive encoder | BIP throughout |
| **BIL→BIP reorder needed?** | **Input path:** No - NZ=1, BIP=BIL=BSQ, no reorder needed at any stage | **Input path (SSD→Predictor):** Yes - BIL→BIP reorder module required before predictor; C-slow requires BIP input. **Predictor→Encoder:** No - predictor already outputs BIP natively | **Input path (SSD→Predictor):** Yes - same BIL→BIP reorder module as Option A. **Predictor→Encoder:** Yes - additional per-band demux FIFO (NZ × J × D = 51 Kbit) converts BIP residuals to per-band BSQ for block-adaptive encoder | **Input path:** Yes - BIL→BIP reorder before predictor. **Predictor→Encoder:** No - BIP used end-to-end |
| **Spatial neighbour RAM (sn, snw, sne)** | **FID row buffer:** 3 x buffers x 2048x16b = 96 Kbits = **6 BRAM18s** (2 BRAM18 per buffer, each 32 Kbits > 18 Kbits). Single `clkcomp` preferred; alternative: 1 BRAM18 at 3x clock (v9 CDC Issues 8&9). | **Spectral slice buffer** replaces FID: 4-FIFO cascade (NX + 2xNZ) x D = (2048+100) x 16 = 34,368 bits = **2 BRAM18s**. Smaller than NZ=1 FID because BIP order delivers neighbours in-stream. | Same as Option A - 2 BRAM18s. FID not used. | ~2 BRAM18s for NX=680, NZ=224 |
| **Datapath LUTs** | ~5,000 (2.2% of ZU7EG 230K) | ~10,000 (4.3%) | ~18,000 (7.8%) - includes block-adaptive option evaluators (600-1,000 LUTs) | 9,462 (11% of V5QV 82K) |
| **Flip-Flops** | ~5,000 (1.1%) | ~10,000 (2.2%) | ~12,000 (2.6%) | 9,990 (12% of V5QV) |
| **DSP48E2** | 0 (shift-add for inner product) | 6 (pipelined inner product, P=3) | 6 (predictor) + **4–32 DSP48E2** (block second-extension: parallelism chosen to sustain 1 residual/clk throughput matching predictor output rate. 4-way parallel: 4 DSPs, evaluates in J/4=16 clks ← sufficient within J-cycle block window. 32-way max parallel: 32 DSPs, evaluates in 1 clk) = **10–38 total**    | 6 (1% of V5QV 320) |
| **Weight vector storage** | 1 × (P+3) × (Ω+3) = 3 × 7 = 21 bits → **registers** | 50 × 6 × 7 = 2,100 bits → **LUTRAM** | Same as Opt-A — 2,100 bits LUTRAM | NZ × (P+3) × (Ω+3) → BRAM |
| **σ/γ/kz encoder state storage** | 3 × 37-bit registers (σ:24b, γ:9b, kz:4b) | NZ × 37 = 1,850 bits → **LUTRAM** (Option A: per-band register file indexed by band_z) | NZ × per-block stats → minimal LUTRAM; **no per-sample running state** — block-adaptive resets state per block, not per sample    | Accumulator storage: BRAM or LUTRAM by NZ |
| **centraldiff storage — BSQ/row-buffer style** | N/A — P_eff=0 at NZ=1, **zero spectral history RAM needed**. Current RTL's `mem[NZ][NZ]` is a passthrough at NZ=1. Blocker only activates if NZ≥2 AND P>0, and current RTL does **not** implement correct NZ≥2 spectral history — new RTL required | 🔴 A correct BSQ/FID implementation would need NZ × NX × NY × Cz ≈ 1 Gbit — exceeds ZU7EG. Must use Tsigkanos BIP local-diff method | 🔴 Same blocker if BSQ-style used — Tsigkanos method mandatory | 🔴 Same if BSQ row-buffer used |
| **centraldiff storage - Tsigkanos BIP local-diff (P previous-band vector only)** | N/A - NZ=1, P_eff=0; no spectral diff history needed at all | 🟢 NZ x P x Cz = 50 x 3 x 19 = **2,850 bits → LUTRAM** (Cz = D+3 = 19b, central diff width) | 🟢 Same - 2,850 bits LUTRAM | 🟢 NZ x Cz per-band → LUTRAM |
| **Per-band demux FIFOs (Option C only)** | N/A | N/A | **NZ × J × D = 50 × 64 × 16 = 51 Kbit → LUTRAM or 3 BRAM18s.** This is the BIL→BIP "fix" for block-adaptive: routes BIP residuals into NZ per-band J-sample FIFOs so encoder sees BSQ-ordered blocks, recovering ~0.1–0.3 bps compression penalty    | N/A |
| **Total BRAM (ZU7EG = 11 Mb / 11,264 Kbit)** | 96 Kbits (0.85%) - 6 BRAM18s (3x row buffers, NZ=1) | PING only: 1,634 Kbits (14.5%) - 89 BRAM18s reorder + 2 BRAM18s spectral slice + 0 encoder. PING+PONG (not recommended): 3,234 Kbits (28.7%) | Same as Opt-A for predictor BRAM; +demux FIFOs (~51 Kbits LUTRAM, 0 extra BRAM) | 2,970 Kbit = 83 BRAMs (27% of V5QV = 2.97 Mb) |
| **Total URAM (ZU7EG = 32 Mb)** | 0 | 0 - all storage fits in BRAM + LUTRAM | 0 | N/A (V5QV has none) |
| **External DDR needed?** | **No** — NZ=1 P_eff=0: centraldiff buffer = 0 (passthrough). If NZ≥2 P>0: 19 Mbit per extra band → DDR at NZ=3+    | No (Tsigkanos BIP local-diff method keeps spectral history in LUTRAM) | No | No — single chip |
| **Min NZ for C-slow benefit** | N/A | NZ ≥ pipeline_depth + 1; paper uses min NZ=14 for 213 MHz on V5QV. JTYU pan (NZ=2): budget = 1 stage — **no C-slow benefit**. JTYU multi (NZ=4): budget = 3 stages — marginal. JTYU pan+multi combined (NZ=6–8): budget = 5–7 stages — **feasible at 300 MHz**    | Same | min NZ=14 for 213 MHz; graceful degradation below |
| **GSS Rust decompressor change needed?** | **No change** — sample-adaptive is current standard; GSS already implements this path    | **No change** — same sample-adaptive format, same CCSDS header field (2-bit coder selection = 00) | **Yes — new decode path required.** Block-adaptive uses a completely different body format (block coding option overhead bits, Rice/second-extension/zero-block per block). GSS Rust must add `block_adaptive_decode()`. J and reference sample interval `r` are new header fields. Coordinate with GSS before committing    | No change (sample-adaptive) |
| **Compression ratio (lossless, NZ=1, P_eff=0)** | **~2:1 verified** on 1M-pixel RTL test (1,200 KB → 600 KB); 4.91 bits/pixel on set2 data | N/A | N/A | N/A |
| **Compression ratio (lossless, P=3 full spectral)** | N/A — P_eff=0 at NZ=1 | ~3:1–5:1 FF hyperspectral (NZ=50+); ~2.5:1–3.5:1 multispectral (NZ=8, P=3) | **~3:1–4.8:1** — per-band demux (Option C) recovers most BIP penalty; without demux (raw BIP into block-adaptive) degrades to ~2.3:1–3:1 due to mixed-band blocks   
| **Compression ratio: Sample-Adaptive vs Block-Adaptive detail** | N/A - NZ=1 P=0 current DUT uses sample-adaptive only. Verified: 4.91 bps on set2 data. No block-adaptive comparison applicable at NZ=1. | Sample-adaptive is best for lossless, encoding-order independent. Block-adaptive J=64 BSQ: 4.95-5.1 bps (0.1-0.3 bps worse). Block-adaptive J=64 with zero-block runs: can beat sample-adaptive by 0.1 bps. | Option C demux nearly matches sample-adaptive. Raw BIP into block-adaptive is the worst compression option (5.2-5.5 bps at J=8 BIP no demux). | Sample-adaptive reference |
| **Clock / Throughput at 100 MHz** | 100 MSamp/s; 1 sample/clk; SA: 1 byte every 1.63 clks = 61 MB/s out | Same input rate; SA encoder: 61 MB/s out | Same input; BA encoder: 100 MB/s avg (silent J−1=63 clks, then burst absorbed by output FIFO) | V5QV ran at 134 MHz ceiling |
| **Clock / Throughput at 200 MHz (estimated)** | 200 MSamp/s; SA: 123 MB/s out. **kz critical path ~5 ns: marginal at 200 MHz** — 400 MHz helper clock for kz update      | Same; C-slow gives Nz−1 pipeline stages — 200 MHz feasible for NZ≥8 | BA encoder: no per-sample feedback, **no kz loop**; 200 MHz feasible — DSP multiplier 3-stage pipeline closes automatically    | 213 MSamp/s paper headline on V5QV |
| **Clock / Throughput at 300 MHz (estimated)** | 🔴 Not feasible for NZ=1 — single-cycle feedback path limits ~120–150 MHz | 300 MSamp/s; C-slow NZ=50 gives 49 pipeline stages; SA: 184 MB/s out | 300 MSamp/s; BA encoder fully pipelineable — **no feedback at any frequency**; 300 MB/s avg; option evaluator critical path ~3 ns fits cleanly    | Not reported — V5QV process ceiling 134 MHz |
| **Clock / Throughput at 400 MHz (estimated)** | 🔴 Not feasible — single-cycle feedback | 400 MSamp/s; C-slow NZ=50 comfortable; SA: 245 MB/s out | BA encoder: 400 MB/s avg; multiplier pipeline (DSP48E2, 3-stage) and all comparators close timing comfortably; input FIFO absorbs J-sample burst    | Not applicable |
| **Radio downlink context ** | 500 Mbps target; requires ~66.5 Mpixel/s uncompressed throughput at 2:1 CR. **100 MHz predictor (100 Mpix/s) comfortably covers 500 Mbps.** Future 1 Gbps reuse on ZU17EG requires ~133 Mpix/s → 150–200 MHz minimum    | Same throughput requirement — C-slow adds no headroom benefit for NZ=1 downlink path | Same — Option C at 200 MHz gives large margin for both 500 Mbps and future 1 Gbps | Paper target: 3.3 Gbps downlink throughput |

***

## Option A vs Option C

| | **Option A** | **Option C** |
|---|---|---|
| **Predictor** | C-slow (NZ=50 register files, same datapath) | Same |
| **Encoder** | C-slow sample-adaptive (per-band σ/γ/kz register file) | Ordinary block-adaptive (no feedback, no C-slow needed in encoder) |
| **BIL→BIP reorder (SSD→Predictor)** | Yes - bil_to_bip_reorder.v required; PING buffer 1,600 Kbits = 89 BRAM18s | Same - identical reorder module shared with Option A |
| **BIP→Encoder reorder** | None - sample-adaptive is encoding-order independent | Per-band demux FIFO (NZ x J x D = 51 Kbit LUTRAM) routes BIP residuals to per-band BSQ for block-adaptive encoder |
| **Frequency (estimated, pre-synthesis)** | 200-300 MHz achievable (kz loop needs C-slow or 2-clock trick) | **300-400 MHz more easily achievable** - no per-sample feedback in encoder at any frequency |
| **Compression** | **Best** - 3:1-5:1, order-independent | Near-parity after per-band demux, 3:1-4.8:1; raw BIP into block-adaptive (no demux) degrades to 2.3:1-3:1 |
| **Compression detail** | Sample-adaptive: encoding-order independent, best lossless result. Block-adaptive J=64 BSQ: 0.1-0.3 bps worse. Block-adaptive J=64 with zero-block runs: can beat sample-adaptive by 0.1 bps. | Option C with per-band demux nearly matches sample-adaptive. Raw BIP into block-adaptive is worst case (5.2-5.5 bps at J=8, no demux). |
| **DSP48E2** | 6 (predictor only) | 6 (predictor) + 4–32 (block second-extension: parallelism chosen to sustain 1 residual/clk; 4-way=sufficient, 32-way=max parallel) = **10–38 total** |
| **GSS decompressor** | **No change needed** | **New decode path required** - coordinate before committing |
| **RTL complexity** | Two C-slow paths to manage (predictor + encoder) | C-slow predictor + new BA encoder + demux FIFO; encoder itself is simpler (no feedback FSM) |
| **What's new vs. today** | bil_to_bip_reorder.v (NEW); weightmem, sigma_mem, kz_mem become NZ-deep LUTRAM arrays; FID buffer replaced by Tsigkanos local diff vector (2,850 bits LUTRAM); C-slow register banking | Same predictor changes as Option A + new block-adaptive encoder mo


## Requirement Compliance Notes

### PAVI-CORE-COMPRESSION-003 — Pixel Sampling in FID Mode
> *"The design of the compression core assumes that pixel data shall be sampled in FID mode."*

**Status for NZ=1 (current verified baseline):** 🟢 Compliant — NZ=1 FID collapses to raster order;
the current RTL ingests data in FID mode with zero reorder buffer cost.

**Status for NZ=50 (Option A / Option C, C-slow multi-band):** ⚠️ Architecture transition required.

The Tsigkanos et al. (B-1, Virtex-5QV) paper demonstrates that **BIP ordering is the optimal input
format for C-slow retiming** — it provides NZ–1 pipeline stages of slack in the weight-update feedback
loop, eliminating the need for the FID diagonal traversal entirely for multi-band operation. Under BIP:
- All spatial neighbours (N, W, NW, NE) are delivered by the 4-FIFO spectral slice buffer (2 BRAM18s)
- All spectral neighbours (d(z-1)) are available in the Tsigkanos local-diff vector (2,850 bits LUTRAM)
- No inter-iteration pipeline stall occurs — same real-time guarantee as FID, different mechanism

The **BIL→BIP reorder module** (`bil_to_bip_reorder.v`, PING buffer = 1,600 Kbits = 89 BRAM18s)
replaces the FID diagonal traversal for multi-band. This satisfies the **spirit** of PAVI-CORE-COMPRESSION-003
(real-time, no pipeline stall) while adopting the architecturally superior BIP ordering proven in the
Tsigkanos paper for NZ ≥ 14.

> **Action required:** Update PAVI-CORE-COMPRESSION-003 wording to reflect BIP as the
> multi-band ingestion mode, with FID retained as the NZ=1 single-band baseline.

---

### PAVI-CORE-COMPRESSION-004 — On-Board Near-Lossless Compression
> *"The second implementation of compression core shall focus on the lossy variant of CCSDS-123.0-B-2 standard."*

**Status:** 🔵 Phase 2 — not yet designed. Planned after lossless (PAVI-CORE-COMPRESSION-001) is verified.

Near-lossless in CCSDS-123.0-B-2 adds:
- **Absolute error bound parameter `A`** — controls the maximum reconstruction error per sample
- **Quantization** of prediction residuals by factor `(2A+1)` before entropy coding
- **Sample representative `s̃`** — the reconstructed sample `s̃ = clip(s_hat + dequantized_residual, 0, 2^D-1)`
  replaces `s_raw` in the predictor feedback path. Specifically:
  - Row buffers store `s̃` (not `s_raw`) as the spatial neighbour for future pixels
  - Weight update uses `s̃` (not `s_raw`) — this ensures the compressor and decompressor
    use identical prediction state, since the decompressor only has access to `s̃`
- The predictor weight-update feedback loop **structure** is unchanged — C-slow retiming
  applies identically to near-lossless mode. Only the data source changes (mux from `s_raw` to `s̃`)
- The encoder (Option A sample-adaptive or Option C block-adaptive) is unchanged;
  it receives mapped residuals regardless of lossless vs near-lossless

**RTL impact:** Quantizer (near 200 LUTs), de-quantizer (near 100 LUTs), sample reconstruction + clip ( near 100 LUTs),
and a mux on the row buffer write path and weight update input. No additional BRAM or DSP.
The `A` parameter register controls the quantization factor; when `A=0`, near-lossless degenerates to lossless.

---

## Options Considered and Dismissed

### Option B — C-slow Predictor (BIP) → Block-Adaptive Encoder (raw BIP, no demux)

| Aspect | Detail |
|---|---|
| **Architecture** | Predictor outputs BIP residuals. Block-adaptive encoder consumes them directly in BIP order — no per-band demux FIFO. |
| **Block formation** | With NZ=50 and J=64, each block contains 64 consecutive BIP samples — i.e. samples from 50+ different bands mixed together. The encoder picks ONE coding option for all mixed-band samples. |
| **Why dismissed** | BIP ordering destroys intra-block correlation. At NZ=8 J=8, each block has exactly 1 sample per band — the worst case for block-adaptive code selection. Green Book data shows **0.3–0.6 bps degradation** vs sample-adaptive (5.2–5.5 bps at J=8 BIP vs 4.91 bps sample-adaptive on set2 data). This is the **worst compression option** of all four. The small FIFO + scheduling FSM cost of Option C's per-band demux fully recovers this penalty — Option B saves nothing meaningful while sacrificing compression. |

### Option D — C-slow Predictor (BIP) → B-2 Hybrid Entropy Coder (per-band accumulators)

| Aspect | Detail |
|---|---|
| **Architecture** | The CCSDS-123.0-B-2 hybrid entropy coder maintains per-band accumulators Ã_z regardless of encoding order, then applies block-level coding decisions using per-band statistics. Per-band accumulator feedback has NZ×J cycles of slack — naturally relaxed, no C-slow needed in encoder. |
| **Advantages** | Architecturally the cleanest B-2 option. Designed for exactly this BIP + multi-band use case. Better compression than raw BIP block-adaptive (Option B) because per-band statistics are maintained. |
| **Why dismissed** | Most complex encoder to implement — full hybrid coder requires: 16 code option evaluators, active-prefix register management, per-band accumulator memory (NZ entries), and a tail-generation FSM that has no equivalent in the current sample-adaptive encoder RTL. The implementation effort is significantly higher than Option A (which reuses the existing encoder structure with C-slow register banking) or Option C (which uses a simpler block-adaptive encoder). The compression benefit over Option C with per-band demux is marginal (<0.1 bps). Not justified given the RTL complexity and verification cost. |

> **Summary:** Options B and D were considered during architectural analysis. Option B was rejected for poor compression under BIP ordering. Option D was rejected for excessive RTL complexity with marginal compression benefit. Options A and C represent the two viable design points — A prioritises compression and GSS compatibility, C prioritises clock frequency and encoder simplicity.
