# Memory Management Lock Contentions | Conference Talk Transcript

- Speaker: Barry Song (宋宝华)
- Date: July 5, 2026
- Slide deck: `mm lock contentions.pptx`
- Chinese version: [中文逐字稿](../zh/宋宝华-memory-management-lock-contentions.md)

![Slide 1: Memory Management Lock Contentions](../assets/宋宝华-memory-management-lock-contentions/slide-1.png)

## Opening

**Host:** You can just follow the PPT. Screen sharing should already be enabled. Advance the slides as you go.

**Barry Song:** All right, thanks. And thank you all very much for coming today.

The date on this slide deck is July 5. I started writing it at about eleven last night. Around midnight, I told myself I would lie down for five minutes. I did—and promptly fell asleep. So I finished the deck this morning.

I would like to give you a quick tour of several kinds of lock contention in the memory-management subsystem: some of the things we are working on, and some work by other developers in the community. I will mainly talk about four locks today: `mmap_lock`, the LRU lock, the zone lock, and `anon_vma->root->rwsem`.

![Slide 2: Four locks in memory management](../assets/宋宝华-memory-management-lock-contentions/slide-2.png)

## `mmap_lock`

![Slide 3: mmap_lock](../assets/宋宝华-memory-management-lock-contentions/slide-3.png)

**Barry Song:** First up is `mmap_lock`. This one is a little painful to talk about, because the community discussions have been quite a struggle for me.

Originally, `mmap_lock` was used all over the place. Then per-VMA locking was merged. In most cases, a page fault and another path that may modify the VMA can now synchronize using only the VMA lock.

There is still one path that can lead to `mmap_lock` contention. Suppose a page fault begins under the VMA lock. In principle, all we need is for the VMA to remain stable while the fault is being handled. But if the fault hits a file mapping, it may have to read from storage. The I/O queue may be long, and the filesystem may be doing something slow, such as garbage collection, so that wait can take a while.

When the existing fault path finds that a folio is not yet up to date and must wait for I/O, it drops whichever lock currently protects the fault—the VMA lock or `mmap_lock`—waits for the I/O to complete, and then retries the page fault. The problem is that the retry path still falls back to `mmap_lock`, bringing the contention back with it.

There are two ways we could address this.

The first is to keep page-fault retry. Before doing I/O, we release either the VMA lock or `mmap_lock`, whichever one we hold. Once the I/O is complete, we retry the fault under the VMA lock instead of falling back to `mmap_lock`. That removes this source of contention.

The second option is to remove this part of the page-fault retry machinery altogether. Since faults now mostly run under the VMA lock, we could keep that lock across the I/O wait, complete the entire fault, and avoid retrying it at all.

The main question is which of these two approaches we should take. I have been pushing the first one, while others in the community would rather pursue the second. That discussion is still ongoing.

## LRU lock

![Slide 4: LRU lock](../assets/宋宝华-memory-management-lock-contentions/slide-4.png)

**Barry Song:** LRU lock contention can also be quite severe in memory management. When we add a new folio to the page cache, for example, the folio eventually has to go onto an LRU list. Removing a folio from an LRU, or reclaiming memory, may also require taking the LRU spinlock.

To reduce that contention, the kernel maintains a per-CPU LRU cache. A newly added folio first goes into a batch on the local CPU. When the batch fills up—or when we encounter a large folio that the current batching scheme cannot hold—we drain the per-CPU cache into the LRU. That drain has to acquire the LRU lock, which is why it has long been a focus of contention work.

The community has discussed many possible designs. One idea is to give file-backed and anonymous pages different spinlocks. Another is to shard memory into regions: on a 32 GiB system, for example, we might use one lock for every 4 GiB. The trouble is that these designs also make the kernel substantially more complicated.

Recently, I came across two pieces of work that I thought were particularly good.

The first is JP Kobryn's patch, `mm/lruvec: preemptively free dead folios during lru_add drain`. While a per-CPU LRU cache is being drained, some folios are already effectively dead: the only reference left is the one held by the batch itself. There is no point adding such a folio to the LRU only to wait for reclaim to find it and free it again. If the batch owns the sole remaining reference, we can free the folio on the spot and avoid the pointless LRU operations and lock acquisitions.

The second is Usama Arif's patch, `mm/swap_state: remove unnecessary lru_add_drain() from readahead`. He found that the swap-in readahead path performs `lru_add_drain()` calls that serve no useful purpose.

Those two patches prompted me to look for similar cases elsewhere in the kernel. I found redundant drains in the anonymous-folio reuse paths as well—one in `wp_can_reuse_anon_folio()`, and another in `do_swap_page()`. Under certain conditions, these paths keep draining the per-CPU LRU cache even though no drain is really needed. That is what I have been working on recently, with Kairui helping review the changes.

I also see another opportunity here. Support for large folios in the per-CPU LRU cache is still limited. Whenever an unsupported large folio arrives, the cache first has to be drained. But if a system mainly uses relatively small large folios—order-2 or order-4 folios, for example—their size is still manageable. In principle, we should be able to batch them in the per-CPU LRU cache too.

I ran an experiment with twenty threads continuously triggering page faults. Once those smaller large folios were allowed into the per-CPU LRU cache, LRU lock contention fell substantially, and the multithreaded workload also completed faster. I am still working on this and will probably send another revision.

## Zone lock

![Slide 5: Zone lock](../assets/宋宝华-memory-management-lock-contentions/slide-5.png)

**Barry Song:** Zone lock contention may not be quite as fierce as the first two, because we already have PCP—the per-CPU pages mechanism. I had that wrong on the slide and am fixing it on the spot. I really did not get enough sleep last night; sorry about that.

Kefeng Wang previously posted a series that allowed selected high-order pages to be stored on PCP lists. Suppose a system mainly consumes order-4 folios: could we keep order-4 pages on the PCP list as well? His experiments showed less contention in certain workloads, but the improvement was not especially dramatic, so the work did not continue.

Still, I wonder whether the idea deserves another look. If a system is configured primarily around 64 KiB large folios, those folios may play much the same role that order-0 folios once did. Putting them on PCP lists could, in theory, reduce zone lock contention. On the other hand, a 64 KiB folio is already fairly large. The system needs fewer folios overall, so zone lock contention may already be lower than it was with order-0 folios, leaving less room for improvement. This is something we can continue to explore.

More recently, Xueyuan Chen and Wenchao Hao have been working on the `zs_free()` path in `zsmalloc`. This is a very hot path: both zswap and zram reach it when freeing objects.

The original `zs_free()` path involves both `pool->lock` and `class->lock`. When a complete zspage is released back to the buddy allocator, the path also enters the zone lock. In other words, `class->lock` and the zone lock can end up nested. Xueyuan and Wenchao split the two apart. While holding `class->lock`, the first half of the path merely records the zspage to be freed in a temporary variable. The actual zspage release happens only after `class->lock` has been dropped, at which point the pages can safely be returned to the buddy allocator. This prevents contention in one lock from propagating into the other.

I also saw Hongru Zhang tackle a related problem. Reading `/proc/pagetypeinfo` requires the kernel to take the zone lock and scan the free lists in each `free_area`, counting the available memory for every migratetype. That scan can take a very long time.

His proposal was to maintain per-migratetype counters directly in the buddy allocator. Then `/proc/pagetypeinfo` could read the counters instead of walking every free list under the zone lock. The trade-off is that maintaining those counters adds a small cost to the allocation and free paths, while `/proc/pagetypeinfo` is not necessarily a common hot path on every system. The community has not reached agreement on that trade-off yet.

## `anon_vma->root->rwsem`

![Slide 6: anon_vma root lock](../assets/宋宝华-memory-management-lock-contentions/slide-6.png)

**Barry Song:** The last one is the anonymous-VMA root lock, `anon_vma->root->rwsem`. Lorenzo Stoakes has talked about this problem before.

Suppose process A forks B and C, and B then forks D and E. After several generations of fork and COW, the anonymous VMAs of those processes form a fairly complicated reverse-mapping relationship.

Even after every page in one of process D's VMAs has gone through COW, D's anonymous VMA may still be attached to the original root. When pages in that VMA go through reverse-mapping operations such as `folio_referenced()` or memory reclaim, they still have to acquire the same `anon_vma->root->rwsem`. That concentrates contention on a single lock.

There is also a memory-footprint problem. A typical Android or Linux process may have thousands of VMAs, and allocating `anon_vma` and its related structures for all of them consumes a considerable amount of memory.

Wang Tao at Honor previously experimented with allocating these `anon_vma`-related structures lazily. Instead of building them for every VMA up front, the kernel would wait until an anonymous reverse mapping was actually needed. If I remember correctly, his tests on a typical 8 GiB Android device saved a meaningful amount of memory used by those structures. The work was difficult to move forward, but I still think it was a valuable direction to explore.

Lorenzo now wants to rewrite anonymous reverse mapping. David once joked that only three people in the world understand the rmap code, which gives you some idea of how complicated this part of the kernel is.

The central idea in Lorenzo's work is to stop maintaining the existing `anon_vma` relationships at VMA granularity. Instead, the mechanism would move up to `mm_struct` granularity, with a COW context for each address space. If A forks B, for example, B's COW context would have some relationship with A's context. There are still many difficult corner cases to work through, and the implementation is still being written.

This would be a very large change. That is about all I wanted to cover today. I put the talk together in a hurry, so there are probably details I have not thought through completely. Thank you very much. If there are questions, we still have time for one or two.

## Closing remarks from the host

**Host:** Many thanks to Barry for the talk. A number of people in our group and across the community have contributed to the work he described. We have spent quite a few weekends digging into these lock-contention problems ourselves.

Barry has also written about some of these topics on his WeChat public account. If you have questions, please keep the discussion going there. We will continue organizing material from this event and discussing these issues in the community, with the goal of moving both the technical conversation and the code forward. Thank you, Barry.

## Mailing-list Threads and Further Reading

The links below correspond to patches or discussions referenced in the slides. Patch series continue to evolve on the mailing lists, so a linked revision may differ slightly from the version shown in the talk.

### LRU lock

- [JP Kobryn: `mm/lruvec: preemptively free dead folios during lru_add drain`](https://www.spinics.net/lists/kernel/msg6170487.html)
- [Usama Arif: `mm/swap_state: remove unnecessary lru_add_drain() from readahead`](https://lkml.iu.edu/2606.1/01456.html)
- [Barry Song: `mm: drop redundant lru_add_drain in anon folio reuse paths` (V2 thread)](https://www.spinics.net/lists/kernel/msg6279924.html)

### Zone lock

- [Kefeng Wang: `mm: allow more high-order pages stored on PCP lists`](https://lore.kernel.org/linux-mm/20240415081220.3246839-1-wangkefeng.wang@huawei.com/)
- [Xueyuan Chen: `mm/zsmalloc: drop class lock before freeing zspage`](https://lore.kernel.org/all/20260626015003.2965881-4-haowenchao22@gmail.com/)
- [Hongru Zhang: `mm: add per-migratetype counts to buddy allocator and optimize pagetypeinfo access`](https://lore.kernel.org/all/cover.1764297987.git.zhanghongru@xiaomi.com/)

### Anonymous reverse mapping

- [Lorenzo Stoakes: Keeping COWs in context (a.k.a. anonymous reverse mapping)](https://lwn.net/Articles/1072378/)
