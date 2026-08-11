# CPUMAP: increase CPU_MAP_BULK_SIZE from 8 to 16

Investigation into the cross-NUMA `ptr_ring` producer bottleneck in
`kernel/bpf/cpumap.c`, with a proposal to increase `CPU_MAP_BULK_SIZE`
from 8 to 16 to align the producer's write stride with the consumer's
batch-NULLing stride and the AMD Infinity Fabric's coherency granularity.

Builds on the cross-NUMA evaluation in
[cpumap05-AMD-cross-NUMA-eval.md](cpumap05-AMD-cross-NUMA-eval.md).

## Background: where the cross-NUMA cost lives

Cross-NUMA redirect from cpu 6 (NUMA node 0) to cpu 104 (NUMA node 1),
remote-action drop (consumer drops immediately in the remote XDP program, so
it runs as fast as possible and the ring stays near-empty):

```
sudo ./xdp-bench redirect-cpu -e -p round-robin --cpu 104 --remote-action drop ice4
```

```
ice4->?                 4,867,852 rx/s                  0 err,drop/s
  receive total         4,867,852 pkt/s                 0 drop/s                0 error/s
    cpu:6               4,867,852 pkt/s                 0 drop/s                0 error/s
  enqueue to cpu 104    4,867,859 pkt/s                 0 drop/s             8.00 bulk-avg
    cpu:6->104          4,867,859 pkt/s                 0 drop/s             8.00 bulk-avg
  kthread total         4,867,848 pkt/s                 0 drop/s           21,652 sched
    cpu:104             4,867,848 pkt/s                 0 drop/s           21,652 sched
    xdp_stats                   0 pass/s        4,867,848 drop/s                0 redir/s
      cpu:104                   0 pass/s        4,867,848 drop/s                0 redir/s
  xdp_exception                 0 hit/s
```

Result: **~4.9 Mpps** with 0 drops. The consumer keeps up easily (21,652
sched/s -- the kthread frequently finds the ring empty and goes to sleep).
The bottleneck is the producer (cpu 6) paying cross-NUMA costs on every
`ptr_ring` produce.

`perf record` on cpu 6 shows ~94% of `xdp_do_redirect` samples landing on
the instructions immediately after `call cpu_map_enqueue`:

```
     -> call    cpu_map_enqueue
31.15|  mov     (%rsp),%r10
15.43|  mov     0xc(%rsp),%r8d
31.69|  mov     %eax,%r15d
15.57|  jmp     194
```

These `mov` instructions are trivial -- the cost is **return-site skid**: the
out-of-order core cannot retire them until `cpu_map_enqueue`'s in-flight
memory operations drain. The stall is inside the inlined
`bq_enqueue -> bq_flush_to_queue -> __ptr_ring_produce` path, where the
producer writes into `ptr_ring` slot memory that lives on the remote NUMA
node.

### Why the producer stalls

The cost is not about which NUMA node allocated the `ptr_ring` memory -- any
CPU can cache remote memory locally. What matters is the **cache coherence
state** of the cachelines being accessed.

The AMD EPYC 9684X (Genoa-X) uses the MOESI (Modified, Owned, Exclusive,
Shared, Invalid) cache coherence protocol with a hierarchical two-level
structure. Each NUMA node on this processor corresponds to one CCD (Core
Complex Die) -- a physical chiplet containing 8 Zen 4 cores sharing 96 MB of
3D V-Cache L3. The 12 CCDs communicate through a central I/O Die (IOD) over
GMI3 (Global Memory Interconnect 3) links -- the on-package fabric that
connects the core chiplets to the IOD.

- **Intra-CCD** (within a NUMA node): coherency between the 8 cores is
  handled rapidly at the local L3 cache level.
- **Inter-CCD** (across NUMA nodes): requests travel over the GMI3 link to a
  centralized Probe Filter (directory) in the IOD. The Probe Filter tracks
  which CCDs hold copies of which cachelines, sending targeted probes only to
  the specific CCD holding the data -- not broadcasting to all 12 nodes.

A key property of MOESI's **Owned state** is that dirty data can be
transferred directly between CCDs through the IOD without writing back to
DRAM first. So the cross-NUMA cost is **pure fabric latency** (GMI3 hop to
the IOD + targeted probe + data transfer), not a memory-bandwidth penalty.
But this fabric round-trip still costs hundreds of cycles per cacheline
ownership transfer.

The `ptr_ring` produce path is a textbook cross-CCD coherence bottleneck:

```c
/* ptr_ring.h: __ptr_ring_produce */
if (data_race(r->queue[r->producer]))   /* read: is slot free? */
    return -ENOSPC;
WRITE_ONCE(r->queue[r->producer++], ptr); /* write: store frame pointer */
```

Each iteration does:
1. A **read** of `r->queue[r->producer]` -- the slot the consumer recently
   NULLed, so the cacheline is in Modified state on the consumer's CCD
2. A **write** of the frame pointer into the same cacheline -- requires
   obtaining exclusive ownership from the consumer's CCD

The consumer (CPUMAP kthread on the remote CPU) NULLs slots as it drains
them (`__ptr_ring_zero_tail`), putting those cachelines in Modified state on
its CCD. When the producer next writes to those same slots, the MOESI
protocol must transfer ownership back: the IOD's Probe Filter identifies the
consumer's CCD as the holder, sends a targeted invalidation probe, the
consumer's CCD responds with the dirty data (via the Owned state, bypassing
DRAM), and the producer obtains the line Exclusive for its write. This is a
full fabric round-trip per cacheline ownership change. The
`spin_lock(&q->producer_lock)` is also a cross-CCD atomic RMW through the
same path.

### Near-full vs near-empty ring confirms the theory

A test that slows down the consumer (XDP_PASS + iptables raw drop instead of
dropping in the remote XDP program) forces the `ptr_ring` near-full,
producing drops:

```
sudo ./xdp-bench redirect-cpu -e -p round-robin --cpu 104 --cpu 105 --cpu 106 --cpu 107 --remote-action pass ice4
```

```
ice4->?                 7,702,120 rx/s                  0 err,drop/s
  receive total         7,702,120 pkt/s                 0 drop/s                0 error/s
    cpu:6               7,702,120 pkt/s                 0 drop/s                0 error/s
  enqueue to cpu 104    1,925,507 pkt/s           259,505 drop/s             8.00 bulk-avg
    cpu:6->104          1,925,507 pkt/s           259,505 drop/s             8.00 bulk-avg
  enqueue to cpu 105    1,925,508 pkt/s           269,977 drop/s             8.00 bulk-avg
    cpu:6->105          1,925,508 pkt/s           269,977 drop/s             8.00 bulk-avg
  enqueue to cpu 106    1,925,508 pkt/s           276,816 drop/s             8.00 bulk-avg
    cpu:6->106          1,925,508 pkt/s           276,816 drop/s             8.00 bulk-avg
  enqueue to cpu 107    1,925,509 pkt/s           274,590 drop/s             8.00 bulk-avg
    cpu:6->107          1,925,509 pkt/s           274,590 drop/s             8.00 bulk-avg
  kthread total         6,621,145 pkt/s                 0 drop/s               13 sched
    cpu:104             1,665,990 pkt/s                 0 drop/s                3 sched
    cpu:105             1,655,544 pkt/s                 0 drop/s                3 sched
    cpu:106             1,648,693 pkt/s                 0 drop/s                3 sched
    cpu:107             1,650,918 pkt/s                 0 drop/s                3 sched
```

Key observations:
- **sched = 3** per kthread: consumers almost never sleep -- the ring is full
  enough that they spin continuously (the ~260k drop/s per queue confirms
  ring overflow)
- **7.7M pps producer throughput** -- significantly higher than the ~4.9M pps
  in the near-empty single-remote-CPU case above

Comparison (both cross-NUMA, cpu 6 -> NUMA node 1):

| Metric               | Near-empty (drop) | Near-full (pass+iptables) |
| -------------------- | ----------------: | ------------------------: |
| Producer pps (cpu 6) |        4,867,852  |               7,702,120   |
| Consumer sched/s     |          21,652   |                    3      |
| Drops/s (per queue)  |               0   |             ~270,000      |
| Ring state           |     mostly empty  |           mostly full     |

The producer is **58% faster** when the ring is full, despite doing the same
work per packet. The difference is entirely in cacheline ownership costs.

The `perf` profile on the near-full case shows the same pattern but with
reduced stall:

```
     -> call    cpu_map_enqueue
27.55|  mov     (%rsp),%r10
13.67|  mov     0xc(%rsp),%r8d
27.51|  mov     %eax,%r15d
13.99|  jmp     194
```

Total stall drops from **93.84% to 82.72%**, an 11 percentage-point
reduction in the execution time of `xdp_do_redirect`.

`perf diff` (10 sec measurement on cpu 6) between near-empty and near-full:

```
# Event 'cycles:P'
#
# Baseline  Delta Abs  Shared Object          Symbol
# ........  .........  .....................  .......................................
#
    43.61%    -13.88%  [kernel.kallsyms]      [k] xdp_do_redirect
     4.83%     +6.24%  [kernel.kallsyms]      [k] page_pool_refill_alloc_cache
    11.73%     +3.54%  [ice]                  [k] ice_napi_poll
     7.55%     -2.58%  [kernel.kallsyms]      [k] bq_flush_to_queue
     4.23%     +2.43%  bpf_prog_...           [k] ...cpumap_round_robin
     6.95%     -2.33%  [kernel.kallsyms]      [k] bpf_trace_run4
     2.78%     +2.24%  bpf_prog_...           [k] ...tp_xdp_cpumap_enqueue
     4.10%     -1.35%  [kernel.kallsyms]      [k] cpu_map_enqueue
     5.13%     +1.35%  [ice]                  [k] ice_alloc_rx_bufs
```

The near-full ring shifts **-13.88%** away from `xdp_do_redirect` (which
contains the inlined `__ptr_ring_produce`) and **-2.58%** from
`bq_flush_to_queue` -- together almost 17 percentage points freed from the
cross-NUMA `ptr_ring` produce path. That time reappears in functions that
were previously starved for cycles: `page_pool_refill_alloc_cache` (+6.24%),
`ice_napi_poll` (+3.54%), `ice_alloc_rx_bufs` (+1.35%) -- the driver and
page allocator can now run faster because they are no longer stalled behind
the remote-memory produce.

The producer is faster because on a near-full ring, the slots it writes into
are ones **it wrote itself on the previous lap** -- those cachelines are
still in Modified state in the producer's local cache. On a near-empty ring,
the consumer has already NULLed those slots, so every produce is a
cross-NUMA cacheline ownership transfer (remote M->I + local I->M).

This confirms that **cacheline ownership bouncing between producer and
consumer is a significant component of the cross-NUMA cost**, on top of the
base cost of remote-memory stores.

## The ptr_ring consumer batch mechanism

`ptr_ring` already tries to mitigate this. The consumer defers NULLing
consumed slots, batching the NULLs to keep its writes away from the
producer's reads (`include/linux/ptr_ring.h`):

```c
static inline void __ptr_ring_set_size(struct ptr_ring *r, int size)
{
    r->size = size;
    r->batch = SMP_CACHE_BYTES * 2 / sizeof(*(r->queue));
    ...
}
```

On x86_64: `SMP_CACHE_BYTES = 64`, so `batch = 64 * 2 / 8 = 16` slots =
128 bytes = **two cachelines**. The consumer NULLs 16 slots at a time in
`__ptr_ring_discard_one` -> `__ptr_ring_zero_tail`, keeping its NULL-writes
two cachelines behind the consumer read head.

## The mismatch: producer writes 1 cacheline, consumer batches 2

`CPU_MAP_BULK_SIZE` is currently 8:

```c
#define CPU_MAP_BULK_SIZE 8  /* 8 == one cacheline on 64-bit archs */
```

So `bq_flush_to_queue` produces 8 entries per flush -- writing 8 pointers =
64 bytes = **one cacheline** of the `r->queue[]` array. But the consumer
batches its NULLing at 16 entries = **two cachelines**.

This means the producer's write stride and the consumer's NULL stride are
**mismatched**: the producer fills one cacheline per flush, the consumer
NULLs two cachelines per batch. On a near-empty ring where they chase each
other closely, the consumer's batch-NULL of 2 cachelines can overlap with the
cacheline the producer is about to write into on the *next* flush.

## Proposal: increase CPU_MAP_BULK_SIZE to 16

Changing `CPU_MAP_BULK_SIZE` from 8 to 16 aligns the producer's flush stride
with the consumer's batch stride:

- **Producer writes 16 slots = 128 bytes = 2 cachelines** per flush
- **Consumer NULLs 16 slots = 128 bytes = 2 cachelines** per batch
- Both step through the ring in the same 2-cacheline regions

### Why 16 specifically

1. **Matches ptr_ring's own batch design.** `ptr_ring` chose `batch =
   SMP_CACHE_BYTES * 2 / sizeof(void *)` = 16 to keep consumer NULLs 2
   cachelines away from producer reads. Making the producer also operate in
   16-entry strides means both sides step through the ring at the same
   granularity, so when one side finishes with a 2-cacheline region the other
   takes it over cleanly.

2. **Matches AMD Infinity Fabric coherency granularity.** On Zen 3/4 (EPYC
   Milan/Genoa), the L1D cache uses 64-byte lines, but the Infinity Fabric's
   prefetch and coherency protocol operates at **128-byte granularity** --
   pairs of cachelines move as a unit across the interconnect. At bulk=8 (64
   bytes), the producer's write burst fills half a fabric transfer unit; the
   other half is wasted bandwidth. At bulk=16 (128 bytes), the produce burst
   fills the **entire fabric transfer unit**.

3. **Halves lock acquisitions.** With NAPI budget = 64 and all frames going
   to the same remote CPU:
   - bulk=8: overflow at frames 8,16,...,56 -> 7 mid-poll + 1 end-of-poll = 8
     flushes, 8 `spin_lock` on remote-NUMA `producer_lock`
   - bulk=16: overflow at 16,32,48 -> 3 mid-poll + 1 end-of-poll = 4 flushes,
     4 `spin_lock` on remote-NUMA `producer_lock`

### Costs

- **percpu bulk queue grows from 64 to 128 bytes.** The `struct
  xdp_bulk_queue` `q[]` array doubles from 8 to 16 pointers. This is percpu
  per remote CPU: on a 192-CPU box with 96 CPUMAP entries, that is 192 * 96 *
  64 bytes = ~1.1 MiB more percpu memory. Modest.

- **`CPUMAP_BATCH` (consumer side) should match.** The consumer's
  `__ptr_ring_consume_batched` batch and the `frames[]`/`skbs[]` stack arrays
  in `cpu_map_kthread_run` use `CPUMAP_BATCH` (currently 8). Increasing to 16
  keeps producer and consumer batch sizes symmetric. Stack cost: 2 * 8 * 8 =
  128 bytes more stack in the kthread. The `napi_skb_cache_get_bulk` and
  `kmem_cache_alloc_bulk` calls scale to 16 naturally.

### Expected effect

The primary gain is not the halved lock count (the lock was already amortized
8x). The gain is **aligning the producer's memory-access pattern with the
consumer's batch-NULLing pattern and the hardware coherency granularity**, so
that on a near-empty ring the two sides exchange ownership of 2-cacheline
regions as complete units rather than interleaving at mismatched strides.
This should reduce the per-produce cacheline bounce cost that accounts for
the near-empty vs near-full performance gap demonstrated above.

### Kernel patch scope

The change is small:

```c
/* kernel/bpf/cpumap.c */
-#define CPU_MAP_BULK_SIZE 8  /* 8 == one cacheline on 64-bit archs */
+#define CPU_MAP_BULK_SIZE 16 /* 16 == two cachelines, matches ptr_ring batch */

-#define CPUMAP_BATCH 8
+#define CPUMAP_BATCH 16
```

No API or UAPI change. The `xdp_bulk_queue` struct grows but is internal.
The `CPUMAP_BATCH` stack arrays in `cpu_map_kthread_run` grow from 64 to 128
bytes each.

## TODO

- [ ] Benchmark with `CPU_MAP_BULK_SIZE=16` on the Gen12 cross-NUMA setup
      from cpumap05 (same-NUMA and cross-NUMA, state-mode none)
- [ ] Compare perf profiles: stall percentage on `cpu_map_enqueue` return
- [ ] Test with round-robin across 4 remote CPUs (the near-full test above)
      to measure the effect when multiple queues share the producer
- [ ] Verify no regression on same-NUMA (the bulk queue local writes go from
      1 to 2 cachelines, but both are L1 hits so should be negligible)
