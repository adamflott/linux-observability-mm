# Linux memory reader guide

`linux-observability-mm` provides typed Linux memory readers and parsers extending
`procfs`. Callers choose which readers to run and how to schedule or retain their
results. See the [README](../README.md) for examples and validation commands, and
the [implementation TODO](../TODO.md) for remaining work.

## Linux mm subsystems

Paths below use default `/proc` and `/sys` roots. Reader constructors also accept
captured trees or alternate roots; each type's Rustdoc describes its root, nested
values and conversions.

### Core readers

| Reader / subsystem | Reported data | Source |
| --- | --- | --- |
| `Meminfo`: host inventory/cache | RAM categories, dirty/writeback, huge-page totals; sizes converted to bytes. | `/proc/meminfo` |
| `Iomem`: physical address map | Labeled physical ranges; addresses may be redacted. | `/proc/iomem` |
| `VmStat`: reclaim/paging/writeback | Native VM gauges and cumulative counters; units vary by key. | `/proc/vmstat` |
| `BuddyInfo`: buddy allocator | Free blocks by node/zone/order; order `n` contains `2^n` base pages. | `/proc/buddyinfo` |
| `ZoneInfo`: zones/page caches | Inventory, watermarks, reserves and node/zone/CPU pagesets. | `/proc/zoneinfo` |
| `PageTypeInfo`: migration/pageblocks | Free blocks, pageblocks and optional mixed blocks; capped values retain lower bounds. | `/proc/pagetypeinfo` |
| `MemoryPressure`: memory PSI | `some`/`full` stall averages and total microseconds. | `/proc/pressure/memory` |
| `SharedMemorySegments`: System V IPC | Segment IDs, sizes, attachments and resident/swapped bytes. | `/proc/sysvipc/shm` |
| `Swaps`: swap backing | Device/file inventory, KiB size/usage and signed priorities. | `/proc/swaps` |
| `SlabInfo`: SLAB/SLUB objects | Object/slab counts, byte sizes, page geometry and optional legacy counters. | `/proc/slabinfo` format 2.1 |
| `MemorySysctls`: VM/kernel policy | VM settings; shared-memory, NUMA and perf settings; selected descriptor/inode/dentry counters. | `/proc/sys/vm/*`, selected `/proc/sys/{kernel,fs}` files |
| `CgroupMemoryStats`: memcg/DMEM/cpuset/HugeTLB | v2 usage, limits, stats, pressure, events, swap/zswap, device-region byte usage/capacity/limits, HugeTLB and allowed memory nodes; selected ancestor usage/limits. | Membership/mount discovery in `/proc/self/{cgroup,mountinfo}`, then `memory.*`, selected `dmem.*`, `hugetlb.*`, `cpuset.mems{,.effective}` |
| `HugePageStats`: THP/mTHP/HugeTLB | Policies, khugepaged/per-size counters and global/node pools. Counts remain size-specific. | `/sys/kernel/mm/{transparent_hugepage,hugepages}`, `/sys/devices/system/node/node*/hugepages` |
| `NumaStats`: locality/balancing/tiers/policy | Node memory/VM/locality tables, CPU lists/maps, distances, rated access-class latency (ns)/bandwidth (MiB/s), tier membership and global weighted-interleave weights/auto mode. | `/sys/devices/system/node` (including `node*/access*/initiators`), `/sys/devices/virtual/memory_tiering`, `/sys/kernel/mm/mempolicy/weighted_interleave` |
| `CompressedMemoryStats`: zswap/zram | Zswap policy, failure counters, stored/incompressible pages and zram device memory/I/O/backing-device statistics. | `/sys/module/zswap/parameters`, `/sys/kernel/debug/zswap`, `/sys/block/zram*` |
| `KsmStats`: identical-page sharing | Global/process counters, scanner policy, signed profit and flags. | `/sys/kernel/mm/ksm/*`, `/proc/self/{ksm_stat,ksm_merging_pages}` |
| `ProcessMemoryStats`: process VM/accounting | Status/peaks, limits, faults, OOM preferences and RSS/PSS/private/shared/swap rollup. | `/proc/self/{status,limits,stat,oom_score,oom_score_adj,smaps_rollup}` |

### Diagnostic readers

| Reader / subsystem | Reported data | Source |
| --- | --- | --- |
| `ProcessMemoryDetails`: mappings/NUMA | Address ranges, backing metadata, native quantities and VM flags. | `/proc/self/{maps,numa_maps,smaps}` |
| `DamonStats`: DAMON_STAT | Existing monitor state, estimated bytes/s, signed idle-time percentiles in ms, aggregation interval in us and worker PID. Disabled monitoring can leave stale results. | `/sys/module/damon_stat/parameters/*` |
| `ShrinkerStats`: reclaimable objects | Object counts by kernel cgroup identifier and possible-node column order; absent rows stay omitted, counts are not bytes. | `/sys/kernel/debug/shrinker/*/count` |
| `SlubStats`: object caches | Per-cache/node occupancy, policy/debug counters and canonical aliases. | Selected `/sys/kernel/slab/<cache>/*` |
| `KernelAllocationStats`: vmalloc/call sites | Area metadata and allocinfo bytes/counts/source locations/accuracy markers. | `/proc/{vmallocinfo,allocinfo}` |
| `FragmentationStats`: contiguous allocation | Per-node/zone/order indices in thousandths, including the `-1.000` sentinel. | `/sys/kernel/debug/extfrag/{unusable_index,extfrag_index}` |
| `LruStats`: Multi-Gen LRU | Policy, basic histograms and full cgroup/node/generation/tier statistics. Preserve page units, expiry/average/unavailable tags and optional MM counts; reviewed 6.12/7.2 layouts. | `/sys/kernel/mm/lru_gen`, `/sys/kernel/debug/{lru_gen,lru_gen_full}` |
| `PhysicalPageStats`: frame metadata | Bounded flags, mapping counts and charged cgroup inode numbers. | `/proc/{kpageflags,kpagecount,kpagecgroup}` |
| `ProcessPageMap`: process page tables | Bounded raw/decoded flags and permitted PFN/swap identities. | `/proc/self/pagemap` |
| `PageOwnerStats`: allocation provenance | Allocation/PFN/pageblock/stack records, migration/charge labels, aggregate handles/base-page counts and threshold; reviewed 6.12/7.2 layouts. | `/sys/kernel/debug/page_owner`, `page_owner_stacks/*` |
| `KernelLeakStats`: kmemleak | Uncertain unreferenced-object suspects: address/scope, byte size, PID/comm, allocation jiffies, checksum and stack. Embedded object bytes are discarded. | `/sys/kernel/debug/kmemleak` |
| `MemoryHardwareStats`: hotplug/EDAC/poison/CMA | Block state/size/zones, controller/DIMM errors, node recovery counters, CMA PFNs/counts/bitmaps. | `/sys/devices/system/{memory,edac/mc,node/node*/memory_failure}`, `/sys/kernel/debug/cma` |

`MemoryTraceParser` consumes caller-supplied ftrace text for `oom/mark_victim`,
direct-reclaim and compaction begin/end, and `migrate/mm_migrate_pages`.
`MemoryTraceRecord` retains event, loss and unknown records. Matched durations use
trace-clock nanoseconds and are not a complete latency census. Callers configure
tracefs, choose the clock and read the input; the parser performs no I/O.

### Reading rules

- Source reads are independent and non-atomic. Missing, denied and malformed
  sources remain errors or unavailable values, distinct from zero.
- Preserve native units: Linux `kB` means KiB; rollups offer checked `bytes()`
  conversion. Base-page counts require the actual page size. Do not sum overlapping scopes.
- NUMA access-class quantities describe rated topology for linked initiators;
  class 1 is CPU-only. Discovery caps at 64 classes per node, with 4 KiB per attribute.
- Shrinker count reads invoke callbacks across applicable cgroups/nodes and can
  repeat them during kernel buffering. Request/byte caps do not bound callback
  count or CPU time within a read. `scan` is excluded.
- Slab/page-type/debugfs/PFN reads may require privileges. smaps/rollups/page-type
  reads can be expensive; page-owner mixed-table reads can drain CPU page lists,
  and kmemleak reads may incur scans.
- Snapshot collectors do not change settings/membership or open process memory files.
  `memory.reclaim`, HugeTLB `demote`, write-only actions and `vm.stat_refresh` are excluded.
- Multi-file readers expose explicit proc/sys/process/debug-root constructors.
  Process roots are `/proc/<pid>`, not the proc mount. Captured cgroup mount paths
  are used as written.
- `MemoryFiles::parsed` uses relative paths; sysctl maps use dotted names.
  Structured values retain exact integers, units, policies and ranges. Unknown
  supported numeric keys survive; typed fields and filename allowlists have limits.

## Supported Linux kernels

Support follows available interfaces/formats. No universal minimum/maximum kernel
has been validated; kernel configuration, mounts, permissions and backports matter.

| Kernel / format | Recorded evidence |
| --- | --- |
| Debian `6.12.95+deb13-cloud-arm64` / `6.12.95+deb13-amd64` | Live fixtures for buddyinfo, zoneinfo, swaps, slabinfo and pagetypeinfo; 4 KiB base pages. |
| 4.19 zoneinfo / 6.1 legacy SLAB | Format review/synthetic coverage, not whole-crate live validation. |
| Other releases, including 7.x | Best-effort supported layouts; source review does not establish live-kernel support. |

See [fixture provenance](../tests/fixtures/README.md). Containers use the host/VM
kernel, regardless of installed headers.

| Interface | Upstream availability / current parser limit |
| --- | --- |
| cgroup v2 memory / cpuset / HugeTLB | 4.5 / 5.0 / 5.6; enabled controllers required; no v1 fallback. [manual][cgroups] |
| DMEM cgroups | 6.14; controller/device support required. Root capacity and child usage stay separate; limits preserve `max`. `dmem.peak` is presence-gated against the 7.3-rc7 development ABI. [ABI][dmem] |
| Weighted interleave | 6.9 node weights (1..=255); 6.16 `auto` mode. Read-only global policy observation; absent mode stays unknown. [ABI][weights] |
| `smaps_rollup` | 4.14; no implicit full-smaps fallback. [source][rollup] |
| Memory PSI | 4.20; PSI must be enabled. [source][psi] |
| MGLRU | 6.1; requires `CONFIG_LRU_GEN`, with optional debug statistics. [source][mglru] |
| Anonymous mTHP | 6.8; per-size files are optional/discovered. [source][mthp] |
| `allocinfo` | 6.10; accepts text formats 1.0/2.0. [source][allocinfo] |
| DAMON_STAT | 6.17; signed active-time percentiles/aggregation interval since 6.19, worker PID since 7.2. Requires `CONFIG_DAMON_STAT`; synthetic layout tests, no monitor enablement. [ABI][damon] |
| Pagemap flags | Modern 4.2+ layout; userfaultfd bit 57 since 5.13, guard bit 58 since 6.15. [ABI][pagemap] |

The reviewed 7.3-rc7 ABI reuses bit 57 for WP or RWP depending on the VMA.
`PageMapEntry::userfaultfd_protection(mapping.fields.get("VmFlags"))` interprets
matching smaps context; missing/conflicting modes stay unknown. The existing
`uffd_write_protected` field retains raw-bit semantics. Callers must match the
process, virtual page and capture time; independent reads can race.

[cgroups]: https://man7.org/linux/man-pages/man7/cgroups.7.html
[dmem]: https://github.com/torvalds/linux/blob/v7.3-rc7/Documentation/admin-guide/cgroup-v2.rst
[weights]: https://github.com/torvalds/linux/blob/v6.16/Documentation/ABI/testing/sysfs-kernel-mm-mempolicy-weighted-interleave
[rollup]: https://github.com/torvalds/linux/blob/v4.14/fs/proc/task_mmu.c
[psi]: https://github.com/torvalds/linux/blob/v4.20/Documentation/accounting/psi.txt
[mglru]: https://github.com/torvalds/linux/blob/v6.1/Documentation/admin-guide/mm/multigen_lru.rst
[mthp]: https://github.com/torvalds/linux/blob/v6.8/Documentation/admin-guide/mm/transhuge.rst
[allocinfo]: https://github.com/torvalds/linux/blob/v6.10/Documentation/mm/allocation-profiling.rst
[damon]: https://github.com/torvalds/linux/blob/v7.2/Documentation/admin-guide/mm/damon/stat.rst
[pagemap]: https://github.com/torvalds/linux/blob/v7.3-rc7/Documentation/admin-guide/mm/pagemap.rst

## Collection and retention limits

`CollectionLimits` configures one synchronous `collect_with_limits` scope on the
current thread. Use `read_source` for typed single-file readers inside that scope;
direct procfs `Current`/`FromRead` calls do not acquire the shared I/O budget.
Nested collection scopes are rejected, and the scope does not follow other threads.

| Default | Limit |
| --- | --- |
| Input per collection scope | 16 MiB |
| File / text line | 4 MiB / 64 KiB |
| Source attempts / directory entries | 65,536 each; at most 4,096 entries per directory |
| Path / visible cgroup ancestors | 4,096 bytes / 64 |
| Instrumented retained / scratch reservations | 64 MiB / 16 MiB per `HeapBudget` |
| Allocation overhead reserve | 64 bytes per admitted allocation |

`CollectionReport::limit_exceeded` marks incomplete collection. Oversized sources
fail instead of parsing a truncated prefix; EOF probes may read one extra byte.
Individual source errors remain separate from the report. Some readers enforce
additional bounds: `PageRange` admits 1 through 65,536 entries; provenance parsers
cap at 65,536 records, 256 stack frames and 4 KiB per line; shrinker collection
defaults to eight count requests. Trace parsing caps lines at 64 KiB and pending
duration matches at 4,096 task/family keys with 16 nested intervals per key.

Configure allocation allowances with `HeapLimits` and `HeapBudget::new`.
`HeapBudget::try_reserve` admits requested layouts plus the configured overhead;
leases release credit on drop, and `usage()` exposes live/peak reservations.
Clones share the budget. `with_heap_budget` supplies it to synchronous builders;
pass the shared budget explicitly to each worker thread.

Structured lists/maps/text, `MemoryFiles` entries/errors and smaps storage reserve
retained capacity; source buffers and borrowed-token lists reserve scratch.
Clones share leases, and replacement capacity is reserved while old storage lives.
Use fallible `ParsedMemoryValue::try_list`, `try_map`, `try_text` and container
builders. `MemoryFiles::allocation_error` preserves failure when an error-map
entry cannot fit.

Only instrumented allocations are charged. Other typed/third-party tables,
directory/path buffers, temporary sets/formatting, budget bookkeeping and Serde
buffering remain outside these counters. The overhead reserve is an accounting
policy. Input bytes and reservations do not measure allocator metadata or RSS.
Callers must separately bound output buffers and retained snapshot generations.
