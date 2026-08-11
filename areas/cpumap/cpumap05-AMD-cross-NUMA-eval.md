# CPUMAP cross-NUMA evaluation on AMD Gen12

Evaluating cross-NUMA overhead of cpumap shared state on a Cloudflare
Gen12 server (AMD EPYC 9684X Genoa-X, 12 NUMA nodes). The benchmark
tool is `xdp-bench redirect-cpu-state`, documented in
[xdp-bench-redirect-cpu-state.md](xdp-bench-redirect-cpu-state.md).

## Findings and results

(results go here)

---

## Test setup

### Device Under Test (DUT): Cloudflare Gen12 metal

Hostname: **12G**

Cloudflare Gen12 servers are described in these public blog posts:
- [Cloudflare's 12th Generation servers](https://blog.cloudflare.com/gen-12-servers/)
- [Analysis of the EPYC 145% performance gain](https://blog.cloudflare.com/analysis-of-the-epyc-145-performance-gain-in-cloudflare-gen-12-servers/)
- [Gen 12 Server: bigger, better, cooler in a 2U1N form factor](https://blog.cloudflare.com/cloudflare-gen-12-server-bigger-better-cooler-in-a-2u1n-form-factor/)

Key specs:
- **CPU**: AMD EPYC 9684X (Genoa-X), 96 cores / 192 threads, single socket
- **Microarchitecture**: Zen 4 with 3D V-Cache
- **L3 cache**: 1.1 GiB total (96 MB per CCD, 12 CCDs)
- **L2 cache**: 1 MB per core (96 MiB total)
- **Memory**: 384 GB DDR5-4800, 12 channels
- **NIC (stock)**: Intel E810-XXV for SFP, dual 25 GbE + NVIDIA Mellanox ConnectX-6 Lx
- **NIC (added for test)**: Intel E810-C for QSFP, 100 GbE (added in our test lab)

```
$ lspci | grep Ether
01:00.0 Ethernet controller: Intel Corporation Ethernet Controller E810-C for QSFP (rev 02)
01:00.1 Ethernet controller: Intel Corporation Ethernet Controller E810-C for QSFP (rev 02)
41:00.0 Ethernet controller: Intel Corporation Ethernet Controller E810-XXV for SFP (rev 02)
41:00.1 Ethernet controller: Intel Corporation Ethernet Controller E810-XXV for SFP (rev 02)
```

#### NUMA topology

12 NUMA nodes, each with 8 cores + 8 HT siblings (one CCD per node):

| NUMA node | CPUs          |
| --------- | ------------- |
| 0         | 0-7, 96-103   |
| 1         | 8-15, 104-111 |
| 2         | 16-23, 112-119|
| 3         | 24-31, 120-127|
| 4         | 32-39, 128-135|
| 5         | 40-47, 136-143|
| 6         | 48-55, 144-151|
| 7         | 56-63, 152-159|
| 8         | 64-71, 160-167|
| 9         | 72-79, 168-175|
| 10        | 80-87, 176-183|
| 11        | 88-95, 184-191|

Each NUMA node corresponds to one CCD (Core Complex Die) with one CCX
(8 Zen 4 cores sharing 96 MB L3). Cross-NUMA traffic must traverse the
Infinity Fabric through the I/O die.

#### lscpu

```
Architecture:                x86_64
CPU(s):                      192
Vendor ID:                   AuthenticAMD
Model name:                  AMD EPYC 9684X 96-Core Processor
  CPU family:                25
  Model:                     17
  Thread(s) per core:        2
  Core(s) per socket:        96
  Socket(s):                 1
  CPU max MHz:               3716.8860
  CPU min MHz:               1500.0000
  BogoMIPS:                  5091.99
Caches (sum of all):
  L1d:                       3 MiB (96 instances)
  L1i:                       3 MiB (96 instances)
  L2:                        96 MiB (96 instances)
  L3:                        1.1 GiB (12 instances)
NUMA node(s):                12
NUMA node0 CPU(s):           0-7,96-103
NUMA node1 CPU(s):           8-15,104-111
NUMA node2 CPU(s):           16-23,112-119
NUMA node3 CPU(s):           24-31,120-127
NUMA node4 CPU(s):           32-39,128-135
NUMA node5 CPU(s):           40-47,136-143
NUMA node6 CPU(s):           48-55,144-151
NUMA node7 CPU(s):           56-63,152-159
NUMA node8 CPU(s):           64-71,160-167
NUMA node9 CPU(s):           72-79,168-175
NUMA node10 CPU(s):          80-87,176-183
NUMA node11 CPU(s):          88-95,184-191
```

### Packet generator

Using the kernel's pktgen framework with 16 generator threads to ensure
the generator is never the bottleneck -- the DUT must be the limiting
factor for these benchmarks.

Command:

```sh
./samples/pktgen/pktgen_sample03_burst_single_flow.sh \
    -vi ice4 -d 198.18.100.1 -m b4:96:91:ad:0b:09 -t 16
```

The script comes from the Linux kernel git tree
(`samples/pktgen/pktgen_sample03_burst_single_flow.sh`). The `-t 16`
flag spawns 16 pktgen kernel threads, each bound to its own TX queue,
to saturate the link.

#### Verifying generator TX rate

Used [ethtool_stats.pl](https://github.com/netoptimizer/network-testing/blob/master/bin/ethtool_stats.pl)
on the generator to confirm aggregate TX rate:

```
~/git/network-testing/bin/ethtool_stats.pl --dev ice4
[...]
Ethtool(ice4    ) stat:     17,186,036 <= tx_size_64.nic /sec
Ethtool(ice4    ) stat:     17,185,670 <= tx_unicast /sec
```

Aggregate: **~17.2 Mpps** of 64-byte packets across 16 TX queues (~1.0-1.2 Mpps
per queue), sending ~1.03 GB/s on the wire (~1.10 GB/s at NIC level
including preamble/IFG).
