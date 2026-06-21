# NVMe SSD — AXI-PCIe Bridge Architecture & Shane Colton Codebase Reference

---
 XDMA AXI-PCIe Bridge Diagram

See attached PG194 Figure 1 (AMD/Xilinx) — High-Level Bridge Architecture for PCIe Gen3.
<img width="1168" height="1094" alt="image" src="https://github.com/user-attachments/assets/77711d3d-bfcd-4ffb-8990-183f1615d4dc" />
<img width="1017" height="640" alt="image" src="https://github.com/user-attachments/assets/2aba09c3-829b-43a0-8108-826594434b77" />

The three AXI interfaces and their roles in NVMe SSD operation:

| Interface | Vivado Port | Who Uses It | Traffic | Purpose |
|-----------|-------------|-------------|---------|---------|
| AXI4-Lite → Register Block | S_AXI(L)_CTL | Processor (R5F/A53) | Tiny, init only | PCIe link status, bridge config |
| AXI Slave → Slave Bridge | S_AXI(B) | Processor (R5F/A53) | Tiny, per-command | NVMe registers, doorbells, queue management |
| AXI Master → Master Bridge | M_AXI(B) | SSD's DMA engine | **ALL bulk data** | SSD DMA reads/writes memory through this path |

The processor only touches the top two interfaces — both carry tiny control traffic. ALL bulk data (GB/s) flows through the bottom Master Bridge, driven by the SSD's internal DMA engine, not the processor.

Reference documents:
- PG194: AXI Bridge for PCI Express Gen3 Subsystem v3.0 (what Shane uses — simpler, bridge only)
- PG195: DMA/Bridge Subsystem for PCIe (adds FPGA-side DMA engine — redundant for NVMe since SSD has its own DMA)


## 1. The AXI-PCIe Bridge: How XDMA Connects the Processor to SSDs

### What the XDMA IP Actually Is

The XDMA IP used in our NVMe SSD architecture is an **AXI Bridge for PCI Express Gen3** (documented in AMD/Xilinx PG194). It is NOT an NVMe controller. It is a pure protocol translator between AXI (the FPGA internal bus) and PCIe (the physical link to the SSD).

From PG194 Chapter 2:

> "The Bridge core is an interface between the AXI4 bus and PCI Express. The core contains the memory mapped AXI4 to AXI4-Stream Bridge and the AXI4-Stream Enhanced Interface Block for PCIe. The memory-mapped AXI4 to AXI4-Stream Bridge contains a register block and two functional half bridges, referred to as the Slave Bridge and Master Bridge. The slave bridge connects to the AXI4 Interconnect as a slave device to handle any issued AXI4 master read or write requests. The master bridge connects to the AXI4 interconnect as a master to process the PCIe generated read or write TLPs."

> "The Bridge core supports both Root Port and Endpoint configurations. When configured as a Root Port, the core supports up to two 32-bit or a single 64-bit PCIe BAR."

We use **Root Port** mode — the FPGA acts as the PCIe host, and the NVMe SSD acts as the PCIe endpoint device.


### PG194 Figure 1: High-Level Bridge Architecture

(See attached diagram: PG194 Figure 1 — "High-Level Bridge Architecture for the PCI Express Gen3 Architecture")

The XDMA Root Port contains three AXI interfaces:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         XDMA Root Port                              │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │  AXI4-Lite Interface → Register Block                        │   │
│  │  "Configuration and Control"                                 │   │
│  │  Vivado port: S_AXI(L)_CTL                                   │   │
│  │                                                              │   │
│  │  ACTIVE IN OUR DESIGN: Yes                                   │   │
│  │  WHO USES IT: Processor (R5F or A53)                         │   │
│  │  WHAT FOR:    PCIe link status check (PHY link up?)          │   │
│  │               Bridge enable/disable                          │   │
│  │               Root port configuration                        │   │
│  │  TRAFFIC:     Tiny — only during initialization              │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │  AXI Slave Interface → Slave Bridge                          │   │
│  │  "ZU+ PS Access to Device BAR0"                              │   │
│  │  Vivado port: S_AXI(B)                                       │   │
│  │                                                              │   │
│  │  ACTIVE IN OUR DESIGN: Yes                                   │   │
│  │  WHO USES IT: Processor (R5F or A53)                         │   │
│  │  WHAT FOR:    Access NVMe controller registers on the SSD    │   │
│  │               Write SQE doorbell (32-bit MMIO write)         │   │
│  │               Read CQE doorbell                              │   │
│  │               Read NVMe CAP, VS, CC, CSTS registers          │   │
│  │  TRAFFIC:     Tiny — 64-byte SQE + 32-bit doorbell per cmd   │   │
│  │                                                              │   │
│  │  This is how the processor "talks to" the SSD. The processor │   │
│  │  writes a 64-byte command (SQE) to memory, then writes the   │   │
│  │  SSD's doorbell register through THIS interface to tell the  │   │
│  │  SSD "new command ready."                                    │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │  AXI Master Interface → Master Bridge                        │   │  PCIe
│  │  "Device Access to ZU+ PS DDR"                               │   │  Rx ◄──
│  │  "(Queues and Data Transfer)"                                │   │
│  │  Vivado port: M_AXI(B)                                       │   │  PCIe
│  │                                                              │   │  Tx ──►
│  │  ACTIVE IN OUR DESIGN: Yes — THIS IS THE BULK DATA PATH      │   │
│  │  WHO USES IT: SSD's internal DMA engine (NOT the processor)  │   │
│  │  WHAT FOR:    SSD DMA reads data FROM memory (NVMe Write)    │   │
│  │               SSD DMA writes data TO memory (NVMe Read)      │   │
│  │               SSD DMA reads SQEs from queue memory           │   │
│  │               SSD DMA writes CQEs to queue memory            │   │
│  │  TRAFFIC:     ALL bulk data — every GB/s flows through here  │   │
│  │                                                              │   │
│  │  Shane Colton routes this to PS DDR4 (via HP ports).         │   │
│  │  Our design routes this to PL DDR4 (via MIG) for SSD1/SSD2   │   │
│  │  and to PS DDR4 (via HP port) for SSD3.                      │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  PCIe Gen3 Solution IP                                       │   │
│  │  GTH transceivers → physical PCIe lanes → M.2 → NVMe SSD     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Key insight:** The processor only touches the top two interfaces — both carry tiny control traffic (KB/s). ALL bulk data (GB/s) flows through the bottom Master Bridge (M_AXI), driven entirely by the SSD's internal DMA engine. The processor never touches the bulk data path.


### How the SSD's DMA Engine Works Through the Bridge

From Shane Colton's blog:

> "The AXI Master Interface is where *all* NVMe data transfer occurs, for both reads and writes. One way to look at it is that the drive itself contains the DMA engine, which issues memory reads and writes to the system (AXI) memory space through the bridge. The host requests that the drive perform these data transfers by submitting them to a queue, which is also contained in system memory and accessed through this interface."

> "The NVMe Controller, which is implemented on the drive itself, does most of the heavy lifting. The host only has to do some initialization and then maintain the queues and lists that control data transfers."


### NVMe Write Operation — Step by Step

```
1. Processor writes a 64-byte Submission Queue Entry (SQE) to memory
   SQE contains:
     - Opcode: 0x01 (NVMe Write)
     - PRP1:   physical address of data buffer in DDR4 (or BRAM/URAM)
     - SLBA:   starting Logical Block Address on SSD (destination)
     - NLB:    number of logical blocks to write

2. Processor writes SSD's doorbell register via Slave Bridge (S_AXI(B))
   "Hey SSD, there's a new command in the queue"

3. SSD's DMA engine reads the SQE from memory
   via Master Bridge (M_AXI) → through PCIe → reads from DDR4/BRAM

4. SSD's DMA engine reads the bulk data from memory
   via Master Bridge (M_AXI) → through PCIe → reads from DDR4/BRAM
   at the address specified in PRP1

5. SSD writes data to its internal NAND flash

6. SSD's DMA engine writes a 16-byte Completion Queue Entry (CQE) to memory
   via Master Bridge (M_AXI) → through PCIe → writes to DDR4
   "Command completed successfully"

7. Processor polls CQE memory location to detect completion
```

Steps 3, 4, 5, 6 are entirely performed by the SSD — the processor does nothing during these steps. The processor is only involved in steps 1, 2, and 7 (tiny control operations).


### PG194 vs PG195 — Which One We Use

| Document | IP Name | What It Provides | Our Usage |
|----------|---------|-------------------|-----------|
| PG194 | AXI Bridge for PCIe Gen3 | Pure AXI↔PCIe bridge. No FPGA-side DMA. | **This is what we use.** Shane Colton uses this. The SSD has its own DMA engine. |
| PG195 | DMA/Bridge Subsystem for PCIe | Bridge + FPGA-side DMA engine. | **Not needed.** The FPGA-side DMA engine would be redundant since the SSD already has DMA. Would waste PL resources. |

PG195's additional DMA engine is useful for non-NVMe PCIe devices that lack their own DMA capability. NVMe SSDs always have their own DMA engine — it is a fundamental part of the NVMe specification.

---

## 2. Shane Colton's Codebase Map

GitHub: https://github.com/coltonshane/SSD_Test

Blog: https://scolton.blogspot.com/2019/11/zynq-ultrascale-fatfs-with-bare-metal.html

Platform: Trenz TE0803 SoM (Zynq UltraScale+ XCZU4EV)

```
REPOSITORY FILE STRUCTURE
══════════════════════════

VIVADO / BUILD SCRIPTS
──────────────────────
  create_project.tcl       Vivado project: target xczu4ev, Trenz TE0803 board
  SSD_Test_TE0803.xdc      Pin constraints: GTH Quad 224 (PCIe), reset, UART
  create_platform.tcl      Vitis baremetal platform: psu_cortexa53_0, standalone
  create_projects.tcl      Vitis application project setup


FatFs GLUE LAYER — Bridges ChaN's FatFs to NVMe driver
──────────────────────────────────────────────────────
  diskio.c                 THE KEY INTEGRATION FILE
    disk_initialize()      Calls nvmeInit() to set up SSD
    disk_read()            Calls nvmeRead(), waits for all completions
    disk_write()           Calls nvmeWrite(), allows pipelining (slip)
    disk_ioctl(CTRL_SYNC)  Calls nvmeFlush(), waits for completion

    The pdrv argument (physical drive number) is passed to all
    disk_*() functions. Currently hardcoded to drive 0.
    For 3 SSDs: add switch(pdrv) routing to XDMA1/XDMA2/XDMA3.
    Set FF_VOLUMES=3 in ffconf.h.

  diskio.h                 Header for disk interface functions


FatFs LIBRARY — ChaN's ultra-light filesystem (~27 KB compiled)
──────────────────────────────────────────────────────────────
  ff.c                     Full FatFs implementation
    f_mount()              Mount a filesystem volume
    f_mkfs()               Format drive as FAT32 or exFAT
    f_open()               Open a file for reading or writing
    f_write()              Write data (internally calls disk_write)
    f_read()               Read data (internally calls disk_read)
    f_close()              Close file, flush metadata to SSD
    f_sync()               Flush file buffers to SSD

  ff.h                     FatFs API declarations
  ffconf.h                 Configuration file
    FF_VOLUMES = 1         Logical drives (change to 3 for 3 SSDs)
    FF_MAX_SS = 4096       Max sector size supported
    FF_FS_EXFAT = 1        exFAT support (requires Microsoft license)
    FF_USE_MKFS = 1        Enable f_mkfs() formatting function
  ffsystem.c               OS-layer functions (memory alloc, time)


MAIN APPLICATION — Speed tests
──────────────────────────────
  main.c
    main()                 Entry point
      pcieInit()           Initialize PCIe link (assert/deassert reset)
      nvmeInit()           Initialize NVMe SSD (full sequence below)
      diskWriteTest()      Raw sequential write speed test (no filesystem)
      diskReadTest()       Raw sequential read speed test
      fatFsWriteTest()     FatFs write speed test (writes to files)
      fatFsReadTest()      FatFs read speed test (reads from files)

    Constants:
      NVME_SLIP_ALLOWED=16   Max NVMe commands in flight (pipeline depth)
      data @ 0x20000000      Data buffer in PS DDR4 (512 MB offset)
      blockSize = 64*1024    64 KiB blocks (matches FAT32 cluster size)

    diskWriteTest():   Loops nvmeWrite() with slip control, measures MB/s
    diskReadTest():    Loops nvmeRead() with slip control, measures MB/s
    fatFsWriteTest():  f_open → f_write loop → f_close, measures MB/s
    fatFsReadTest():   f_open → f_read loop → f_close, measures MB/s


NVMe DRIVER — The core (~800 lines)
────────────────────────────────────
  nvme.c

    INITIALIZATION SEQUENCE (called by nvmeInit()):
      nvmeInitBridge()           Check PCIe PHY status (link up? Gen3 x4?)
                                 Verify SSD device class code
      nvmeInitAdminQueue()       Set up Admin SQ/CQ at 0x10000000 in PS DDR4
                                 Write queue base addresses to NVMe registers
      nvmeInitController()       Set CC.EN=1 (enable controller), wait CSTS.RDY
      nvmeIdentifyController()   Read controller identity: model, FW, capacity
      nvmeIdentifyNamespace()    Read namespace info: LBA count, block size
      nvmeCreateIOQueues()       Create I/O SQ/CQ at 0x10002000/0x10003000

    I/O OPERATIONS:
      nvmeWrite(srcByte, destLBA, numLBA)
        ★ THE WRITE FUNCTION ★
        Constructs 64-byte SQE:
          Opcode = 0x01 (NVMe Write)
          PRP1   = source memory address (where data lives in DDR4/BRAM)
          SLBA   = destination LBA on SSD
          NLB    = number of logical blocks
        Calls nvmeSubmitIOCommand() → writes SQE to I/O SQ, rings doorbell
        NON-BLOCKING: returns immediately (does not wait for SSD to finish)

      nvmeRead(destByte, srcLBA, numLBA)
        Same structure as nvmeWrite, but opcode 0x02 (NVMe Read)
        PRP1 = destination memory address (where to put data from SSD)
        Also NON-BLOCKING

      nvmeFlush()
        Opcode 0x00 — forces SSD to commit buffered data to NAND

      nvmeTrim(startLBA, numLBA)
        Opcode 0x09 — TRIM/deallocate command

    QUEUE MANAGEMENT (pipelining):
      nvmeSubmitIOCommand(sqe)      Write SQE to I/O SQ, ring doorbell
      nvmeServiceIOCompletions(max) Poll I/O CQ, process completed entries,
                                    update completion head doorbell
      nvmeGetIOSlip()               Returns (submitted - completed) count
                                    Used for pipelining: keep submitting
                                    while slip < NVME_SLIP_ALLOWED (16)

      PIPELINING PATTERN (from Shane's blog):
        "The nvmeWrite() function is non-blocking, so you can set up
        multiple transfers before the first is completed. But it becomes
        the application's responsibility to make sure the data for
        outstanding writes remains valid until they are completed."

        while (data_available) {
            while (nvmeGetIOSlip() > NVME_SLIP_ALLOWED)
                nvmeServiceIOCompletions(16);
            nvmeWrite(data, lba, blockSize);
        }

    ADMIN COMMANDS:
      nvmeAdminCommand(sqe, cqe, timeout)   Submit + wait for completion
      nvmeSubmitAdminCommand(sqe)            Write to Admin SQ, ring doorbell
      nvmeCompleteAdminCommand(cqe, timeout) Poll Admin CQ for response

  nvme.h                    Public API declarations
    nvmeInit, nvmeWrite, nvmeRead, nvmeFlush
    nvmeGetIOSlip, nvmeServiceIOCompletions
    Status: NVME_OK, NVME_NOINIT, NVME_ERROR_PHY

  nvme_priv.h               Private definitions
    SQE/CQE struct typedefs (sqe_prp_type, cqe_type)
    NVMe opcodes (WRITE=0x01, READ=0x02, FLUSH=0x00, TRIM=0x09)
    Memory addresses:
      Admin SQ: 0x10000000    (PS DDR4)
      Admin CQ: 0x10001000    (PS DDR4)
      I/O SQ:   0x10002000    (PS DDR4)
      I/O CQ:   0x10003000    (PS DDR4)
    NVMe register offsets: CAP, VS, CC, CSTS, AQA, ASQ, ACQ
    Doorbell register offsets


PCIe BRIDGE INTERFACE
─────────────────────
  pcie.c
    pcieInit()              Assert/deassert PCIe reset via GPIO, wait for link
    pcieDeinit()            Clean shutdown

  pcie.h                    Register definitions and addresses
    PHY status register:    0x500000144
    Root port status:       0x500000148
    Bridge enable:          0x500000208

    KEY MEMORY MAP (from Shane's Vivado block design):
      0x500000000 region:   XDMA bridge control/config (S_AXI(L)_CTL)
      0xA0000000-AF...:     Mapped to PCIe address space 0x00-0F...
      0xB0000000:           NVMe controller registers (SSD BAR0, via S_AXI(B))
      0x10000000:           NVMe queues in PS DDR4
      0x20000000:           Data buffer in PS DDR4

    FOR R5F PORT (from Shane's blog comments, August 2024):
      S_AXI_LITE  → 0xA0000000 (64M)   via M_AXI_HPM0_FPD
      S_AXI_B     → 0xA8000000 (1M)    via M_AXI_HPM0_FPD
      (Moved down to fit R5F's 32-bit address space)
```

---

## 3. Shane Colton's System-Level Data Flow Diagram

From Shane Colton's blog — system-level NVMe data flow:

```
                              XDMA Root Port (PL)
  Processor                 ┌─────────────────────┐          NVMe SSD
  (A53 or R5F)              │                     │
       │                    │  S_AXI(L)_CTL       │
       │  Config/status ───►│  Register Block     │
       │  (init only)       │                     │
       │                    │                     │
       │  SQE doorbell  ───►│  S_AXI(B)           │          ┌──────────┐
       │  NVMe registers    │  Slave Bridge       │──────────│  NVMe    │
       │  (tiny, per-cmd)   │  (PS→PCIe)          │  PCIe    │  Ctrl    │
       │                    │                     │  Gen3    │          │
       │                    │  M_AXI(B)           │  x4      │ ┌──────┐ │
       │                    │  Master Bridge      │◄─────────│ │ DMA  │ │
       │                    │  (PCIe→AXI)         │  TLPs    │ │Engine│ │
       │                    │                     │──────────│ └──────┘ │
       │                    └─────────┬───────────┘          │          │
       │                              │ M_AXI                │ ┌──────┐ │
       │                              │                      │ │ NAND │ │
       │                              ▼                      │ │ Flash│ │
       │                       ┌──────────┐                  │ └──────┘ │
       │  CQE poll ◄────────── │  DDR4    │ ◄── SSD DMA      └──────────┘
       │  (tiny, per-cmd)      │  (PS or  │     reads/writes
       │                       │   PL)    │     bulk data here
       │                       └──────────┘
       │
       │  The processor NEVER touches the bulk data.
       │  It only writes 64-byte SQEs and reads 16-byte CQEs.
       │  All GB/s data transfer is between DDR4 and SSD,
       │  through the Master Bridge, driven by the SSD's DMA engine.
```

Shane Colton's blog describes this as: "It's worth looking at a high-level diagram of what should be happening before diving in to the details of how to do it."

### Bare Metal NVMe Queue Structure

From Shane Colton's blog:

> "Fortunately, NVMe is an open standard. The specification is about 400 pages, but it's fairly easy to follow, especially with help from this tutorial. The NVMe Controller, which is implemented on the drive itself, does most of the heavy lifting. The host only has to do some initialization and then maintain the queues and lists that control data transfers."

> "One of the first things the host has to do is allocate some memory for the Admin Submission Queue and Admin Completion Queue. A Submission Queue (SQ) is a circular buffer of commands submitted to the drive by the host. It's written by the host and read by the drive (via the bridge AXI Master Interface). A Completion Queue (CQ) is a circular buffer of notifications of completed commands from the drive. It's written by the drive (via the bridge AXI Master Interface) and read by the host."

### FatFs Integration

From Shane Colton's blog:

> "I am more than content with just dumping data to the SSD directly as described above and leaving the task of organizing it to some later, non-time-critical process. But, if I can have it arranged neatly into files on the way in, all the better. I don't have much overhead to spare for the file system operations, though. Luckily, ChaN gifted the world FatFs, an ultralight FAT file system module written in C. It's both tiny and fast, since it's designed to run on small microcontrollers."

> "Linking FatFs to NVMe couldn't really get much simpler: FatFs's diskio.c device interface functions already request reads and writes in units of LBs, a.k.a. sectors. There's also a sync function that matches up nicely to the NVMe flush command."

### SLC Cache Behavior — Shane's Observations

From Shane Colton's blog:

> "Sustained write speeds begin to drop off after about 32 GiB. The Samsung Evo SSDs have a feature called TurboWrite that uses some fraction of the non-volatile memory array as fast Single-Level Cell (SLC) memory to buffer writes. Unlike the VWC, this is still non-volatile memory, but it gets transferred to more compact Multi-Level Cell (MLC) memory later since it's slower to write multi-level cells. The 1TB drive that I'm using has around 42GB of TurboWrite capacity according to this review, so a drop off in sustained write speeds after 32 GiB makes sense. Even the sustained write speed is 1.7 GB/s, though, which is more than fast enough for my application."
