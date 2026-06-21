# NVMe SSD Architecture Methods — ZU17EG
## Sensor-to-SSD Write & SSD Read Paths

---

## System Parameters

**Current board:**

| Parameter | Value |
|---|---|
| SSDs | 2× NVMe M.2 (current board max) |
| SSD type | Samsung 970 Evo Plus 1TB class (consumer) |
| SSD sustained write | ~1.7 GB/s post-SLC-cache per SSD |
| 2-SSD combined write | ~3.4 GB/s sustained |
| 2-SSD combined read | ~3.3 GB/s per SSD = ~6.6 GB/s combined |
| PCIe | Gen3 x4 per SSD (3.94 GB/s theoretical each) |
| PL DDR4 | 1 interface, 8 GB, DDR4-2400 x64 (~19.2 GB/s) |
| PS DDR4 | 1 interface, 8 GB (hardened PS8 controller) |
| Processor | Quad A53 APU (1.5 GHz) + Dual R5F RPU (600 MHz) |

**Serviceable sensor data rate with 2 SSDs: ~3.4 GB/s**
(sensor bands may be selectively discarded to match this rate)

**Using both PL-DDR4 and PS-DDR4, with 3 SSDs: ~5.1 GB/s**
This is discussed below, after the 2 SSD discussion is complete.
**Search for  "3-SSD Configurations: PL DDR4 + PS DDR4"**

---

## Two Independent Data Paths

**PATH A — Sensor → Memory (XDMA NOT involved):**
Sensor PL logic writes to BRAM/URAM or DDR4 directly. This is a PL-internal write. XDMA plays no role.

**PATH B — Memory → SSD (XDMA IS involved):**
SSD's internal DMA engine issues PCIe Memory Read TLPs that travel through XDMA's AXI-PCIe bridge (M_AXI port) to reach BRAM/URAM or DDR4 and pull data out.

These are two independent AXI master paths targeting the same memory. The processor (A53 or R5F) is involved in neither data path — it only writes tiny 64-byte queue entries and 32-bit doorbells.

---

## METHOD 1a: Linux O_DIRECT + BRAM/URAM buffer

```
                         ZYNQ PS (A53 Linux)
                    ┌────────────────────────────────┐
                    │  Yocto Linux                   │
                    │  NVMe driver + O_DIRECT         │
                    │  libaio, io_submit()            │
                    │                                 │
                    │  Writes 64B SQEs to PS DDR4     │
                    │  Rings 32-bit doorbells (MMIO)  │
                    │  Polls 16B CQEs                 │
                    │                                 │
                    │  ~5-8% of 1 A53 core @3.4 GB/s  │
                    └──────────┬──────────────────────┘
                               │ AXI (control only, KB/s)
                               │ S_AXI_LITE + S_AXI_B
 ╔═════════════════════════════│═════════════════════════════════════╗
 ║                     FPGA PL FABRIC                                ║
 ║                             │                                     ║
 ║                             ▼                                     ║
 ║                 ┌──────────────────────┐                          ║
 ║                 │   AXI Interconnect   │ (processor → XDMA        ║
 ║                 │   (control plane)    │  register access)        ║
 ║                 └───┬──────────────┬───┘                          ║
 ║                     │              │                              ║
 ║                     ▼              ▼                              ║
 ║    ┌─────────────────────┐  ┌─────────────────────┐               ║
 ║    │       XDMA 1        │  │       XDMA 2        │               ║
 ║    │    Root Port Mode   │  │    Root Port Mode   │               ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │               ║
 ║    │ │  AXI-PCIe       │ │  │ │  AXI-PCIe       │ │               ║
 ║    │ │  Bridge         │ │  │ │  Bridge         │ │               ║
 ║    │ │                 │ │  │ │                 │ │               ║
 ║    │ │ S_AXI ◄── ctrl  │ │  │ │ S_AXI ◄── ctrl  │ │               ║
 ║    │ │ M_AXI ──► data  │ │  │ │ M_AXI ──► data  │ │               ║
 ║    │ └─────────────────┘ │  │ └─────────────────┘ │               ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │               ║
 ║    │ │ PCIe PHY (GTH)  │ │  │ │ PCIe PHY (GTH)  │ │               ║
 ║    │ │ Gen3 x4 lanes   │ │  │ │ Gen3 x4 lanes   │ │               ║
 ║    │ └────────┬────────┘ │  │ └────────┬────────┘ │               ║
 ║    └──┬───────│──────────┘  └──┬───────│──────────┘               ║
 ║       │ M_AXI │                │ M_AXI │                          ║
 ║       │       │                │       │                          ║
 ║       ▼       │                ▼       │                          ║
 ║    ┌──────────────────────────────┐    │                          ║
 ║    │     AXI SmartConnect         │    │                          ║
 ║    │     (data plane)             │    │                          ║
 ║    │                              │    │                          ║
 ║    │  SSD1 DMA + SSD2 DMA engines │    │                          ║
 ║    │  both read from BRAM/URAM    │    │                          ║
 ║    │  via this path               │    │                          ║
 ║    └──────────────┬───────────────┘    │                          ║
 ║                   │                    │                          ║
 ║                   ▼                    │                          ║
 ║    ┌───────────────────────────┐       │                          ║
 ║    │     BRAM / URAM           │       │                          ║
 ║    │     (circular buffer)     │       │                          ║
 ║    │     ~2-8 MB               │       │                          ║
 ║    │                           │       │                          ║
 ║    │  Write port ◄── Sensor PL │       │                          ║
 ║    │                 logic     │       │                          ║
 ║    │                           │       │                          ║
 ║    │  Read port ──► SSD DMA    │       │                          ║
 ║    │                reads via  │       │                          ║
 ║    │                XDMA M_AXI │       │                          ║
 ║    └───────────────────────────┘       │                          ║
 ║                                        │                          ║
 ║    Sensor ──► PL Logic ──► BRAM/URAM   │                          ║
 ║    (GTH)    (metadata,    (direct PL   │                          ║
 ║              band routing) write, no   │                          ║
 ║                            MIG needed) │                          ║
 ╚════════════════════════════════════════│══════════════════════════╝
                                          │
                              ┌───────────┴───────────┐
                              ▼                       ▼
                    ┌──────────────────┐     ┌──────────────────┐
                    │      SSD 1       │     │      SSD 2       │
                    │ ┌──────────────┐ │     │ ┌──────────────┐ │
                    │ │  NVMe        │ │     │ │  NVMe        │ │
                    │ │  Controller  │ │     │ │  Controller  │ │
                    │ │ ┌──────────┐ │ │     │ │ ┌──────────┐ │ │
                    │ │ │DMA Engine│ │ │     │ │ │DMA Engine│ │ │
                    │ │ └──────────┘ │ │     │ │ └──────────┘ │ │
                    │ └──────────────┘ │     │ └──────────────┘ │
                    │ ┌──────────────┐ │     │ ┌──────────────┐ │
                    │ │  NAND Flash  │ │     │ │  NAND Flash  │ │
                    │ │  (TLC)       │ │     │ │  (TLC)       │ │
                    │ │  BOTTLENECK  │ │     │ │  BOTTLENECK  │ │
                    │ └──────────────┘ │     │ └──────────────┘ │
                    │  ~1.7 GB/s sust. │     │  ~1.7 GB/s sust. │
                    └──────────────────┘     └──────────────────┘
```

**Write data flow (sensor → SSD):**

1. Sensor PL logic writes data into BRAM/URAM  *(PL direct write)*
2. A53 writes SQE with PRP pointing to BRAM address  *(tiny MMIO)*
3. SSD DMA engine reads data from BRAM via XDMA M_AXI  *(bulk data)*

**Read data flow (SSD → readback):**

Needs DDR4 — BRAM/URAM too small for multi-GB session readback.

1. A53 writes SQE with PRP → PL DDR4 address
2. SSD DMA engine writes data to PL DDR4 via XDMA M_AXI → MIG
3. PL reads from PL DDR4 via MIG

**Limits:**

- BRAM/URAM capacity: ~2–8 MB (shared with other PL functions)
- At 3.4 GB/s combined write: buffer turns over every ~0.6–2.4 ms
- Zero tolerance for SSD stalls (no large buffer for burst absorption)
- Best suited for lower data rates or initial testing

---

## METHOD 1b: Linux O_DIRECT + PL DDR4 buffer

```
                         ZYNQ PS (A53 Linux)
                    ┌────────────────────────────────-┐
                    │  Yocto Linux                    │
                    │  NVMe driver + O_DIRECT         │
                    │  libaio, io_submit()            │
                    │                                 │
                    │  Writes 64B SQEs to PS DDR4     │
                    │  Rings 32-bit doorbells (MMIO)  │
                    │  Polls 16B CQEs                 │
                    │                                 │
                    │  ~5-8% of 1 A53 core @3.4 GB/s  │
                    └──────────┬──────────────────────┘
                               │ AXI (control only, KB/s)
 ╔═════════════════════════════│══════════════════════════════════════╗
 ║                     FPGA PL FABRIC                                 ║
 ║                             │                                      ║
 ║                 ┌───────────┴──────────────────┐                   ║
 ║                 │      AXI Interconnect        │                   ║
 ║                 │      (control plane)         │                   ║
 ║                 └───┬──────────────────────┬───┘                   ║
 ║                     │                      │                       ║
 ║                     ▼                      ▼                       ║
 ║    ┌─────────────────────┐  ┌─────────────────────┐                ║
 ║    │       XDMA 1        │  │       XDMA 2        │                ║
 ║    │    Root Port Mode   │  │    Root Port Mode   │                ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │                ║
 ║    │ │  AXI-PCIe       │ │  │ │  AXI-PCIe       │ │                ║
 ║    │ │  Bridge         │ │  │ │  Bridge         │ │                ║
 ║    │ │ S_AXI ◄── ctrl  │ │  │ │ S_AXI ◄── ctrl  │ │                ║
 ║    │ │ M_AXI ──► data  │ │  │ │ M_AXI ──► data  │ │                ║
 ║    │ └─────────────────┘ │  │ └─────────────────┘ │                ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │                ║
 ║    │ │ PCIe PHY (GTH)  │ │  │ │ PCIe PHY (GTH)  │ │                ║
 ║    │ │ Gen3 x4 lanes   │ │  │ │ Gen3 x4 lanes   │ │                ║
 ║    │ └────────┬────────┘ │  │ └────────┬────────┘ │                ║
 ║    └──┬───────│──────────┘  └──┬───────│──────────┘                ║
 ║       │ M_AXI │                │ M_AXI │                           ║
 ║       │       │                │       │                           ║
 ║       ▼       │                ▼       │                           ║
 ║    ┌──────────────────────────────┐    │                           ║
 ║    │     AXI SmartConnect         │    │                           ║
 ║    │     (data plane + clk xing)  │    │                           ║
 ║    │                              │    │                           ║
 ║    │  AXI Masters:                │    │                           ║
 ║    │   • Sensor PL AXI Master     │    │                           ║
 ║    │     (writes sensor data IN)  │    │                           ║
 ║    │   • XDMA 1 M_AXI             │    │                           ║
 ║    │   • XDMA 1 M_AXI             │    │                           ║
 ║    │     (SSD1 DMA reads OUT)     │    │                           ║
 ║    │   • XDMA 2 M_AXI             │    │                           ║
 ║    │     (SSD2 DMA reads OUT)     │    │                           ║
 ║    │                              │    │                           ║
 ║    │  AXI Slave:                  │    │                           ║
 ║    │   • MIG S_AXI                │    │                           ║
 ║    └──────────────┬───────────────┘    │                           ║
 ║                   │                    │                           ║
 ║                   ▼                    │                           ║
 ║            ┌─────────────┐             │                           ║
 ║            │     MIG     │             │                           ║
 ║            │  Controller │             │                           ║
 ║            └──────┬──────┘             │                           ║
 ║                   │                    │                           ║
 ║    Sensor ──► PL Logic ──► AXI Master  │                           ║
 ║    (GTH)    (metadata,     writes to   │                           ║
 ║              band routing) MIG directly│                           ║
 ║                            (no XDMA)   │                           ║
 ╚═══════════════════│════════════════════│═══════════════════════════╝
                     ▼                    │
               ┌──────────┐   ┌───────────┴───────────┐
               │ PL DDR4  │   ▼                       ▼
               │  8 GB    │  ┌──────────────────┐  ┌──────────────────┐
               └──────────┘  │      SSD 1       │  │      SSD 2       │
                    ▲        │ ┌──────────────┐ │  │ ┌──────────────┐ │
                    │        │ │  NVMe Ctrl   │ │  │ │  NVMe Ctrl   │ │
                    │        │ │ ┌──────────┐ │ │  │ │ ┌──────────┐ │ │
                    │        │ │ │DMA Engine│ │ │  │ │ │DMA Engine│ │ │
                    │        │ │ └──────────┘ │ │  │ │ └──────────┘ │ │
                    │        │ └──────────────┘ │  │ └──────────────┘ │
                    │        │ ┌──────────────┐ │  │ ┌──────────────┐ │
                    │        │ │  NAND (TLC)  │ │  │ │  NAND (TLC)  │ │
                    │        │ │  BOTTLENECK  │ │  │ │  BOTTLENECK  │ │
                    │        │ └──────────────┘ │  │ └──────────────┘ │
                    │        │  ~1.7 GB/s sust. │  │  ~1.7 GB/s sust. │
                    │        └──────────────────┘  └──────────────────┘
                    │
                    │  MIG bandwidth budget:
                    │    Sensor IN:     up to 3.4 GB/s  (from PL)
                    │    SSD1 DMA OUT:  ~1.7 GB/s       (via XDMA1 M_AXI)
                    │    SSD2 DMA OUT:  ~1.7 GB/s       (via XDMA2 M_AXI)
                    │    ─────────────────────────────────
                    │    Total:         ~6.8 GB/s bidirectional
                    │    MIG capacity:  ~19.2 GB/s (DDR4-2400 x64)
                    │    Utilization:   ~35%    very comfortable
```

**Write data flow (sensor → SSD):**

1. **Sensor PL → AXI Master → SmartConnect → MIG → PL DDR4** — PL writes sensor data IN (XDMA not involved)
2. A53 writes SQE with PRP → PL DDR4 buffer address *(tiny MMIO)*
3. **SSD DMA → PCIe → XDMA M_AXI → SmartConnect → MIG → PL DDR4** — SSD DMA engine READS data OUT from PL DDR4

**Read data flow (SSD → readback):**

1. A53 writes SQE with PRP → PL DDR4 address
2. **SSD DMA → XDMA M_AXI → SmartConnect → MIG → PL DDR4** — SSD DMA engine WRITES data INTO PL DDR4
3. PL logic reads from PL DDR4 via MIG

**Notes:**
Current board has 1 PL DDR4 interface. Future boards should target 2 independent PL DDR4 interfaces for bandwidth isolation between sensor write and SSD DMA read.

---

## METHOD 2a: R5F Baremetal + BRAM/URAM buffer

```
                         ZYNQ PS
          ┌───────────────────────────────────────────────────┐
          │                                                   │
          │  A53 APU (Yocto Linux)      R5F RPU (baremetal)   │
          │  ┌────────────────────┐    ┌───────────────────┐  │
          │  │ Mission management │    │ Shane Colton's    │  │
          │  │ Telemetry          │    │ NVMe driver       │  │
          │  │ Downlink control   │    │ (ported to R5F)   │  │
          │  │                    │    │                   │  │
          │  │ 0% SSD work        │    │ Constructs SQEs   │  │
          │  │                    │    │ Rings doorbells   │  │
          │  │ Sends commands:    │    │ Polls CQEs        │  │
          │  │ "start/stop        │    │                   │  │
          │  │  session"        ──┼────┤ No O_DIRECT needed│  │
          │  │ via OpenAMP/RPMsg  │    │ (no Linux on R5F) │  │
          │  │                    │    │                   │  │
          │  │                    │    │ ~8-13% of 1 R5F   │  │
          │  │                    │    │  core @3.4 GB/s   │  │
          │  └────────────────────┘    └─────────┬─────────┘  │
          └──────────────────────────────────────┼────────────┘
                                                 │ AXI (control, KB/s)
 ╔═══════════════════════════════════════════════│═════════════════╗
 ║                     FPGA PL FABRIC            │                 ║
 ║                             ┌─────────────────┘                 ║
 ║                             ▼                                   ║
 ║                 ┌──────────────────────┐                        ║
 ║                 │   AXI Interconnect   │                        ║
 ║                 │   (R5F → XDMA ctrl)  │                        ║
 ║                 └───┬──────────────┬───┘                        ║
 ║                     ▼              ▼                            ║
 ║    ┌─────────────────────┐  ┌─────────────────────┐             ║
 ║    │       XDMA 1        │  │       XDMA 2        │             ║
 ║    │    Root Port Mode   │  │    Root Port Mode   │             ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │             ║
 ║    │ │  AXI-PCIe       │ │  │ │  AXI-PCIe       │ │             ║
 ║    │ │  Bridge         │ │  │ │  Bridge         │ │             ║
 ║    │ │ S_AXI ◄── ctrl  │ │  │ │ S_AXI ◄── ctrl  │ │             ║
 ║    │ │ M_AXI ──► data  │ │  │ │ M_AXI ──► data  │ │             ║
 ║    │ └─────────────────┘ │  │ └─────────────────┘ │             ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │             ║
 ║    │ │ PCIe PHY (GTH)  │ │  │ │ PCIe PHY (GTH)  │ │             ║
 ║    │ │ Gen3 x4 lanes   │ │  │ │ Gen3 x4 lanes   │ │             ║
 ║    │ └────────┬────────┘ │  │ └────────┬────────┘ │             ║
 ║    └──┬───────│──────────┘  └──┬───────│──────────┘             ║
 ║       │ M_AXI │                │ M_AXI │                        ║
 ║       ▼       │                ▼       │                        ║
 ║    ┌──────────────────────────────┐    │                        ║
 ║    │     AXI SmartConnect         │    │                        ║
 ║    └──────────────┬───────────────┘    │                        ║
 ║                   ▼                    │                        ║
 ║    ┌───────────────────────────┐       │                        ║
 ║    │     BRAM / URAM           │       │                        ║
 ║    │     (circular buffer)     │       │                        ║
 ║    │     ~2-8 MB               │       │                        ║
 ║    │                           │       │                        ║
 ║    │  Write ◄── Sensor PL      │       │                        ║
 ║    │  Read  ──► SSD DMA        │       │                        ║
 ║    │            via XDMA M_AXI │       │                        ║
 ║    └───────────────────────────┘       │                        ║
 ║                                        │                        ║
 ║    Sensor ──► PL Logic ──► BRAM/URAM   │                        ║
 ║    (GTH)    (metadata)                 │                        ║
 ║                        ┌───────────────┘                        ║
 ║              PL ──IRQ──► R5F                                    ║
 ║              "buffer full, write to SSD"                        ║
 ╚═════════════════════════════════════════════════════════════════╝
                              │
                   ┌──────────┴──────────┐
                   ▼                     ▼
          ┌──────────────────┐  ┌──────────────────┐
          │      SSD 1       │  │      SSD 2       │
          │ ┌──────────────┐ │  │ ┌──────────────┐ │
          │ │  NVMe Ctrl   │ │  │ │  NVMe Ctrl   │ │
          │ │ ┌──────────┐ │ │  │ │ ┌──────────┐ │ │
          │ │ │DMA Engine│ │ │  │ │ │DMA Engine│ │ │
          │ │ └──────────┘ │ │  │ │ └──────────┘ │ │
          │ └──────────────┘ │  │ └──────────────┘ │
          │ ┌──────────────┐ │  │ ┌──────────────┐ │
          │ │ NAND (TLC)   │ │  │ │ NAND (TLC)   │ │
          │ │ BOTTLENECK   │ │  │ │ BOTTLENECK   │ │
          │ └──────────────┘ │  │ └──────────────┘ │
          │  ~1.7 GB/s sust. │  │  ~1.7 GB/s sust. │
          └──────────────────┘  └──────────────────┘
```

**Real-time write loop (no A53 involvement):**

1. PL sensor logic writes data into BRAM/URAM
2. PL asserts IRQ to R5F: "buffer N is full"
3. R5F constructs NVMe SQE with PRP → BRAM/URAM address
4. R5F rings doorbell via MMIO (XDMA S_AXI_LITE)
5. SSD DMA engine reads from BRAM/URAM via XDMA M_AXI
6. R5F polls CQE for completion
7. R5F signals PL: "buffer N drained, reuse"

**Control (infrequent):**
A53 Linux → OpenAMP/RPMsg → R5F: "start session".
R5F → OpenAMP/RPMsg → A53: "session complete".

---

## METHOD 2b: R5F Baremetal + PL DDR4 buffer

```
                         ZYNQ PS
          ┌──────────────────────────────────────────────────┐
          │                                                  │
          │  A53 APU (Yocto Linux)      R5F RPU (baremetal)  │
          │  ┌────────────────────┐    ┌───────────────────┐ │
          │  │ Mission management │    │ Shane Colton's    │ │
          │  │ Telemetry          │    │ NVMe driver       │ │
          │  │ Downlink control   │    │ (ported to R5F)   │ │
          │  │                    │    │                   │ │
          │  │ 0% SSD work        │    │ Constructs SQEs   │ │
          │  │                    │    │ Rings doorbells   │ │
          │  │ IPC to R5F:        │    │ Polls CQEs        │ │
          │  │ "start/stop      ──┼────┤                   │ │
          │  │  session"          │    │ No O_DIRECT needed│ │
          │  │ via OpenAMP/RPMsg  │    │ (no Linux on R5F) │ │
          │  │                    │    │                   │ │
          │  │                    │    │ ~8-13% of 1 R5F   │ │
          │  │                    │    │  core @3.4 GB/s   │ │
          │  └────────────────────┘    └─────────┬─────────┘ │
          └──────────────────────────────────────┼───────────┘
                                                 │ AXI (control, KB/s)
 ╔═══════════════════════════════════════════════│════════════════╗
 ║                     FPGA PL FABRIC            │                ║
 ║                             ┌─────────────────┘                ║
 ║                             ▼                                  ║
 ║                 ┌──────────────────────┐                       ║
 ║                 │   AXI Interconnect   │                       ║
 ║                 │   (R5F → XDMA ctrl)  │                       ║
 ║                 └───┬──────────────┬───┘                       ║
 ║                     ▼              ▼                           ║
 ║    ┌─────────────────────┐  ┌─────────────────────┐            ║
 ║    │       XDMA 1        │  │       XDMA 2        │            ║
 ║    │    Root Port Mode   │  │    Root Port Mode   │            ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │            ║
 ║    │ │  AXI-PCIe       │ │  │ │  AXI-PCIe       │ │            ║
 ║    │ │  Bridge         │ │  │ │  Bridge         │ │            ║
 ║    │ │ S_AXI ◄── ctrl  │ │  │ │ S_AXI ◄── ctrl  │ │            ║
 ║    │ │ M_AXI ──► data  │ │  │ │ M_AXI ──► data  │ │            ║
 ║    │ └─────────────────┘ │  │ └─────────────────┘ │            ║
 ║    │ ┌─────────────────┐ │  │ ┌─────────────────┐ │            ║
 ║    │ │ PCIe PHY (GTH)  │ │  │ │ PCIe PHY (GTH)  │ │            ║
 ║    │ │ Gen3 x4 lanes   │ │  │ │ Gen3 x4 lanes   │ │            ║
 ║    │ └────────┬────────┘ │  │ └────────┬────────┘ │            ║
 ║    └──┬───────│──────────┘  └──┬───────│──────────┘            ║
 ║       │ M_AXI │                │ M_AXI │                       ║
 ║       ▼       │                ▼       │                       ║
 ║    ┌──────────────────────────────┐    │                       ║
 ║    │     AXI SmartConnect         │    │                       ║
 ║    │     (data plane + clk xing)  │    │                       ║
 ║    │                              │    │                       ║
 ║    │  AXI Masters:                │    │                       ║
 ║    │   • Sensor PL AXI Master     │    │                       ║
 ║    │     (writes sensor data IN)  │    │                       ║
 ║    │   • XDMA 1 M_AXI             │    │                       ║
 ║    │     (SSD1 DMA reads OUT)     │    │                       ║
 ║    │   • XDMA 2 M_AXI             │    │                       ║
 ║    │     (SSD2 DMA reads OUT)     │    │                       ║
 ║    │                              │    │                       ║
 ║    │  AXI Slave: MIG S_AXI        │    │                       ║
 ║    └──────────────┬───────────────┘    │                       ║
 ║                   ▼                    │                       ║
 ║            ┌─────────────┐             │                       ║
 ║            │     MIG     │             │                       ║
 ║            │  Controller │             │                       ║
 ║            └──────┬──────┘             │                       ║
 ║                   │                    │                       ║
 ║    Sensor ──► PL Logic ──► AXI Master  │                       ║
 ║    (GTH)    (metadata)     writes to   │                       ║
 ║                            MIG directly│                       ║
 ║                            (no XDMA)   │                       ║
 ║                        ┌───────────────┘                       ║
 ║              PL ──IRQ──► R5F                                   ║
 ║              "buffer full, write to SSD"                       ║
 ╚═══════════════════│════════════════════════════════════════════╝
                     ▼
               ┌──────────┐    ┌───────────┴───────────┐
               │ PL DDR4  │    ▼                       ▼
               │  8 GB    │  ┌──────────────────┐  ┌──────────────────┐
               └──────────┘  │      SSD 1       │  │      SSD 2       │
                    ▲        │ ┌──────────────┐ │  │ ┌──────────────┐ │
                    │        │ │  NVMe Ctrl   │ │  │ │  NVMe Ctrl   │ │
                    │        │ │ ┌──────────┐ │ │  │ │ ┌──────────┐ │ │
                    │        │ │ │DMA Engine│ │ │  │ │ │DMA Engine│ │ │
                    │        │ │ └──────────┘ │ │  │ │ └──────────┘ │ │
                    │        │ └──────────────┘ │  │ └──────────────┘ │
                    │        │ ┌──────────────┐ │  │ ┌──────────────┐ │
                    │        │ │  NAND (TLC)  │ │  │ │  NAND (TLC)  │ │
                    │        │ │  BOTTLENECK  │ │  │ │  BOTTLENECK  │ │
                    │        │ └──────────────┘ │  │ └──────────────┘ │
                    │        │  ~1.7 GB/s sust. │  │  ~1.7 GB/s sust. │
                    │        └──────────────────┘  └──────────────────┘
                    │
                    │  MIG bandwidth budget @3.4 GB/s write:
                    │    Sensor IN:     ~3.4 GB/s  (from PL)
                    │    SSD1 DMA OUT:  ~1.7 GB/s  (via XDMA1 M_AXI)
                    │    SSD2 DMA OUT:  ~1.7 GB/s  (via XDMA2 M_AXI)
                    │    ─────────────────────────────────
                    │    Total:         ~6.8 GB/s bidirectional
                    │    MIG capacity:  ~19.2 GB/s
                    │    Utilization:   ~35%    very comfortable
```

**Real-time write loop (no A53 involvement):**

1. PL sensor logic fills buffer in PL DDR4 via MIG
2. PL asserts IRQ to R5F: "buffer N is full"
3. R5F constructs NVMe SQE with PRP → PL DDR4 buffer address
4. R5F rings doorbell via MMIO (XDMA S_AXI_LITE)
5. SSD DMA engine reads from PL DDR4 via XDMA M_AXI → SmartConnect → MIG
6. R5F polls CQE for completion
7. R5F signals PL: "buffer N drained, reuse"

**Vivado address map (R5F 32-bit addressing, from Shane Colton):**

| Region | Address | Size | Purpose |
|---|---|---|---|
| XDMA1 S_AXI_LITE | `0xA000_0000` | 64 MB | via M_AXI_HPM0_FPD |
| XDMA1 S_AXI_B | `0xA800_0000` | 1 MB | BAR0 / NVMe regs SSD1 |
| XDMA2 S_AXI_LITE | `0xB000_0000` | 64 MB | |
| XDMA2 S_AXI_B | `0xB800_0000` | 1 MB | BAR0 / NVMe regs SSD2 |
| NVMe queues | PS DDR4 `0x1000_0000` | KB | SQE/CQE buffers |
| Data buffers | PL DDR4 via MIG range | GB | Bulk sensor data |

---

## METHOD 3: PL NVMe Hardware IP (NVMeCHA-style) — not recommended

```
 ╔═════════════════════════════════════════════════════════════════╗
 ║                     FPGA PL FABRIC                              ║
 ║                                                                 ║
 ║  Sensor ──► PL Logic ──► Custom NVMe Host Controller (RTL)      ║
 ║  (GTH)                  ┌──────────────────────────────────┐    ║
 ║                         │  Admin Queue Manager      (RTL)  │    ║
 ║                         │  I/O Queue Manager(s)     (RTL)  │    ║
 ║                         │  PRP List Builder         (RTL)  │    ║
 ║                         │  Completion Parser        (RTL)  │    ║
 ║                         │  Error Recovery           (RTL)  │    ║
 ║                         │  PCIe TLP Packetizer      (RTL)  │    ║
 ║                         │  + FW on MicroBlaze for init     │    ║
 ║                         │                                  │    ║
 ║                         │  Per-SSD resources (see table):  │    ║
 ║                         │   ~50K LUTs, ~62K FFs, ~116 BRAM │    ║
 ║                         └──────────────┬───────────────────┘    ║
 ║                                        │ GTH Gen3 x4            ║
 ╚════════════════════════════════════════│════════════════════════╝
                                          ▼
                                ┌──────────────────┐
                                │      SSD         │
                                │ ┌──────────────┐ │
                                │ │  NVMe Ctrl   │ │
                                │ │ ┌──────────┐ │ │
                                │ │ │DMA Engine│ │ │
                                │ │ └──────────┘ │ │
                                │ └──────────────┘ │
                                │ ┌──────────────┐ │
                                │ │ NAND (TLC)   │ │
                                │ │ BOTTLENECK   │ │
                                │ └──────────────┘ │
                                │  ~1.7 GB/s sust. │  ◄── SAME speed
                                └──────────────────┘      as all other
                                                          methods
```

- **Effort:** 6–12 months, 2–3 RTL engineers + 1–2 FW engineers
- **Speed:** Identical to Methods 1 & 2 (SSD NAND is the bottleneck)
- **Advantage:** Sub-10μs per-command latency (irrelevant for sequential I/O)
- **Cost:** Massive PL resource consumption, ongoing maintenance

---

## Comparison Summary

```
                          │ Method 1       │ Method 2       │ Method 3
                          │ Linux O_DIRECT │ R5F Baremetal  │ PL NVMe IP
──────────────────────────┼────────────────┼────────────────┼───────────────
A53 core usage @3.4 GB/s  │ ~5-8% of 1     │ 0%             │ 0%
R5F core usage @3.4 GB/s  │ 0%             │ ~8-13% of 1    │ 0%
PL resource cost per SSD  │ XDMA only      │ XDMA only      │ ~50K LUT+
SSD write speed           │ SSD-limited    │ SSD-limited    │ SSD-limited
Development effort        │ Weeks (SW)     │ Weeks (port)   │ 6-12 months
Risk                      │ Low (proven)   │ Low (proven)   │ High
Filesystem support        │ Yes            │ No (raw block) │ No
Error recovery            │ Linux kernel   │ Custom         │ Custom
Multi-SSD pipelining      │ Linux native   │ Code extension │ RTL extension
Existing code/reference   │ Xilinx, 3 projs│ Shane Colton   │ NVMeCHA
```

**Buffer sub-options:**

- **a) BRAM/URAM:** no DDR4 in write path, but small capacity (~2–8 MB)
- **b) PL DDR4:** large capacity (8 GB), burst absorption, ~35% MIG util.
- **c) PS DDR4:** fallback if PL DDR4 unavailable, uses HP/HPC ports

Method and buffer choice depends on sensor data rate to be serviced, burst absorption requirements, and whether A53 involvement is acceptable (even at 5–8%).

---

## Scaling to Higher Data Rates

For sustained write rates beyond what 2 consumer SSDs can deliver (3.4 GB/s):

**Target 5 GB/s sustained write:**

- Consumer SSDs (~1.5–1.7 GB/s each): 3–4 SSDs needed
- Enterprise SSDs (~2.5+ GB/s each): 2–3 SSDs needed — e.g. Kioxia CD8, Samsung PM9A3, Micron 7450 MAX

**Target 5 GB/s requires future board changes:**

- More PCIe root port connections (more XDMA instances, more GTH quads)
- Possibly 2 independent PL DDR4 interfaces for bandwidth isolation
- R5F manages more NVMe queue sets (1 per SSD)

SSD count drives PL resource usage (see resource table below).

---

## ZU17EG PL Resource Utilization: Software vs NVMeCHA

**ZU17EG approximate total PL resources:**
LUT: ~243,000 — FF: ~486,000 — BRAM: ~1,080 (36 Kb each) — URAM: ~80

### Software approach (Methods 1 & 2): XDMA IP only, per SSD

XDMA per SSD instance: LUT ~34,000 / FF ~34,000 / BRAM 47

```
                    │  LUT      │  FF       │  BRAM
                    │  (of 243K)│  (of 486K)│  (of 1080)
────────────────────┼───────────┼───────────┼───────────
1 SSD (1x XDMA)    │  34K (14%)│  34K  (7%)│   47  (4%)
2 SSDs (2x XDMA)   │  68K (28%)│  68K (14%)│   94  (9%)
3 SSDs (3x XDMA)   │ 102K (42%)│ 102K (21%)│  141 (13%)
4 SSDs (4x XDMA)   │ 136K (56%)│ 136K (28%)│  188 (17%)
```

### NVMeCHA approach (Method 3): Full NVMe controller per SSD

NVMeCHA per SSD (XDMA + Admin Ctrl + 2× I/O Ctrl, minimal config): LUT ~50,000 / FF ~62,000 / BRAM ~116

```
                    │  LUT      │  FF       │  BRAM
                    │  (of 243K)│  (of 486K)│  (of 1080)
────────────────────┼───────────┼───────────┼───────────
1 SSD (1x NVMeCHA) │  50K (21%)│  62K (13%)│  116 (11%)
2 SSDs (2x NVMeCHA)│ 100K (41%)│ 124K (26%)│  232 (21%)
3 SSDs (3x NVMeCHA)│ 150K (62%)│ 186K (38%)│  348 (32%)
4 SSDs (4x NVMeCHA)│ 200K (82%)│ 248K (51%)│  464 (43%)
```

### Additional cost of NVMeCHA over software (wasted resources)

```
                    │  Extra LUT │  Extra FF  │  Extra BRAM
                    │  wasted    │  wasted    │  wasted
────────────────────┼────────────┼────────────┼────────────
1 SSD               │  +16K  (7%)│  +28K  (6%)│  +69  (6%)
2 SSDs              │  +32K (13%)│  +56K (12%)│ +138 (13%)
3 SSDs              │  +48K (20%)│  +84K (17%)│ +207 (19%)
4 SSDs              │  +64K (26%)│ +112K (23%)│ +276 (26%)
```

These extra resources are consumed for NVMe protocol processing in RTL — work that the SSD's own firmware and a ~500-line C driver already handle. The speed result is identical because the bottleneck is SSD NAND, not NVMe protocol overhead.

Every BRAM consumed by NVMeCHA is a BRAM unavailable for sensor data FIFOs, BRAM/URAM circular buffers, or other PL processing pipelines.

---
---
---

## 3-SSD Configurations: PL DDR4 + PS DDR4

**3-SSD target:** 3 × 1.7 GB/s = ~5.1 GB/s sustained write

**How sensor data reaches PS DDR4:**
Sensor data arrives in PL fabric via GTH receivers. PL DDR4 is accessed via MIG (PL-internal, no PS involvement). PS DDR4 is accessed from PL via HP (High Performance) AXI slave ports.

Sensor stream for SSD3: Sensor GTH → PL logic (metadata, band routing) → AXI Master (PL) → S_AXI_HP port → PS Interconnect → PS DDR4. This HP port path is standard Xilinx PL-to-PS data transfer.

Shane Colton's proven architecture already uses PS DDR4 + PL XDMA. His entire system routes SSD DMA traffic through: XDMA (PL) M_AXI → HP port → PS DDR4. This is not a new or untested path.

**PCIe options for the 3rd SSD:**

- **Option A:** 3rd XDMA instance in PL (Gen3 x4, ~3.94 GB/s theoretical). Uses PL GTH lanes. Same IP as SSD1/SSD2.
- **Option B:** PS hardened PCIe (Gen2 x4, ~2.0 GB/s theoretical). Uses PS GTR lanes. No PL XDMA needed. But PS GTR lanes may be allocated to USB3/DP/SATA on board. Gen2 is slower but still exceeds 1.7 GB/s SSD sustained.

---

### CONFIG C1: 3 SSDs, 3x PL XDMA, all managed by R5F

```
                         ZYNQ PS
          ┌───────────────────────────────────────────────────-───┐
          │                                                       │
          │  A53 APU (Yocto Linux)        R5F RPU (baremetal)     │
          │  ┌────────────────────┐      ┌─────────────────────┐  │
          │  │ Mission management │      │ Shane Colton's      │  │
          │  │ Telemetry          │      │ NVMe driver         │  │
          │  │ Downlink control   │      │ (ported to R5F)     │  │
          │  │                    │      │                     │  │
          │  │ 0% SSD work        │      │ Manages ALL 3 SSDs  │  │
          │  │                    │      │                     │  │
          │  │ IPC to R5F:        │      │ XDMA1 → SSD1        │  │
          │  │ "start/stop      ──┼──────┤ XDMA2 → SSD2        │  │
          │  │  session"          │      │ XDMA3 → SSD3        │  │
          │  │ via OpenAMP/RPMsg  │      │                     │  │
          │  │                    │      │ ~12-20% of 1 R5F    │  │
          │  │                    │      │  core @5.1 GB/s     │  │
          │  └────────────────────┘      └──────────┬──────────┘  │
          │                                         │             │
          │           PS DDR4                       │ AXI (ctrl)  │
          │         ┌────────┐                      │             │
          │         │ 8 GB   │                      │             │
          │         └───▲──▲─┘                      │             │
          │  ┌──────────│──│────────────────────────│─────────┐   │
          │  │  S_AXI_HP0  S_AXI_HP1                │         │   │
          │  │  (sensor    (XDMA3 M_AXI             │         │   │
          │  │   stream 3   reaches PS DDR4         │         │   │
          │  │   from PL)   for SSD3 DMA)           │         │   │
          │  └──────────────────────────────────────│─────────┘   │
          └─────────────────────────────────────────│─────────────┘
                                                    │
 ╔══════════════════════════════════════════════════│══════════════╗
 ║                     FPGA PL FABRIC               │              ║
 ║                                                  ▼              ║
 ║                                      ┌──────────────────┐       ║
 ║                                      │ AXI Interconnect │       ║
 ║                                      │ (R5F → XDMAs)    │       ║
 ║                                      └─┬──────┬──────┬──┘       ║
 ║                                        ▼      ▼      ▼          ║
 ║                                  ┌────────┐┌────────┐┌────────┐ ║
 ║                                  │ XDMA 1 ││ XDMA 2 ││ XDMA 3 │ ║
 ║                                  │┌──────┐││┌──────┐││┌──────┐│ ║
 ║                                  ││Bridge││││Bridge││││Bridge││ ║
 ║                                  │└──────┘││└──────┘││└──────┘│ ║
 ║                                  │┌──────┐││┌──────┐││┌──────┐│ ║
 ║                                  ││ GTH  ││││ GTH  ││││ GTH  ││ ║
 ║                                  ││Gen3x4││││Gen3x4││││Gen3x4││ ║
 ║                                  │└──┬───┘││└──┬───┘││└──┬───┘│ ║
 ║                                  └─┬─│────┘└─┬─│────┘└─┬─│────┘ ║
 ║                               M_AXI│ │  M_AXI│ │  M_AXI│ │      ║
 ║                                    │ │       │ │       │ │      ║
 ║  XDMA1+XDMA2 M_AXI ──► SmartConnect ──► MIG  │ │       │ │      ║
 ║  (SSD1+SSD2 DMA read from PL DDR4) │ │       │ │       │ │      ║
 ║                                    │ │       │ │       │ │      ║
 ║  XDMA3 M_AXI ──► S_AXI_HP1 ──► PS DDR4       │ │       │ │      ║
 ║  (SSD3 DMA reads from PS DDR4)     │ │       │ │       │ │      ║
 ║                                    │ │       │ │       │ │      ║
 ║  Sensor ──► PL Logic               │ │       │ │       │ │      ║
 ║  (GTH)    (metadata, band routing) │ │       │ │       │ │      ║
 ║    │                               │ │       │ │       │ │      ║
 ║    ├── streams 1+2 ──► AXI Master ──► SmartConnect ──► MIG      ║
 ║    │   (writes to PL DDR4)         │ │       │ │       │ │      ║
 ║    │                               │ │       │ │       │ │      ║
 ║    └── stream 3 ──► AXI Master ──► S_AXI_HP0 ──► PS DDR4        ║
 ║        (writes to PS DDR4 via HP port)                          ║
 ╚═══════════════│════════════════════│═│═══════│═│════════════════╝
                 ▼                    │ │       │ │
           ┌──────────┐               │ │       │ │
           │ PL DDR4  │               ▼ ▼       ▼ ▼
           │  8 GB    │          ┌───────┐ ┌───────┐ ┌───────┐
           └──────────┘          │ SSD 1 │ │ SSD 2 │ │ SSD 3 │
                                 │┌─────┐│ │┌─────┐│ │┌─────┐│
            SSD1+SSD2 DMA        ││ DMA ││ ││ DMA ││ ││ DMA ││
            read from PL DDR4    │└─────┘│ │└─────┘│ │└─────┘│
            via XDMA M_AXI→MIG   │┌─────┐│ │┌─────┐│ │┌─────┐│
                                 ││NAND ││ ││NAND ││ ││NAND ││
            SSD3 DMA reads       │└─────┘│ │└─────┘│ │└─────┘│
            from PS DDR4         │1.7GB/s│ │1.7GB/s│ │1.7GB/s│
            via XDMA M_AXI→HP1   └───────┘ └───────┘ └───────┘
```

**Bandwidth budget:**

- **PL DDR4 (MIG):** ~3.4 GB/s in (sensor 1+2) + ~3.4 GB/s out (SSD1+SSD2) = ~6.8 GB/s = 35% of 19.2 GB/s ✓
- **PS DDR4 (HP):** ~1.7 GB/s in (sensor 3 via HP0) + ~1.7 GB/s out (SSD3 via HP1) = ~3.4 GB/s = 18% of 19.2 GB/s ✓ — remaining ~15.8 GB/s free for A53 Linux

**HP ports used:** 2 of 6 (HP0: sensor stream 3 in, HP1: XDMA3 M_AXI out)
**PL resources:** 3× XDMA = ~102K LUT (42%), 141 BRAM (13%)

---

### CONFIG C2: 3 SSDs, 3x PL XDMA, split R5F + A53

```
                         ZYNQ PS
          ┌──────────────────────────────────────────────────────-┐
          │                                                       │
          │  A53 APU (Yocto Linux)        R5F RPU (baremetal)     │
          │  ┌────────────────────┐      ┌─────────────────────┐  │
          │  │ Mission management │      │ Shane Colton's      │  │
          │  │ Telemetry          │      │ NVMe driver         │  │
          │  │ Downlink control   │      │ (ported to R5F)     │  │
          │  │                    │      │                     │  │
          │  │ ALSO manages SSD3  │      │ Manages SSD1 + SSD2 │  │
          │  │ via Linux O_DIRECT │      │ (PL DDR4 path)      │  │
          │  │ on PS DDR4         │      │                     │  │
          │  │                    │      │ ~8-13% of 1 R5F     │  │
          │  │ Standard Linux     │      │  core @3.4 GB/s     │  │
          │  │ NVMe path — zero  ─┼──────┤                     │  │
          │  │ custom DMA config  │      │ IPC from A53:       │  │
          │  │ needed for SSD3    │      │ "start/stop session"│  │
          │  │                    │      │                     │  │
          │  │ ~3-5% of 1 A53     │      │                     │  │
          │  │  core for SSD3     │      │                     │  │
          │  └─────────┬──────────┘      └──────────┬──────────┘  │
          │            │                            │             │
          │            │ A53 manages SSD3           │ R5F manages │
          │            │ queues + data in PS DDR4   │ SSD1+SSD2   │
          │            ▼                 ┌──────────┘             │
          │       PS DDR4                │ AXI (ctrl)             │
          │     ┌────────┐               │                        │
          │     │ 8 GB   │               │                        │
          │     └──▲──▲──┘               │                        │
          │  ┌─────│──│──────────────────│────────────────────┐   │
          │  │ S_AXI  S_AXI              │                    │   │
          │  │ _HP0   _HP1               │                    │   │
          │  │(sensor (XDMA3             │                    │   │
          │  │ str 3)  M_AXI)            │                    │   │
          │  └───────────────────────────│────────────────────┘   │
          └──────────────────────────────│────────────────────────┘
                                         │
 ╔═══════════════════════════════════════│═══════════════════════╗
 ║                  FPGA PL FABRIC       │                       ║
 ║                                       ▼                       ║
 ║                           ┌──────────────────┐                ║
 ║                           │ AXI Interconnect │                ║
 ║                           │(R5F→XDMA1/2,     │                ║
 ║                           │ A53→XDMA3)       │                ║
 ║                           └─┬──────┬──────┬──┘                ║
 ║                             ▼      ▼      ▼                   ║
 ║                       ┌────────┐┌────────┐┌────────┐          ║
 ║                       │ XDMA 1 ││ XDMA 2 ││ XDMA 3 │          ║
 ║                       │┌──────┐││┌──────┐││┌──────┐│          ║
 ║                       ││Bridge││││Bridge││││Bridge││          ║
 ║                       │└──────┘││└──────┘││└──────┘│          ║
 ║                       │┌──────┐││┌──────┐││┌──────┐│          ║
 ║                       ││ GTH  ││││ GTH  ││││ GTH  ││          ║
 ║                       ││Gen3x4││││Gen3x4││││Gen3x4││          ║
 ║                       │└──┬───┘││└──┬───┘││└──┬───┘│          ║
 ║                       └─┬─│────┘└─┬─│────┘└─┬─│────┘          ║
 ║                    M_AXI│ │  M_AXI│ │  M_AXI│ │               ║
 ║                         │ │       │ │       │ │               ║
 ║  XDMA1+2 M_AXI ──► SmartConnect ──► MIG     │ │               ║
 ║                         │ │       │ │       │ │               ║
 ║  XDMA3 M_AXI ──────────-│─│───────│─│───────┘ │               ║
 ║    ──► S_AXI_HP1 ──► PS DDR4                  │               ║
 ║                         │ │       │ │         │               ║
 ║  Sensor ──► PL Logic    │ │       │ │         │               ║
 ║  (GTH)    (metadata)    │ │       │ │         │               ║
 ║    │                    │ │       │ │         │               ║
 ║    ├── str 1+2 ──► AXI Master ──► SmartConnect ──► MIG        ║
 ║    │   (PL DDR4)        │ │       │ │         │               ║
 ║    │                    │ │       │ │         │               ║
 ║    └── str 3 ──► AXI Master ──► S_AXI_HP0 ──► PS DDR4         ║
 ║        (PS DDR4 via HP port)                                  ║
 ╚═══════════════════│═════════════════════│═════════════════════╝
                     ▼                     │
               ┌──────────┐    ┌───────────┴───────────┐
               │ PL DDR4  │    ▼           ▼           ▼
               │  8 GB    │  ┌───────┐ ┌───────┐ ┌───────┐
               └──────────┘  │ SSD 1 │ │ SSD 2 │ │ SSD 3 │
                             │┌─────┐│ │┌─────┐│ │┌─────┐│
                             ││ DMA ││ ││ DMA ││ ││ DMA ││
                             │└─────┘│ │└─────┘│ │└─────┘│
                             │┌─────┐│ │┌─────┐│ │┌─────┐│
                             ││NAND ││ ││NAND ││ ││NAND ││
                             │└─────┘│ │└─────┘│ │└─────┘│
                             │1.7GB/s│ │1.7GB/s│ │1.7GB/s│
                             └───────┘ └───────┘ └───────┘
                               ▲ R5F     ▲ R5F     ▲ A53
                               │manages  │manages  │manages
                               │PL DDR4  │PL DDR4  │PS DDR4
                               │buffers  │buffers  │buffers
```

**Why this split is clean:**

**R5F + PL DDR4 + XDMA1/2:** Custom baremetal path. R5F constructs PRP → PL DDR4 addresses. Data stays PL-internal (sensor → MIG → PL DDR4 → XDMA M_AXI → SSD).

**A53 + PS DDR4 + XDMA3:** Standard Linux NVMe path. THIS IS Shane Colton's exact proven architecture. Linux NVMe driver allocates DMA buffers in PS DDR4 by default. PRP entries naturally point to PS DDR4. Zero custom DMA config. Sensor data for SSD3 enters PS DDR4 via HP0 (standard PL→PS transfer). SSD3 DMA reads from PS DDR4 via XDMA3 M_AXI → HP1.

Each processor manages SSDs on "its own" DDR4. No cross-domain buffer management complexity.

**HP ports used:** 2 of 6 (HP0: sensor stream 3 in, HP1: XDMA3 M_AXI out)

---

### CONFIG C3: 2x PL XDMA + PS hardened PCIe Gen2 (no 3rd XDMA)

```
                         ZYNQ PS
          ┌───────────────────────────────────────────────────-───┐
          │                                                       │
          │  A53 APU (Yocto Linux)        R5F RPU (baremetal)     │
          │  ┌────────────────────┐      ┌─────────────────────┐  │
          │  │ Mission management │      │ Shane Colton's      │  │
          │  │                    │      │ NVMe driver         │  │
          │  │ ALSO manages SSD3  │      │ Manages SSD1 + SSD2 │  │
          │  │ via Linux NVMe     │      │ (PL DDR4 path)      │  │
          │  │ on PS PCIe Gen2    │      │                     │  │
          │  │                  ──┼──────┤ ~8-13% of 1 R5F     │  │
          │  │ PS PCIe uses       │      │  core @3.4 GB/s     │  │
          │  │ PS GTR lanes       │      │                     │  │
          │  │ (not PL GTH)       │      │                     │  │
          │  │                    │      │                     │  │
          │  │ ~3-5% of 1 A53     │      │                     │  │
          │  │  core for SSD3     │      │                     │  │
          │  └─────────┬──────────┘      └──────────┬──────────┘  │
          │            │                            │             │
          │    ┌───────┘                 ┌──────────┘             │
          │    │                         │                        │
          │    ▼                         │ AXI (ctrl)             │
          │  ┌──────────────────┐        │                        │
          │  │ PS Hardened PCIe │        │                        │
          │  │ (inside PS8)     │        │                        │
          │  │ Gen2 x4          │        │                        │
          │  │ via PS GTR lanes │        │                        │
          │  │                  │        │                        │
          │  │ SSD3 DMA reaches │        │                        │
          │  │ PS DDR4 directly │        │                        │
          │  │ via PS internal  │        │                        │
          │  │ interconnect     │        │                        │
          │  │ (no HP port      │        │                        │
          │  │  needed for      │        │                        │
          │  │  SSD3 DMA)       │        │                        │
          │  └────────┬─────────┘        │                        │
          │           │ PS GTR           │                        │
          │           │ Gen2 x4          │                        │
          │           │ (~2 GB/s max)    │                        │
          │           │                  │                        │
          │    PS DDR4│                  │                        │
          │  ┌────────┤                  │                        │
          │  │ 8 GB   │                  │                        │
          │  └──▲─────┘                  │                        │
          │  ┌──│────────────────────────│────────────────────┐   │
          │  │  S_AXI_HP0                │                    │   │
          │  │  (sensor stream 3         │                    │   │
          │  │   from PL into PS DDR4)   │                    │   │
          │  │                           │                    │   │
          │  │  NOTE: only 1 HP port     │                    │   │
          │  │  needed. SSD3 DMA does    │                    │   │
          │  │  NOT use an HP port —     │                    │   │
          │  │  PS PCIe accesses PS DDR4 │                    │   │
          │  │  through PS internal bus  │                    │   │
          │  └───────────────────────────│────────────────────┘   │
          └──────────────────────────────│────────────────────────┘
                                         │
 ╔═══════════════════════════════════════│═══════════════════════╗
 ║                  FPGA PL FABRIC       │                       ║
 ║                                       ▼                       ║
 ║                           ┌──────────────────┐                ║
 ║                           │ AXI Interconnect │                ║
 ║                           │ (R5F → XDMA1/2)  │                ║
 ║                           └───┬──────────┬───┘                ║
 ║                               ▼          ▼                    ║
 ║                         ┌────────┐ ┌────────┐                 ║
 ║                         │ XDMA 1 │ │ XDMA 2 │  only 2 XDMAs   ║
 ║                         │┌──────┐│ │┌──────┐│  in PL          ║
 ║                         ││Bridge││ ││Bridge││                 ║
 ║                         │└──────┘│ │└──────┘│                 ║
 ║                         │┌──────┐│ │┌──────┐│                 ║
 ║                         ││ GTH  ││ ││ GTH  ││                 ║
 ║                         ││Gen3x4││ ││Gen3x4││                 ║
 ║                         │└──┬───┘│ │└──┬───┘│                 ║
 ║                         └─┬─│────┘ └─┬─│────┘                 ║
 ║                      M_AXI│ │   M_AXI│ │                      ║
 ║                           │ │        │ │                      ║
 ║  XDMA1+2 M_AXI ──► SmartConnect ──► MIG                       ║
 ║                           │ │        │ │                      ║
 ║  Sensor ──► PL Logic      │ │        │ │                      ║
 ║  (GTH)    (metadata)      │ │        │ │                      ║
 ║    │                      │ │        │ │                      ║
 ║    ├── str 1+2 ──► AXI Master ──► SmartConnect ──► MIG        ║
 ║    │   (PL DDR4)          │ │        │ │                      ║
 ║    │                      │ │        │ │                      ║
 ║    └── str 3 ──► AXI Master ──► S_AXI_HP0 ──► PS DDR4         ║
 ║        (PS DDR4 via HP port — sensor data crosses PL→PS)      ║
 ╚═══════════════════│═══════│═│════════│═│══════════════════════╝
                     ▼       │ │        │ │
               ┌──────────┐  ▼ ▼        ▼ ▼
               │ PL DDR4  │ ┌───────┐ ┌───────┐       ┌───────┐
               │  8 GB    │ │ SSD 1 │ │ SSD 2 │       │ SSD 3 │
               └──────────┘ │ Gen3  │ │ Gen3  │       │ Gen2  │
                            │via GTH│ │via GTH│       │via GTR│
                            │┌─────┐│ │┌─────┐│       │┌─────┐│
                            ││ DMA ││ ││ DMA ││       ││ DMA ││
                            │└─────┘│ │└─────┘│       │└─────┘│
                            │┌─────┐│ │┌─────┐│       │┌─────┐│
                            ││NAND ││ ││NAND ││       ││NAND ││
                            │└─────┘│ │└─────┘│       │└─────┘│
                            │1.7GB/s│ │1.7GB/s│       │1.7GB/s│
                            └───────┘ └───────┘       └───────┘
                              R5F       R5F             A53
                              PL DDR4   PL DDR4         PS DDR4
```

**Sensor data → PS DDR4 path (for SSD3):**
Sensor GTH → PL logic (band routing) → AXI Master (PL) → S_AXI_HP0 → PS Interconnect → PS DDR4. This is the ONLY HP port consumed in this config. SSD3's DMA engine reaches PS DDR4 through the PS hardened PCIe controller's internal path — no HP port needed for DMA.

**Bandwidth budget:**

- **PL DDR4 (MIG):** ~3.4 GB/s in + ~3.4 GB/s out = ~6.8 GB/s = 35% ✓
- **PS DDR4:** ~1.7 GB/s in (sensor via HP0) + ~1.7 GB/s out (SSD3 DMA) = ~3.4 GB/s = 18% ✓

**Advantages over C1/C2:**

- Only 2 XDMA instances in PL (saves ~34K LUT, 34K FF, 47 BRAM)
- Only 1 HP port consumed (SSD3 DMA uses PS internal bus, not HP)
- SSD3 path entirely within PS — A53 Linux NVMe works natively
- PS hardened PCIe is silicon, not PL fabric

**Disadvantages:**

- PS GTR lanes must be available (check board: USB3/DP/SATA allocation)
- Gen2 x4 max ~2 GB/s (vs Gen3 ~3.94 GB/s) — less burst headroom. Still OK: 2 GB/s > 1.7 GB/s SSD sustained write
- PS PCIe config is different from XDMA (device tree, not Vivado IP)

---

### Config Comparison: 3-SSD Options

```
                            │ Config C1       │ Config C2       │ Config C3
                            │ All R5F         │ R5F+A53 split   │ R5F+A53+PS PCIe
────────────────────────────┼─────────────────┼─────────────────┼─────────────────
SSDs managed by R5F         │ 3 (all)         │ 2 (SSD1+SSD2)   │ 2 (SSD1+SSD2)
SSDs managed by A53         │ 0               │ 1 (SSD3)        │ 1 (SSD3)
A53 core usage              │ 0%              │ ~3-5% of 1 core │ ~3-5% of 1 core
R5F core usage              │ ~12-20% of 1    │ ~8-13% of 1     │ ~8-13% of 1
XDMA instances in PL        │ 3               │ 3               │ 2
PL GTH quads for PCIe       │ 3               │ 3               │ 2
PS GTR lanes needed         │ 0               │ 0               │ 4 (Gen2 x4)
HP ports consumed           │ 2               │ 2               │ 1
  HP for sensor data in     │ 1 (HP0)         │ 1 (HP0)         │ 1 (HP0)
  HP for SSD3 DMA           │ 1 (HP1)         │ 1 (HP1)         │ 0 (PS internal)
SSD3 PCIe generation        │ Gen3            │ Gen3            │ Gen2
SSD3 data buffer location   │ PS DDR4         │ PS DDR4         │ PS DDR4
How sensor fills PS DDR4    │ PL→HP0→PS DDR4  │ PL→HP0→PS DDR4  │ PL→HP0→PS DDR4
How SSD3 DMA reaches DDR4   │ XDMA3→HP1→PSDDR│ XDMA3→HP1→PSDDR│ PS PCIe internal
Custom DMA config for SSD3  │ PRP→PS DDR4 addr│ None (standard) │ None (standard)
PL LUT cost (XDMAs only)    │ ~102K (42%)     │ ~102K (42%)     │ ~68K (28%)
PL BRAM cost (XDMAs only)   │ 141 (13%)       │ 141 (13%)       │ 94 (9%)
```

**Note:** In ALL configs, sensor data for SSD3 reaches PS DDR4 the same way: PL sensor logic → AXI Master → S_AXI_HP0 → PS DDR4. The difference is only how SSD3's DMA engine reaches PS DDR4: C1/C2 via XDMA3 M_AXI → S_AXI_HP1 (PL→PS path, 2nd HP port), C3 via PS hardened PCIe → PS internal bus (no HP port).

---

### Shane Colton's Architecture — Where It Appears in Each Config

```
Shane Colton's proven, tested, working architecture:

  Processor              PL                    External
  (baremetal)       ┌──────────┐
       │            │          │
       │  SQE/bell  │  XDMA    │    GTH      ┌──────┐
       │───────────►│ ┌──────┐ │───Gen3x4───►│ SSD  │
       │            │ │Bridge│ │             │      │
       │            │ └──────┘ │◄────────────│(DMA) │
       │            │  M_AXI   │  PCIe TLPs  └──────┘
       │            └────┬─────┘
       │                 │
       │                 ▼ HP port
       │            ┌──────────┐
       │  CQE poll  │ PS DDR4  │ ◄── SSD DMA reads/writes here
       │◄───────────│ (data    │
       │            │ +queues) │
       │            └──────────┘
```

**Where this exact path appears:**

- **Config C1:** SSD3 channel (R5F → XDMA3 → HP1 → PS DDR4)
- **Config C2:** SSD3 channel (A53 → XDMA3 → HP1 → PS DDR4) — *this is the most standard version: A53 Linux + PS DDR4*
- **Config C3:** SSD3 channel (A53 → PS PCIe → PS DDR4) — variant; uses PS hardened PCIe instead of PL XDMA, but same DDR4 path

**What's NEW compared to Shane (for SSD1+SSD2):**
Shane: XDMA M_AXI → HP port → PS DDR4.
You: XDMA M_AXI → SmartConnect → MIG → PL DDR4.
The only change is the M_AXI destination — from HP port to MIG. The XDMA IP, NVMe driver, and SSD interaction are identical.

---
---
---
---

### Filesystem Options: Raw Block vs FatFs

Both Method 1 (Linux O_DIRECT) and Method 2 (R5F Baremetal) support filesystem and raw block writes. Shane Colton's repo includes FatFs (ChaN's ultra-light FAT filesystem in C, ~27 KB) already integrated with his NVMe driver via `diskio.c`.

#### Raw Block Writes

Processor writes directly to SSD LBA offsets. No filesystem overhead. PL-embedded metadata headers (sync word, band ID, timestamp, CRC) make data self-describing. A small index table at a reserved LBA range tracks session boundaries.

**Write speed:** maximum — no cluster allocation, no FAT table updates, no file open/close overhead. Just SQE construction + doorbell.

- For 1 SSD @128 KB blocks: ~13,300 NVMe cmds/sec → ~1.7 GB/s (SSD-limited, not CPU-limited)
- For 2 SSDs (PL DDR4): ~26,600 NVMe cmds/sec → ~3.4 GB/s combined
- For 3 SSDs (2 PL DDR4 + 1 PS DDR4): ~39,900 NVMe cmds/sec → ~5.1 GB/s combined

**Readback:** also maximum speed. Sequential LBA reads, no filesystem metadata lookups. Single-SSD read at ~3.3 GB/s (O_DIRECT or baremetal).

**Tradeoff:** SSD is NOT readable by a standard PC. Requires custom software to parse the raw blocks using the embedded headers/index. All readback must go through your R5F/A53 system.

#### FatFs (Baremetal Filesystem)

Shane Colton integrated FatFs with his NVMe driver. Writes sensor data as standard FAT32 or exFAT files. One file per capture session, or one file per band — application decides the file structure.

**Write speed:** slightly slower than raw blocks due to filesystem operations. Shane measured FatFs write speed on 970 Evo Plus 1TB:

- Sequential write to large files (1 GiB each): >1 GB/s sustained
- Overhead from file create/close is visible as brief speed dips
- exFAT significantly outperforms FAT32 (bitmap allocation vs FAT chain — 32× lower overhead for sequential writes)
- FAT32 limit: 4 GiB max file size, 2 TiB max drive (512B LBA)
- exFAT: no practical file size limit, but requires Microsoft license

**For 1 SSD with FatFs:** Still SSD-limited at ~1.5–1.7 GB/s sustained. FatFs overhead is small compared to the SSD NAND bottleneck. The filesystem metadata operations (cluster allocation, directory updates) add maybe 2–5% CPU overhead on top of raw NVMe queue management.

**For 2–3 SSDs with FatFs:** Each SSD independently runs at SSD-limited speed. FatFs manages separate file handles per SSD. CPU overhead scales linearly with SSD count but remains small (~5–15% of one R5F core for 3 SSDs). The main cost is file create/close pauses — avoid by pre-allocating large files or writing one large file per capture session.

**Readback:** SSD is a standard FAT32/exFAT volume. Plug into any PC, mount, and read files directly. No custom tools needed. This matters for ground station operations if SSDs are physically swapped. Read speed through FatFs: similar to raw — sequential file reads are essentially sequential LBA reads with minimal metadata overhead.

#### Linux Filesystem (Method 1 Only)

Linux supports ext4, xfs, exfat, fat32, and others natively. O_DIRECT bypasses page cache but still goes through the filesystem layer for block allocation and metadata. Overhead is slightly higher than FatFs due to kernel filesystem complexity, but negligible compared to the SSD NAND bottleneck.

**Advantage over FatFs:** mature, well-tested, journaling (ext4/xfs) provides crash recovery, supports arbitrarily large files.
**Disadvantage:** requires Linux, so only available on A53 APU path.

#### Recommendation

**For initial development and testing:** raw block writes. Simplest, fastest, eliminates filesystem as a variable when debugging.

**For production:** FatFs on R5F (or Linux filesystem on A53) if ground station needs to read SSDs directly. Raw blocks if all readback is through the satellite's own downlink path.

**Note:** raw block writes with PL-embedded metadata headers closely mirror CCSDS packet structures used in standard satellite missions. Ground station software that parses CCSDS-style packets can parse the raw SSD data with minimal adaptation.

---

### Comparison Summary — All Methods and Configurations

#### Processor + Buffer Combinations

##### 2-SSD Configurations (current board: 2 SSDs, 1 PL DDR4)

```
                          │ Method 1a      │ Method 1b      │ Method 2a      │ Method 2b
                          │ A53 Linux      │ A53 Linux      │ R5F Baremetal  │ R5F Baremetal
                          │ BRAM/URAM buf  │ PL DDR4 buf    │ BRAM/URAM buf  │ PL DDR4 buf
──────────────────────────┼────────────────┼────────────────┼────────────────┼────────────────
Processor for NVMe queues │ A53 (Linux)    │ A53 (Linux)    │ R5F (baremetal)│ R5F (baremetal)
A53 core usage @3.4 GB/s  │ ~5-8% of 1     │ ~5-8% of 1     │ 0%             │ 0%
R5F core usage @3.4 GB/s  │ 0%             │ 0%             │ ~8-13% of 1    │ ~8-13% of 1
Data buffer type          │ BRAM/URAM      │ PL DDR4 (MIG)  │ BRAM/URAM      │ PL DDR4 (MIG)
Buffer capacity           │ ~2-8 MB        │ 8 GB           │ ~2-8 MB        │ 8 GB
MIG utilization           │ N/A            │ ~35%           │ N/A            │ ~35%
HP ports consumed         │ 0              │ 0              │ 0              │ 0
Burst absorption          │ None (tight)   │ Large (GB)     │ None (tight)   │ Large (GB)
Filesystem support        │ Linux ext4/xfs │ Linux ext4/xfs │ FatFs or raw   │ FatFs or raw
XDMA instances in PL      │ 2              │ 2              │ 2              │ 2
PL LUT cost (XDMAs)       │ ~68K (28%)     │ ~68K (28%)     │ ~68K (28%)     │ ~68K (28%)
PL BRAM cost (XDMAs)      │ 94 (9%)        │ 94 (9%)        │ 94 (9%)        │ 94 (9%)
Development effort        │ Weeks (SW)     │ Weeks (SW)     │ Weeks (port)   │ Weeks (port)
Risk                      │ Low            │ Low            │ Low            │ Low
Proven by                 │ 3 ext. projects│ 3 ext. projects│ Shane Colton   │ Shane Colton
```

##### 3-SSD Configurations (adding PS DDR4 for 3rd SSD)

```
                          │ Config C1      │ Config C2      │ Config C3
                          │ All R5F        │ R5F + A53      │ R5F + A53
                          │ 3x PL XDMA    │ 3x PL XDMA    │ 2x PL XDMA
                          │                │                │ + PS PCIe Gen2
──────────────────────────┼────────────────┼────────────────┼────────────────
SSDs on PL DDR4 (MIG)     │ 2 (SSD1+SSD2)  │ 2 (SSD1+SSD2)  │ 2 (SSD1+SSD2)
SSDs on PS DDR4           │ 1 (SSD3)       │ 1 (SSD3)       │ 1 (SSD3)
SSD3 PCIe path            │ PL XDMA Gen3   │ PL XDMA Gen3   │ PS PCIe Gen2
Total sustained write     │ ~5.1 GB/s      │ ~5.1 GB/s      │ ~5.1 GB/s
                          │                │                │
Processor for SSD1+SSD2   │ R5F (baremetal)│ R5F (baremetal)│ R5F (baremetal)
Processor for SSD3        │ R5F (baremetal)│ A53 (Linux)    │ A53 (Linux)
A53 core usage            │ 0%             │ ~3-5% of 1     │ ~3-5% of 1
R5F core usage            │ ~12-20% of 1   │ ~8-13% of 1    │ ~8-13% of 1
                          │                │                │
Sensor→PS DDR4 path       │ PL→HP0→PS DDR4 │ PL→HP0→PS DDR4 │ PL→HP0→PS DDR4
SSD3 DMA→PS DDR4 path     │ XDMA3→HP1→PSDR│ XDMA3→HP1→PSDR│ PS PCIe internal
HP ports consumed         │ 2 (HP0+HP1)    │ 2 (HP0+HP1)    │ 1 (HP0 only)
PS GTR lanes needed       │ 0              │ 0              │ 4 (Gen2 x4)
                          │                │                │
MIG utilization           │ ~35%           │ ~35%           │ ~35%
PS DDR4 utilization       │ ~18%           │ ~18%           │ ~18%
PS DDR4 free for A53      │ ~15.8 GB/s     │ ~15.8 GB/s     │ ~15.8 GB/s
                          │                │                │
Filesystem (SSD1+SSD2)    │ FatFs or raw   │ FatFs or raw   │ FatFs or raw
Filesystem (SSD3)         │ FatFs or raw   │ Linux FS or raw│ Linux FS or raw
                          │                │                │
XDMA instances in PL      │ 3              │ 3              │ 2
PL LUT cost (XDMAs)       │ ~102K (42%)    │ ~102K (42%)    │ ~68K (28%)
PL BRAM cost (XDMAs)      │ 141 (13%)      │ 141 (13%)      │ 94 (9%)
                          │                │                │
Development effort        │ Medium         │ Medium         │ Medium
                          │ (3-SSD R5F     │ (2-SSD R5F     │ (2-SSD R5F
                          │  driver + IPC) │  + Linux SSD3  │  + PS PCIe cfg
                          │                │  + IPC)        │  + IPC)
                          │                │                │
SSD3 path proven by       │ Shane Colton   │ Shane Colton   │ ADI Kuiper
                          │ (PL XDMA +     │ (exact arch:   │ Linux example
                          │  PS DDR4)      │  A53 + XDMA +  │ (PS PCIe Gen2
                          │                │  PS DDR4)      │  + PS DDR4)
Risk                      │ Low            │ Low            │ Low-Medium
                          │                │                │ (PS GTR avail?)
```

##### METHOD 3: PL NVMe IP (NVMeCHA-style) — NOT RECOMMENDED

```
                          │ Method 3
                          │ PL NVMe IP
──────────────────────────┼────────────────
A53 core usage            │ 0%
R5F core usage            │ 0%
SSD write speed           │ SSD-limited (SAME as Methods 1 & 2)
PL LUT cost per SSD       │ ~50K (21%)  vs XDMA-only ~34K (14%)
PL BRAM cost per SSD      │ ~116 (11%)  vs XDMA-only 47 (4%)
Development effort        │ 6-12 months, multiple RTL + FW engineers
Risk                      │ High
Filesystem support        │ No (custom raw only)
Advantage                 │ Sub-10μs command latency
                          │ (irrelevant for sequential sensor I/O)
Proven by                 │ NVMeCHA (but ONLY with dummy transceivers,
                          │ NEVER tested with real SSDs)
```

**Notes on table:**

All methods achieve the same SSD-limited sustained write speed (~1.7 GB/s per consumer SSD post-SLC-cache). The differences are in processor involvement, PL resource cost, and development effort.

"Shane Colton's exact architecture" = PL XDMA (root port) + PS DDR4 for data buffers + baremetal NVMe driver. This is the baseline proven path. SSD1+SSD2 on PL DDR4 is a modification of this path (XDMA M_AXI routed to MIG instead of HP port). SSD3 on PS DDR4 IS the original path.

Config C2 is notable because SSD3's channel requires zero custom DMA configuration — Linux NVMe driver allocates buffers in PS DDR4 by default, and PRP entries naturally point there. It is the most standard, lowest-risk path for the third SSD.
