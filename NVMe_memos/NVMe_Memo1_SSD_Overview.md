# NVMe SSD Interface — Overview & Recommendation


---

## 1. Three Paths to NVMe SSD Read/Write

### SW Path 1: Custom/Yocto Linux NVMe Driver with `O_DIRECT`

Work already done, proven by at least 3 projects outside Pixxel (see Section 4 below).

Work to be done at Pixxel: SW side testing on SSDs, multiple SSDs, simultaneous read/write.

SSD's internal DMA engine takes care of generating Read/Write transactions. Zynq PS 1 core will be utilized at near 5% to manage IO queues.

```
PS (Yocto Linux + NVMe driver) → AXI → XDMA IP (PL, Root Port + AXI Bridge mode)
  → GTH/GTY → Gen3 PCIe → SSD
```

The XDMA IP here is not an NVMe controller — it's just a PCIe-to-AXI bridge. The NVMe protocol intelligence lives in two places: (1) the Linux NVMe block device driver running on PS, and (2) the SSD's own internal DMA engine/firmware.

The XDMA bridge simply translates PCIe TLPs to AXI transactions. The SSD's DMA engine issues memory reads/writes back through the bridge into DDR. The PS software just manages NVMe command/completion queues — it doesn't do the bulk data transfer itself.


### SW Path 2: Baremetal NVMe Driver on R5F (Cortex-R5F)

To remove Zynq PS A53/Linux from SSD I/O entirely: offload the NVMe queue management to a baremetal driver running on the R5F (Cortex-R5F) real-time processor inside the same Zynq UltraScale+ chip.

Does not require `O_DIRECT` since there is no Linux on the R5F — no page cache exists to bypass.

Code available from Shane Colton running a baremetal NVMe driver on Zynq PS A53 Cortex processor:
- Blog: https://scolton.blogspot.com/2019/11/zynq-ultrascale-fatfs-with-bare-metal.html
- Code: https://github.com/coltonshane/SSD_Test

Shane confirmed in blog comments he ran the same code with minor changes on the R5F processor. Multiple other people confirmed tests with actual SSDs.

Linux communicates with R5F via standard Xilinx OpenAMP/RPMsg inter-processor communication for high-level commands (start/stop recording, session management). R5F handles all SSD operations autonomously.


### HW Path: FPGA PL NVMe (NVMeCHA-style)

Needs a lot of work by both RTL and SW teams. Needs multiple people for both RTL and SW.

NVMeCHA project only achieved high speeds because it was not connected to any SSDs. The project tested only GTH transceiver max speed. The PCIe link (the physical connection to the SSD) has GTH inside it — NVMeCHA measured the GTH link speed, not SSD write speed.


### Recommendation

**SW Path 2 (R5F baremetal)** gives us zero A53/Linux involvement in SSD I/O, achieves the same SSD-limited speeds as any approach, and is based on proven open-source code. HW Path (PL NVMe) would require months of RTL+SW effort for zero speed improvement.

---

## 2. All Three Paths Hit the Same SSD Speed Ceiling

All three paths — Linux `O_DIRECT`, R5F baremetal, and FPGA PL NVMe — are eventually limited by SSD type and cache technology.

The moment the NVMeCHA PL NVMe circuit is connected to a real SSD, speed drops to the SSD's native TLC/QLC NAND write speed, which is significantly lower than the SLC cache burst speed advertised on datasheets.

The resulting speed will be similar to what is achieved by the software methods (Custom/Yocto Linux NVMe driver with `O_DIRECT` flag set, or baremetal driver on R5F).

---

## 3. SSD Cache Technology: Consumer vs Datacenter

Sustained high write speeds to SSDs depend entirely on the type of SSD. Only enterprise datacenter server-grade SSDs can guarantee sustained similar write speed from first to last byte.

### The Cache-to-Speed Situation

**Consumer SSDs:** Use a pseudo-SLC write cache over TLC NAND — fast burst, then sharp speed drop when cache fills. The SSD temporarily operates a portion of its TLC NAND in SLC mode (1 bit per cell instead of 3 — faster because fewer voltage levels to distinguish). After the pseudo-SLC cache fills, the controller must write directly to TLC at native speed (much slower), AND later "fold" the SLC-cached data into TLC in the background, which steals even more bandwidth.

**Enterprise/Datacenter SSDs:** Same 3D TLC NAND technology, but skip the SLC cache entirely and write directly to TLC at a consistent rate using higher NAND parallelism and extra reserved NAND capacity — sustained speed from first byte to last. No SLC cache trick, no folding.

### Consumer SSD Example: Samsung 970 Evo Plus

Shane Colton used the Samsung 970 Evo Plus 1TB:
- 42 GB of "TurboWrite" SLC cache capacity
- Initial write speed inside cache: ~2.5 GB/s
- Once cache was exhausted: sustained write speed dropped to **1.7 GB/s**
- This 1.7 GB/s held steady from cache exhaustion all the way to drive full

**Always use the largest capacity version of any SSD — 1TB or 2TB minimum.** Larger drives have more NAND dies, giving more internal write parallelism and significantly higher sustained post-cache TLC write speeds (e.g., 970 Evo Plus 1TB sustains 1.7 GB/s post-cache vs 250GB version dropping to ~400 MB/s).

Samsung 970 EVO Plus:


https://www.storagereview.com/review/samsung-970-evo-plus-2tb-review
<img width="850" height="401" alt="image" src="https://github.com/user-attachments/assets/625ced6d-c159-4f1a-8f36-e00163496eb1" />



Samsung 990 Pro 2TB:



https://www.storagereview.com/review/samsung-990-pro-ssd-review-2tb
<img width="850" height="401" alt="image" src="https://github.com/user-attachments/assets/36d1732c-51a4-40d1-9c20-836a56c14de7" />

* Rs. 66K in India now https://mdcomputers.in/product/samsung-990-pro-2tb-nvme-ssd-mz-v9p2t0bw
* Samsung 990 Pro SSD seems to be good Consumer Drive with post cache sequential write speed at avg 2.25GB/s
* is Gen4, while FPGA PCIe Ip is Gen 3(PL side) and Gen2(PS side)
* Still possible we get this speed if we do a IO_Direct or Baremetal driver test
* Speed bottleneck is SSD, not Gen 2/3/4 which all have speed ceiling greater than SSD

| PCIe Generation | Speed per ×1 Lane | Total Speed for a ×4 Link |
|---|---|---|
| PCIe Gen 2 | ≈ 500 MB/s | ≈ 2 GB/s |
| PCIe Gen 3 | ≈ 1 GB/s | ≈ 4 GB/s |
| PCIe Gen 4 | ≈ 2 GB/s | ≈ 8 GB/s |

* Gen4 SSDs are fully backward compatible with Gen2/Gen3.** PCIe link training auto-negotiates to the highest mutually supported generation. A 990 Pro (Gen4) on our PL Gen3 x4 trains at Gen3 (~4 GB/s ceiling — no penalty, SSD sustained write is only ~2.25 GB/s). On PS Gen2 x4, it trains at Gen2 (~2 GB/s ceiling — slight link bottleneck, ~1.5–1.8 GB/s practical). NVMe registers, BARs, command set, and M.2 connector are identical across generations.
---

The StorageReview sequential write benchmarks above illustrate the consumer SSD cache wall. In the 970 EVO Plus graph, write latency stays low during the initial SLC cache burst phase while throughput climbs quickly. Once the 42 GB TurboWrite cache fills, throughput stabilizes at the native TLC write speed. Both the 1TB and 2TB variants of the 970 EVO Plus converge to approximately the same post-cache sustained throughput of **~1.45 GB/s** with latency settling around **~750 μs**. The 990 Pro 2TB shows the same cache-to-sustained transition pattern but at significantly higher post-cache performance, sustaining **~2.25 GB/s**.

For our continuous satellite sensor recording workload, the SLC cache exhausts within seconds. The vast majority of capture time is spent at post-cache sustained TLC write speed: **~1.45 GB/s** for the 970 EVO Plus, and **~2.25 GB/s** for the 990 Pro 2TB.

Enterprise/datacenter SSDs do not exhibit this behavior. Their sequential write speeds in the datasheets below are the actual sustained speeds from first byte to last — no cache burst, no latency spike, no speed drop. What you see in the spec sheet is what you get continuously.

---
### Enterprise/Datacenter SSD Examples

Form factors: U.2, U.3, some M.2 (but maybe longer)

Models: 

Micron 7450 MAX https://www.storagereview.com/news/micron-7450-ssd-announced


<img width="393" height="265" alt="image" src="https://github.com/user-attachments/assets/fc32a313-31ff-44d9-ba39-393103b50e65" />

<img width="393" height="306" alt="image" src="https://github.com/user-attachments/assets/0145b072-3c28-47c1-8f76-c7b229b62791" />

Micron 9400 MAX  https://www.storagereview.com/review/micron-9400-pro-ssd-review
<img width="419" height="283" alt="image" src="https://github.com/user-attachments/assets/b2ffd7eb-2004-4e5c-92d6-dc49ed22b0c1" />


Samsung PM9A3 https://www.storagereview.com/review/samsung-pm9a3-ssd-review

Samsung PM983 https://www.galaxus.nl/en/s1/product/samsung-pm893-1920-gb-25-ssd-16483427

Kioxia CD8  https://www.storagereview.com/news/kioxia-cd8-series-pcie-5-0-ssd-announced

Kioxia CM7 https://www.galaxus.nl/en/s1/product/kioxia-ssd-384tb-cm7-r-series-25-pcie-50-3840-gb-25-ssd-40729074

---

## 4. External Project References: Proven NVMe Speeds on Zynq

These links prove that Zynq PS with Yocto-Linux NVMe driver and SSD's own DMA engine, with PL-side XDMA IP configured as an AXI bridge, can indeed reach up to minimum 460–550 MB/s (near 4 Gbps) with page-caching on and up to 1.5 GB/s (12 Gbps) with page caching off.

Architecture common to all three:
```
PS (Yocto Linux + NVMe driver) → AXI → XDMA IP (PL, Root Port + AXI Bridge mode)
  → GTH/GTY → Gen3 PCIe → SSD
```


### Case 1: Semiengineering — Custom Linux + DIRECT_IO on Gen3 x4

Custom Linux (built from Xilinx kernel tree) + DIRECT_IO on Gen3 x4:
- Read: 3.3 GB/s | Write: 1.5 GB/s (single SSD)
- Read: 4.0 GB/s | Write: 3.4 GB/s (4x SSDs)
- Ref: https://semiengineering.com/evaluating-nvme-ssd-multi-gigabit-performance/

Note: They built a custom embedded Linux directly from the Xilinx kernel tree v2018.3 and used DIRECT_IO to bypass Linux VFS/Page Cache overhead. Without `O_DIRECT`, speeds were ~3x slower (~850 MB/s read, ~550 MB/s write).


### Case 2: User "kuku" — Debian + O_DIRECT on Gen3 x4

1.2 GB/s write confirmed using the same XDMA PL bridge driver (`pcie-xdma-pl.c`) + Debian distro (ZynqMP-FPGA-Linux) with `O_DIRECT` on Gen3 x4.

Command used:
```
dd if=/dev/zero of=/mnt/ssd/test_file bs=8M count=128 conv=fdatasync oflag=direct
```

Ref: Comments on https://scolton.blogspot.com/2019/07/benchmarking-nvme-through-zynq.html

For context — without `O_DIRECT` (page caching ON), the same XDMA bridge topology only achieved ~460 MB/s write and ~630 MB/s read due to Linux VFS/Page Cache overhead.


### Case 3: ADI Kuiper Linux + O_DIRECT on PS PCIe Gen2 x4

1.5 GB/s write on PS-side hardened PCIe (Gen2 x4) using ADI Kuiper Linux + `O_DIRECT` on ZCU102. No PL-side XDMA — uses the hardened PS GTR PCIe controller directly. All 4 PSGTR lanes routed to PCIe via device tree (USB3, DP, SATA disabled).

Commands used:
```
# dd with oflag=direct:
time sudo dd if=/dev/zero of=/mnt/myNVMe/dd.log bs=64M count=16 oflag=direct

# fio with direct=1:
sudo fio --name=test --filename=/mnt/myNVMe/dd.log --size=8G --bs=16M \
  --rw=write --ioengine=libaio --direct=1 --iodepth=16 --numjobs=1
```

fio result: near theoretical PCIe Gen2 x4 upper bound (~1.4–1.8 GB/s write).

Note: fio with libaio + direct=1 significantly outperformed dd + oflag=direct, since dd uses standard write() syscalls which are simpler but less optimized. SSD used: FANXIANG S790 2TB. PS clock set to 1333 MHz (max for ZCU102).

Ref: https://github.com/wonderfulnx/Xilinx-MPSoC-NVMeSSD-Config


### Key Insight: O_DIRECT

The main bottleneck is Linux page caching. With `O_DIRECT` enabled (either via `oflag=direct` in dd, or `O_DIRECT` flag in application `open()` calls), page cache translation overhead is bypassed and I/O requests go directly to the Block I/O layer and then to the NVMe hardware driver.

Using Yocto with full GNU coreutils ensures dd supports `oflag=direct`. Alternatively, `O_DIRECT` can be set programmatically in application code:
```c
int fd = open("/dev/nvme0n1", O_WRONLY | O_DIRECT | O_CREAT, 0777);
```


### Write Speed Bottleneck Analysis Across 3 Cases

**Case 1's** read speed of 3.3 GB/s (84% of Gen3 x4 theoretical max) proves the XDMA bridge and PCIe link can sustain high bandwidth. The 4x SSD result (3.4 GB/s write) further confirms the link has headroom. Yet single-SSD write tops out at 1.5 GB/s — the semiengineering article attributes this directly to SSD TLC NAND and SLC cache limits, not the I/O path. So for Case 1, switching to fio would likely yield only marginal improvement since the bottleneck is already inside the SSD.

**Case 2** (kuku's 1.2 GB/s) is more likely to benefit from fio. kuku used dd with `bs=8M` which runs synchronous single-threaded writes at effectively `iodepth=1`. fio with `libaio + direct=1 + iodepth=16` enables async I/O and deep NVMe command queues, which NVMe drives are designed to exploit. This could push Case 2's 1.2 GB/s closer to the ~1.5 GB/s ceiling seen in Case 1 — after which the SSD itself becomes the limit again.

**Case 3** already demonstrates this: on Gen2 x4, fio with `libaio + direct=1` reached near-theoretical PCIe bandwidth, significantly outperforming `dd + oflag=direct` on the same hardware.

**Bottom line:** To push single-SSD write beyond ~1.5 GB/s on Gen3 x4, the path is not software optimisation — it's choosing SSDs with higher sustained TLC write performance or larger SLC caches. Enterprise/datacenter NVMe SSDs are designed for steady-state write workloads without SLC cache tricks and can sustain higher write rates. Alternatively, striping across multiple SSDs in parallel (as Case 1's 4x SSD result shows) scales write bandwidth linearly with the number of drives.

---

## 5. About the DMA Engine Inside SSDs

From Shane Colton's blog:

> The AXI Master Interface is where all NVMe data transfer occurs, for both reads and writes. One way to look at it is that the drive itself contains the DMA engine, which issues memory reads and writes to the system (AXI) memory space through the bridge. The host requests that the drive perform these data transfers by submitting them to a queue, which is also contained in system memory and accessed through this interface.

> The NVMe Controller, which is implemented on the drive itself, does most of the heavy lifting. The host (Zynq PS A53 or R5) only has to do some initialization and then maintain the queues and lists that control data transfers.

---

## 6. Shane Colton Blog — Key Discussion Points

### R5F Processor Baremetal Driver

Shane Colton (August 2, 2024): "I've run the NVMe driver on the R5 with only some minor modifications, I think mostly to clear compiler warnings about pointer casts. I think somebody else in the comments was running it on a Microblaze."

Shane Colton (August 30, 2024): "In my case, I saw a slight decrease from about 3.05 GB/s to 2.90 GB/s for sequential 128K raw disk writes. I'm not sure where the actual bottleneck is, though. It's possible that with some optimization, there would be no difference, especially for a smaller link. The DMA transfers to/from the drive to DDR4 should be unaffected. It's just the overhead of reading and writing the queues, plus any file system or data management overhead, that might take a small hit from the R5's lower speed."

Note: The speeds mentioned (3.05 GB/s to 2.90 GB/s) are within the SLC cache burst. They will drop to ~1.7 GB/s after cache fills on his 970 Evo Plus 1TB.

### Using PL BRAM/URAM as Data Source Instead of DDR4

Anonymous commenter (November 4, 2024): "I opted for a BRAM/URAM solution which has worked out nicely and allows easy access from the PL side. Right now I am sustaining write transactions at a rate of 737 MB/s. [...] how many pcie lanes do you have? I only have one pcie lane due to the constraints of my board."

Shane Colton (November 5, 2024): "For PCIe 3.0 x1, the raw data rate is only 1 GB/s, so practical transfer of 737 MB/s is pretty good. You would need four lanes to get to 3 GB/s."

### Pipelining Writes for Maximum Speed

Shane Colton (November 5, 2024): "The second half of this post describes how you can pipeline writes to maximize bus speed. The `nvmeWrite()` function in this driver is non-blocking, so you can set up multiple transfers before the first is completed. But, it becomes the application's responsibility to make sure the data for outstanding writes remains valid until they are completed. In your case, this would probably mean operating the BRAM/URAM as a circular buffer and using `nvmeGetSlip()` to make sure the number of writes in-progress never exceeds the capacity of the buffer."

### Multi-SSD Pipelining

Each SSD gets its own independent NVMe I/O queue pair. The `nvmeWrite()` function is non-blocking — it submits the command and returns immediately. The `nvmeGetIOSlip()` function tracks how many commands are in flight. Keep submitting while `slip < 16`. For multiple SSDs, each SSD's pipeline runs independently — one SSD stalling does not block writes to others.

---

## 7. Filesystem Support

### FatFs (Baremetal, R5F Path)

Shane Colton's repo includes ChaN's FatFs — an ultra-light FAT filesystem written in C (~27 KB). Already integrated with his NVMe driver via `diskio.c`. Supports FAT32 and exFAT.

- FAT32 limits: 2 TiB max drive (512B sectors), 4 GiB max file size
- exFAT: no practical limits, but requires Microsoft license for baremetal use
- Multi-SSD: FatFs supports up to 10 logical volumes (`FF_VOLUMES` in `ffconf.h`)
- For 3 SSDs: expand `diskio.c` with `switch(pdrv)` to route to XDMA1/XDMA2/XDMA3

### Raw Block Writes (Alternative)

Write directly to SSD LBA offsets. No filesystem overhead. PL-embedded metadata headers (sync word, band ID, timestamp, CRC) make data self-describing. Maximum speed, but SSD is not readable by a standard PC without custom software.

### Linux Filesystem (A53 Path)

Standard Linux ext4, xfs, exfat. `O_DIRECT` bypasses page cache. exFAT is royalty-free on Linux (since Microsoft joined OIN in 2019).

---

