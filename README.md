# Marie LRU

**Multi-graded Adaptive Reclaim & Independent Eviction**

Marie is an out-of-tree, third-pillar LRU implementation for the Linux kernel, sitting alongside Legacy LRU and MGLRU rather than replacing either.  It composes three established ideas — working-set protection ([le9uo](https://github.com/firelzrd/le9uo)), proportional anon/file aging under a generational ring ([Re-swappiness](https://github.com/firelzrd/re-swappiness)), and async swap-out offload ([kcompressd-unofficial](https://github.com/firelzrd/kcompressd-unofficial)) — onto a redesigned core: per-lruvec lock-free pending queues, a bounded generation ring without forced oldest-gen drain, and a SIMD-accelerated PTE walker driven by rmap-to-walker bloom-filter feedback.

Marie's paths are gated behind a runtime static key (`lru_marie_enabled()`), so a kernel built with `CONFIG_LRU_MARIE=y` but Marie turned off behaves exactly as MGLRU / Legacy LRU would.

le9uo: https://github.com/firelzrd/le9uo
Re-swappiness: https://github.com/firelzrd/re-swappiness
kcompressd-unofficial: https://github.com/firelzrd/kcompressd-unofficial

---

## Background

Linux's page-reclaim LRU has gone through two main eras.

**Legacy LRU** (the upstream default before MGLRU, and still the fallback) maintains per-memcg active/inactive list pairs.  Folios are promoted on `PG_referenced` and demoted on scan-tail age-out.  The model is small and cheap, but the binary active/inactive boundary is coarse — within each bucket there is no further ordering, so when the inactive list isn't large enough to span the working set, refault-driven re-promotion turns into a thrash treadmill.

**MGLRU** (`lru_gen`, mainlined in 6.1) replaced the two-bucket model with a generational ring, actively harvested by a PTE walker rather than relying solely on rmap-side `PG_referenced`.  This delivered a large improvement on workloads where the active/inactive cut sliced through the working set, and remains the right baseline for most general-purpose use.

Marie is a third design point.  The axes where MGLRU's choices turn into trade-offs under specific workloads — and where Marie picks the other end:

Legend — ✅ behaves as expected · ⚠️ works but with a structural trade-off · ❌ not provided / cannot deliver the property

| Aspect                | Legacy LRU                                                                | MGLRU                                                                                                       | Marie                                                                                                       |
| :-------------------- | :------------------------------------------------------------------------ | :---------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------------------------- |
| Aging granularity     | ⚠️ binary active / inactive only                                           | ✅ 4-gen ring (`MAX_NR_GENS=4`); a full ring triggers a forced `inc_min_seq` oldest-gen drain                 | ✅ 4-gen ring (`MARIE_MAX_NR_GENS=4`); advance skips when the ring is full, no forced drain                  |
| anon/file iteration   | ✅ per-type lists; `vm.swappiness` works as a proportional anon:file ratio | ❌ shared `max_seq` across anon/file reduces `vm.swappiness` to a per-generation hint                        | ✅ per-type `seq` / `min_seq` / ring; restores `vm.swappiness` as a proportional ratio                       |
| Hot-signal harvest    | ⚠️ rmap-driven `PG_referenced` only; no active walker                      | ✅ PTE walker + walker-internal PMD bloom filter + rmap look-around cascading `PG_referenced`                | ✅ Vectorized PTE walker + rmap-fed PMD bloom filter + rmap look-around updating a saturating tier bit only            |
| Compression stall     | ⚠️ kswapd blocks inside `zswap_store`                                      | ⚠️ kswapd blocks inside `zswap_store`                                                                        | ✅ offloaded to per-node `kcompmari` kthread                                                                 |
| Clean-file floor      | ❌ none                                                                    | ❌ none                                                                                                      | ✅ `clean_min_ratio` hard floor against thrashing                                                            |
| Reclaim-livelock OOM  | ❌ retries until `MAX_RECLAIM_RETRIES`                                     | ❌ retries until `MAX_RECLAIM_RETRIES`                                                                       | ✅ early-OOM via `oom_min_free_kbytes` + swap-write-failure gate                                             |

These are deliberate choices on MGLRU's side: fewer state bits per folio, fewer fast-path branches, a single iteration counter shared between memory types, a self-contained walker that owns its own feedback.  Marie's complementary design accepts the resulting state cost in exchange for proportional `vm.swappiness` control under a generational ring, hard working-set protection, and a hot-signal pipeline that pulls the rmap into the loop instead of letting it cascade `PG_referenced` onto neighbouring folios.

---

## Lineage

```
              le9uo           Re-swappiness        kcompressd-unofficial
                │                   │                       │
        clean_min_ratio,        per-type            kcompressd kthread +
        oom_min_free_kbytes     seq / min_seq       FIFO write-back
                │                   │                       │
                └─────────┬─────────┴───────────────────────┘
                          │
                       Marie LRU
                          │
                  (own pipeline core:
                   residency ADT,
                   bounded gen ring w/o
                   forced inc_min_seq,
                   SIMD PTE walker,
                   rmap-fed bloom)
```

Marie keeps the **goals** of its predecessors but doesn't reuse their implementations verbatim — it re-derives each idea inside its own LRU core so the surrounding state (gen ring, lock topology, scan order) is internally consistent.

- From **le9uo**: hard working-set protection (`clean_min_ratio`) and early-OOM watermark (`oom_min_free_kbytes`).  `anon_min_ratio` and `clean_low_ratio` from le9uo are intentionally **not** ported — under Marie the former is largely redundant with `vm.swappiness` once `kcompmari` brings anon refault sub-millisecond, and the latter overlaps with the existing `swap_tokens` bucket.
- From **Re-swappiness**: per-type `seq` / `min_seq` so `vm.swappiness` acts as a proportional anon:file scan ratio under a generational ring (the property Legacy LRU already has and MGLRU loses by sharing `max_seq`).  Re-swappiness ports this onto MGLRU's structure; Marie owns its own per-type state inside `marie_lruvec` and does not depend on MGLRU being per-type.
- From **kcompressd-unofficial**: per-node async compression FIFO so kswapd doesn't block in `zswap_store` / `__swap_writepage`.  Marie carries this over as `kcompmari` (renamed from `kcompressd` for Marie's namespace) with the same per-node FIFO design and the same 0–256 depth knob, default 24.

---

## Architecture

```
                  fault path
                       │
                       ▼
              ┌────────────────────────┐
              │ marie_add_queue        │   lock-free per-CPU llist
              │  (PENDING_ADD)         │   drained at watermark or
              └───────────┬────────────┘   from direct reclaim
                          │
                          ▼
       rmap     ┌────────────────────────┐    tail-gen pick
      look- ──▶ │ marie_lruvec (sl)      │ ───────────────┐
      around   │  per-(memcg, node)     │                │
       tier++   │  per-type seq/min_seq │                ▼
       │       │  4-gen ring, no forced │     ┌─────────────────────┐
       │       │  inc_min_seq drain     │     │ marie_natural_sort  │
       │       └───────────┬────────────┘     │ + shrink_folio_list │
       │                   ▲                  └──────────┬──────────┘
       │                   │                             │
       │   tier++ on young │                             │
       │   PTE (SIMD scan) │                       anon  │  file
       │                   │                             ▼
       │   ┌──────────────────────────┐       ┌──────────────────────┐
       └──▶│ pmd-bloom (per-pgdat)    │       │ kcompmari kthread    │
           │  Producer: rmap         │       │  per-node FIFO       │
           │  Consumer: SIMD walker  │       │  zswap_store /       │
           └────────────┬─────────────┘       │  __swap_writepage   │
                        ▲                     └──────────────────────┘
           ┌────────────┴─────────────┐
           │ SIMD PTE walker          │
           │  per-pgdat, adaptive     │
           │  interval (critical →    │
           │  idle), boot-dispatched  │
           │  AVX-512 / AVX2 / SSE2 / │
           │  scalar                  │
           └──────────────────────────┘
```

- `marie_lruvec` (`sl`) is Marie's per-(memcg, node) state container — the rough analogue of MGLRU's `lru_gen_folio` but with **independent per-type rings** and a ring-advance policy that **skips on full** rather than forcing an oldest-gen drain.
- Hot-signal pickup is split across rmap and the walker.  MGLRU also uses a bloom filter, but it's walker-internal — a self-feedback to skip PMDs that the walker itself did not touch on the previous pass.  Marie's bloom is cross-component: rmap is the producer (the PTL-bounded look-around flags PMDs that just took a young hit), the walker is the consumer (it pays the SIMD scan + tier++ cost only on bloom-hit PMDs).  This lets a hot signal observed by rmap influence the walker within the same reclaim window, not the next one.

---

## Design

### Reclaim pipeline: residency ADT + lock-free pending queues

Marie tracks a folio's lifecycle as a 4-state machine derived from 3 bits in `folio->flags` plus the topology of `folio->lru`:

| State             | `MARIE_TRACKED` | `folio->lru`        | Where                          |
| :---------------- | :-------------- | :------------------ | :----------------------------- |
| `NONE`            | 0               | —                   | not owned by Marie             |
| `PENDING_ADD`     | 1               | linked to add queue | on per-CPU `marie_add_queue`   |
| `RESIDENT`        | 1               | self-loop in gen    | inside `marie_lruvec` gen ring |
| `PENDING_PROMOTE` | 1               | linked to promote queue | hot-saturated, awaiting drain |

`MARIE_TIER` (2 bits) is a saturating 0..3 hotness counter, bumped by the walker on young-PTE hits (lock-free `CMPXCHG`) and by the explicit access path `lru_marie_mark_accessed`.

The pending queues (`marie_add_queue`, `marie_promote_queue`) are lock-free per-CPU `llist`s.  They absorb fault-burst spikes without touching the per-lruvec lock; a watermark-triggered background worker (`marie_drain_wq`) or direct reclaim drains them in bulk into the appropriate gen.

### Gen-ring advance policy: skip on full, no forced drain

MGLRU's ring has `MAX_NR_GENS=4`.  When it fills, aging calls `try_to_inc_min_seq()` to push the oldest gen forward before cutting a new head.  If that oldest gen holds folios that won't move — clean file pages under a working-set policy, mlocked anon, etc. — aging still demands forward progress, so the same untouchable folios get rewalked over and over: the *forced `inc_min_seq` treadmill*.

Marie keeps the same 4-gen cap (`MARIE_MAX_NR_GENS=4`) but `marie_advance_max_seq()` simply returns `-EAGAIN` when the ring is full — no forced drain, no `inc_min_seq` retry on the oldest gen.  Aging callers (`marie_age_on_add()` on head growth, `marie_age()` at every isolation entry) treat the failure as "not now" and continue.  The ring is unstuck only by **reclaim consuming the tail**: when the tail-most gen empties, `marie_try_drop_oldest()` returns its storage to the per-type spare pool, freeing a ring slot.  No explicit retry queue is needed because both callers naturally re-probe the cap at fault rate and at every reclaim cycle.

ADD is not affected by the cap in steady state — once the ring has any gen, `marie_get_or_create_gen()` returns the existing head without touching `advance`.  The rare degenerate case (gen-spare empty + GFP_ATOMIC failure on the very first head) is also covered: `__marie_install_locked()` is a no-op on failure, the wrapper stays on `add_queue` for the next drain to retry, and Phase 1b of `marie_isolate_folios()` routes the folio straight to reclaim if installs keep failing.

Practical effect: an oldest gen full of `clean_min_ratio`-protected folios is left alone.  Aging stops growing new heads until reclaim drains the existing tail (or `clean_min_ratio` diverts the type selector to anon).  The treadmill never starts because nothing in Marie demands aging-side progress regardless of consumer state.

### Independent anon/file pipelines

`marie_lruvec` keeps **fully independent** per-type state — each of ANON and FILE has its own monotonic `seq`, its own oldest sequence (`min_seq`), its own generation ring, and its own per-type `sl_lock` half.  There is no shared iteration counter between the two types, so `vm.swappiness` works as a proportional anon:file scan ratio — the behaviour Legacy LRU has always had, recovered inside a generational ring.  This is the philosophical successor of Re-swappiness, rebuilt around Marie's own ring rather than as a patch over MGLRU's structure.

### Hot-signal harvest

- **SIMD PTE walker** (per-pgdat).  On x86-64, `arch_initcall` picks the widest available SIMD instruction set (AVX-512F > AVX2 > SSE2) and flips a static branch so the walker's young-bit extraction has no indirect call.  ARM64 and other arches use a scalar fallback for now.
- **FPU batching**.  `kernel_fpu_begin/end` are amortised across `MARIE_FPU_BATCH=16` consecutive bloom-hit PMDs, then released around the `walk_page_range` boundary so the preempt-disabled window stays bounded by `MARIE_FPU_BATCH`, not by walk length.
- **rmap look-around**.  Called from `folio_referenced_one()`, this scans up to `BITS_PER_LONG` PTEs around the target folio's PMD under the existing PTL.  Unlike MGLRU's look-around it does **not** set `PG_referenced` on neighbouring folios — that cascade is the main source of reclaim starvation under fault-heavy workloads.  Instead it feeds the walker via a bloom filter (below).
- **rmap-fed PMD bloom feedback**.  MGLRU's bloom filter is walker-internal: the walker remembers which PMDs it touched last pass and uses that to skip cold PMDs next pass.  Marie's bloom is cross-component — rmap is the producer ("this PMD had a young PTE on its target"), the walker is the consumer ("only scan PMDs the bloom flagged").  The filter is double-buffered and swapped at the end of each walker pass, so each pass acts on the rmap signal accumulated during the previous reclaim window.
- **Pressure-adaptive cadence**.  The walker re-evaluates its scan interval (`walker_interval_{critical,low,normal,idle}_ms`) on each pass.  Defaults: `HZ/30`, `HZ/10`, `HZ/4`, `HZ` — all tunable.

### Pressure resilience

- **`clean_min_ratio` (hard working-set floor).**  At reclaim time Marie diverts file → anon selection when clean file pages would otherwise be evicted below the configured percentage of node RAM. Equivalent in spirit to le9uo's knob of the same name.  Default 15%; set to 0 to disable.
- **`oom_min_free_kbytes` (early-OOM).**  Inside `should_reclaim_retry()`, Marie aborts the retry loop and lets the OOM killer fire as soon as either: (a) free RAM + free swap drops below the threshold, or (b) the swap backend has rejected more than `MAX_SWAP_WRITE_FAIL_RETRIES` writes during this allocation attempt (e.g. ZRAM `zs_malloc` starvation). Defaults to ~1% of total RAM, clamped to [4 MiB, 512 MiB], or whatever's set via `sysctl vm.oom_min_free_kbytes`.
- **`kcompmari` async swap-out.**  Per-node kernel thread that drains a `kfifo` of anon folios deposited by kswapd, running `zswap_store` / `__swap_writepage` off the reclaim critical path. Same depth knob as kcompressd-unofficial: 0–256, default 24, 0 disables the offload.

### Coexistence

- **Static-key gate.**  Every Marie entry point — `lru_marie_add_folio`, `lru_marie_look_around`, `lru_marie_shrink_lruvec`, `lru_marie_age_node`, `lru_marie_mark_accessed`, `lru_marie_extra_ref_count` — is fronted by `lru_marie_enabled()`, a `DEFINE_STATIC_KEY_TRUE`.  When Marie is runtime-disabled the branch is patched out, so MGLRU / Legacy users see zero added instructions on the hot paths.
- **3-bit folio-flag reservation.**  `MARIE_BITS_WIDTH` reserves 3 bits below `LRU_REFS_PGOFF`.  `LRU_REFS_WIDTH` shrinks accordingly when `CONFIG_LRU_MARIE=y`.
- **VMA-affinity sort.**  Before handing an isolated batch to `shrink_folio_list`, Marie groups folios by mapping so that the per-folio rmap walks inside reclaim reuse the VMA cache instead of bouncing between mappings.  The kernel ships a generic `lib/list_sort` (`O(n log n)` Timsort-style merge sort), but the isolation batch comes off the LRU in near-ascending mapping order to begin with — adjacent folios in the gen ring usually share a VMA — so a generic comparison sort does far more work than necessary.  Marie uses a custom **natural merge sort** (`marie_natural_sort`) that detects existing ascending runs and merges only the run boundaries, giving best-case **O(n)** on a fully-sorted batch and degrading gracefully toward O(n log n) only when affinity is genuinely shuffled.  The cost saved on sorting is paid back as VMA-cache hits on the downstream `shrink_folio_list` walk.

---

## Tunables

All knobs live under `/sys/kernel/mm/lru_marie/` and accept runtime writes.  The OOM watermark is the one exception — it's a global sysctl.

| Knob                          | Default       | Range / unit                | What it does                                                                                                 |
| :---------------------------- | :------------ | :-------------------------- | :----------------------------------------------------------------------------------------------------------- |
| `enabled`                     | `1`           | 0 / 1, static key           | Master gate.  `0` patches Marie out of the hot paths; MGLRU / Legacy take over.                              |
| `simd`                        | `1`           | 0 / 1, static key           | Toggles SIMD young-bit scan.  `0` falls back to scalar (A/B testing).                                        |
| `clean_min_ratio`             | `15`          | 0–100, %RAM                 | Hard floor on clean file pages.  Below this, reclaim diverts to anon instead of evicting file.               |
| `kcompmari`                   | `24`          | 0–256, FIFO depth           | Max queued folios per `kcompmari` thread.  `0` disables the offload (falls back to synchronous swap-out).    |
| `simd_alg`                    | auto          | read-only                   | Reports the chosen SIMD ISA (`avx512f` / `avx2` / `sse2` / `scalar`).                                        |
| `shrink_batch_threshold`      | `256`         | folios                      | Max folios isolated per shrink batch.                                                                        |
| `gen_growth_threshold`        | `8192`        | folios (`SWAP_CLUSTER_MAX << 8`) | Head-generation growth limit before a new gen is cut.                                                   |
| `add_watermark`               | `256`         | folios                      | Per-CPU `add_queue` length that triggers a background drain.                                                 |
| `walker_interval_critical_ms` | `HZ/30` (~33 ms) | ms                       | Walker cadence under critical memory pressure.                                                               |
| `walker_interval_low_ms`      | `HZ/10` (100 ms) | ms                        | Walker cadence under low pressure.                                                                           |
| `walker_interval_normal_ms`   | `HZ/4` (250 ms) | ms                         | Walker cadence under normal pressure.                                                                        |
| `walker_interval_idle_ms`     | `HZ` (1000 ms)  | ms                         | Walker cadence when idle.                                                                                    |

Global sysctl:

| Sysctl                       | Default                       | What it does                                                                                                |
| :--------------------------- | :---------------------------- | :---------------------------------------------------------------------------------------------------------- |
| `vm.oom_min_free_kbytes`     | ~1% of RAM, clamped 4–512 MiB | Free-RAM + free-swap floor below which `should_reclaim_retry()` gives up and the OOM killer fires.  `0` disables. |

Boot cmdline: `lru_marie=0` disables Marie at boot (equivalent to writing `0` to `enabled` immediately after boot).

---

## Build & install

The `patches/` directory contains tested patches against tagged kernel bases:

```
patches/test/0001-linux6.18.22-lru_marie-0.1.0.patch
patches/test/0001-linux7.0-lru_marie-0.1.0.patch
patches/test/0001-linux7.1-rc1-lru_marie-0.1.0.patch
```

To apply against a matching source tree:

```
cd /path/to/linux
patch -p1 < /path/to/lru_marie/patches/test/0001-linux<base>-lru_marie-0.1.0.patch
make olddefconfig    # answer Y to CONFIG_LRU_MARIE
make -j$(nproc)
```

`CONFIG_LRU_MARIE` defaults to `y` (Marie depends on `CONFIG_MMU`). `CONFIG_LRU_GEN` and `CONFIG_LRU_GEN_ENABLED` should stay as they were — Marie does not replace MGLRU at build time, only at runtime.

To verify Marie is active after boot:

```
cat /sys/kernel/mm/lru_marie/enabled       # 1 = active
cat /sys/kernel/mm/lru_marie/simd_alg      # avx512f / avx2 / sse2 / scalar
```

---

## Status & roadmap

| Area                      | State            | Notes                                                                                                                |
| :------------------------ | :--------------- | :------------------------------------------------------------------------------------------------------------------- |
| x86-64 (AVX-512F)         | ✅ working        | preferred SIMD path                                                                                                  |
| x86-64 (AVX2 / SSE2)      | ✅ working        | auto-selected when AVX-512F absent                                                                                   |
| ARM64                     | ⚠️ scalar only    | NEON walker pending FPU save/restore profiling vs. the scalar baseline                                               |
| Other arches              | ⚠️ scalar only    | functional, no SIMD acceleration                                                                                     |
| THP (large folios)        | ⚠️ partial        | large folios bypass the lock-free pending queues and use a synchronous install path; full pending-queue THP planned  |
| Page-level memcg reparent | ❌ not wired      | Marie state for dying memcgs is freed via `lru_marie_exit_memcg()` at css_free; no page-level migration to parent     |

Known kernel bases with Marie 0.1.0 patches in this repo: 6.18.22, 7.0, 7.1-rc1.  All three ports share the same Marie source tree (`mm/lru_marie*.c/.h`, `include/linux/lru_marie*.h`) — the patches differ only in base-kernel-side adaptations.  In particular, the batched no-flush young-PTE API used by Marie's walker is native on 7.1-rc1 (`test_and_clear_young_ptes` / `test_and_clear_young_ptes_notify`), absent on 7.0 (the 7.0 patch back-ports it into `include/linux/pgtable.h` and `include/linux/mmu_notifier.h`), and absent on 6.18 (the 6.18 patch back-ports the same API plus thin `lazy_mmu_mode_enable/disable` wrappers, since 6.18 only has `arch_enter_lazy_mmu_mode` and lacks the 7.0 reentrancy-tracking helpers).

---

## License

GPL-2.0, same as the Linux kernel.

## References

- [le9uo](https://github.com/firelzrd/le9uo) — working-set protection
- [Re-swappiness](https://github.com/firelzrd/re-swappiness) — anon/file separation under MGLRU
- [kcompressd-unofficial](https://github.com/firelzrd/kcompressd-unofficial) — async swap-out offload; based on the original **Kcompressd** idea by Qun-Wei Lin (MediaTek) 
