# xdp-bench: `redirect-cpu-state` benchmark — tool summary

Reference for our cross-NUMA / cpumap experiments.

Upstream (Jonas Köppeler's tree), branch `cpumap-shared-state-benchmark`:
<https://github.com/jkoeppeler/xdp-tools/commits/cpumap-shared-state-benchmark/>
- BPF programs (ingress + cpumap-remote):
  <https://github.com/jkoeppeler/xdp-tools/blob/cpumap-shared-state-benchmark/xdp-bench/xdp_redirect_cpumap_state.bpf.c>
- Userspace loader/config:
  <https://github.com/jkoeppeler/xdp-tools/blob/cpumap-shared-state-benchmark/xdp-bench/xdp_redirect_cpumap_state.c>

Local checkout lives in `~/git/xdp/xdp-tools.jonas/xdp-bench/`:
- `xdp_redirect_cpumap_state.c` — userspace loader/config
- `xdp_redirect_cpumap_state.bpf.c` — BPF programs (ingress + cpumap-remote)
- `xdp-bench.c` / `xdp-bench.h` — CLI option parsing and enums

## What it does

`redirect-cpu-state` is a micro-benchmark on top of the normal `redirect-cpu`
path. It uses `BPF_MAP_TYPE_CPUMAP` to redirect packets from the RX (ingress)
CPU to a set of remote CPUs, where the cpumap kthread runs a second XDP program
that drops the packets (`XDP_DROP`).

On top of the plain redirect, it optionally exercises a **shared state counter**
that is touched by *both* the ingress CPU and the remote CPU. This lets us
measure the cost of cache-line sharing / cross-core (and cross-NUMA) memory
traffic on the cpumap path, isolated from the rest of the stack.

### Packet flow

```
NIC RXq -> ingress XDP prog (RX CPU)         [selects dest CPU round-robin,
                                              optionally updates state]
        -> bpf_redirect_map(cpu_map, dest)
        -> cpumap kthread on remote CPU
        -> remote XDP prog                    [optionally updates state,
                                              returns XDP_DROP]
```

- `select_cpu()` picks the destination CPU round-robin over the `--cpu` list
  (via the `cpus_available` array + `cpus_iterator`).
- The ingress program is the "no_touch" variant (it does not read the packet),
  so we measure state/redirect overhead, not packet parsing.

## The shared state

The state lives in the `cpumap_state` BPF ARRAY map. Value is a
`struct cpumap_state` = 128 bytes holding **two counter slots**:

```c
struct cpumap_state {
    union {
        struct cpumap_state_slot padded[2];  // each slot 64B (own cacheline)
        __u64 packed[2];                      // two u64s adjacent (same line)
    };
};
```

Two slots let the benchmark model the producer/consumer counter pattern:
slot 0 is the "ingress-side" counter, slot 1 the "remote-side" counter. The
`transfer_state_*` helpers copy source_slot -> dest_slot (+delta).

## Options and how they map to code

Options are defined in `redirect_cpumap_state_options[]` (`xdp-bench.c`).

### `--cpu <cpu>` (required, repeatable, `-c`)
Destination CPUs inserted into the CPUMAP. Repeat to fan out to multiple remote
CPUs. Choosing RX CPU vs. remote CPU on same/other NUMA node is how we drive the
cross-NUMA evaluation.

### `<ifname>` (required, positional)
Interface to attach the ingress XDP program to.

### `--state-mode {none|separate|shared}` (default `none`)
Controls which counter slot the **remote** program uses, i.e. whether the two
CPUs hit the same or different counters. Maps to program-name selection in
`xdp_redirect_cpumap_state.c` (slot 0 vs slot 1).

- `none` — no state touched. Ingress = `cpumap_state_no_touch`, remote =
  `cpumap_state_drop`. Pure redirect+drop baseline.
- `shared` — **slot 0** on both sides. Ingress writes slot 0, remote writes
  slot 0. Both CPUs hammer the *same* cacheline → measures **true sharing /
  contention** (worst case; the cross-NUMA cost we care about).
- `separate` — remote uses **slot 1**. Ingress writes slot 0, remote writes
  slot 1. Different counters; with `padded` layout they are on different
  cachelines → avoids false sharing. Baseline for "what if we split the state".

(In the ingress program the `transfer_state_*(cpu_dest, remote_slot, 0, 1)`
call reads the remote slot and writes slot 0; in `shared` remote_slot=0, in
`separate` remote_slot=1.)

### `--state-update {atomic|non-atomic}` (default `atomic`)
- `atomic` — uses `__sync_fetch_and_add` / `__sync_lock_test_and_set`.
- `non-atomic` — plain read-modify-write.
Lets us isolate the cost of the atomic op vs. the cache-coherency traffic.

### `--state-layout {padded|packed}` (default `padded`)
- `padded` — each counter on its own 64B cacheline (slots 64B apart).
- `packed` — two `__u64` adjacent (8B apart, same cacheline).
With `separate` + `packed` you get **false sharing** (different counters, same
line); with `separate` + `padded` you don't. `shared` hits the same line
regardless.

### `--state-batch-size <packets>` (default 1)
Only touch/update the shared state once every N packets, instead of every
packet. Requires `--state-mode separate` or `shared` (batching with `none` is
rejected). Amortizes the shared-state cost; sweeping this shows how sensitive
throughput is to state-update frequency. Uses a per-CPU
`state_batch_counters` map to count down to the batch boundary.

### `--qsize <packets>` (`-q`, default 2048)
CPUMAP per-CPU ring/queue size.

### `--interval <seconds>` (`-i`, default 2)
Stats polling/print interval.

### `--extended` (`-e`)
Start in extended output mode (toggle with Ctrl-\).

### `--xdp-mode {native|skb|...}` (`-m`, default native)
XDP attach mode.

## The program matrix (BPF side)

`xdp_redirect_cpumap_state.bpf.c` generates a program per combination via
macros, and the loader selects one by name at load time:

- Ingress: `cpumap_state_no_touch[_<update>][_batched][_packed]_slot{0,1}`
- Remote:  `cpumap_state_drop[_<update>][_batched][_packed]_slot{0,1}`

So `{atomic|non_atomic} x {padded|packed} x {single|batched} x {slot0|slot1}`.
The loader builds the exact name from `state_update`, `state_layout`,
`state_batch_size`, and `state_mode` (slot 0 = shared, slot 1 = separate).

## Stats reported

Uses the shared `xdp_sample` infra with mask:
`RX_CNT | CPUMAP_ENQUEUE_CNT | CPUMAP_KTHREAD_CNT | EXCEPTION_CNT`, i.e. RX rate
on the ingress CPU, enqueue rate into the cpumap, kthread (remote) processing
rate, and redirect exceptions/errors.

## Typical usage for cross-NUMA eval

```sh
# baseline: redirect + drop, no state
xdp-bench redirect-cpu-state -c <remoteCPU> <ifname> --state-mode none

# worst-case contention on one shared cacheline
xdp-bench redirect-cpu-state -c <remoteCPU> <ifname> \
    --state-mode shared --state-update atomic --state-layout padded

# separate counters (no false sharing), compare same-NUMA vs cross-NUMA remoteCPU
xdp-bench redirect-cpu-state -c <remoteCPU> <ifname> \
    --state-mode separate --state-layout padded

# amortize state cost
xdp-bench redirect-cpu-state -c <remoteCPU> <ifname> \
    --state-mode shared --state-batch-size 16
```

Vary `-c` between a CPU on the same NUMA node as the RX CPU and one on a remote
NUMA node to quantify the cross-NUMA penalty of the shared cpumap state.
