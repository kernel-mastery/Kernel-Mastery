# DAMON in Production: Experiences, Gaps, and Future Directions | Meeting Transcript

- Speakers: SeongJae Park (SJ), Lian Wang, and Kunwu Chan
- Date: July 5, 2026
- Slides: `DAMON-Kernel-Subsystem-for-Data-Access-Monitoring-and-Access-aware-System-Operations.pdf` and `DAMON-in-Production_王炼_陈涛(KunWuChan).pdf`
- Chinese version: [中文逐字稿](../zh/SeongJae-Park-王炼-陈涛-damon-in-production.md)

> The images in the first part come from SeongJae Park’s DAMON overview deck in this repository (the FOSDEM’25 version), which covers the same fundamentals discussed in this talk. The second part uses the production deck prepared for this event.

![Slide 1 of the DAMON overview deck: the DAMON kernel subsystem](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-1.png)

## Opening

**Host:** Our next speaker is SeongJae Park, who will introduce DAMON.

**SeongJae Park:** Thank you for the introduction. It is very nice to meet the Chinese kernel developers here.

Today I will briefly introduce what DAMON is and what projects are going on. I will first explain why memory efficiency matters, then what DAMON is today, and finally what it may become in the near future.

## Why memory efficiency matters

People love data. Data is where money and fun come from. Memory, meanwhile, is where data lives, but memory costs money and consumes resources.

Modern systems often have a memory hierarchy: smaller and faster memory at the top, and larger but slower memory below it. That gives us room to improve efficiency. Efficient memory management can be understood as keeping precious data in the smaller, faster memory.

So what counts as precious data? A simple answer is frequently accessed data. We keep data because we want to access it. If we will never access it again, it no longer needs to occupy precious memory.

![Slide 3 of the DAMON overview deck: data accesses across memory space and time](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-3.png)

That is the background to DAMON, the Linux kernel subsystem for data access monitoring and access-aware system operations.

## What DAMON is today

Once upon a time, at a cloud company far, far away, a team wanted to build virtual machines whose memory could scale automatically. They wanted each VM to use only as much memory as it actually needed. To make that work, the system first had to know which data could be removed and which data should remain in memory. The team developed what later became DAMON in the Linux kernel.

At a high level, DAMON can be summarized with a short loop. A kernel thread called `kdamond` periodically checks whether each monitored region has been accessed. In a typical configuration, it samples every 5 milliseconds and aggregates the results every 100 milliseconds. Those intervals can also be tuned automatically.

![Slide 51 of the DAMON overview deck: DAMON v5.15 pseudocode](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-51.png)

DAMON tells you where a memory region is, how frequently it is accessed, how stable that access pattern is, and how recently it was accessed. Adaptive region construction, sampling, and aggregation keep the cost under control. Users can set an upper bound on overhead. Whether the monitored system has 10 GiB or 10 TiB of memory, DAMON makes a best effort to deliver the most accurate result it can within that bound.

On production workloads, we have seen overhead below 0.1% of a single CPU. The exact number still depends on the configuration, and users can choose their own trade-off between overhead and accuracy.

![Slide 25 of the DAMON overview deck: a DAMON monitoring snapshot](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-25.png)

DAMON has been in mainline Linux since v5.15. It is enabled in many distribution kernels and has also been backported to older kernels. It is now used in both research and production.

![Slide 35 of the DAMON overview deck: availability](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-35.png)

### DAMOS: turning observations into actions

DAMON also provides DAMOS, short for DAMON-based Operation Schemes. A user can ask DAMON to find memory with a particular access pattern and apply an operation to it. For example, DAMOS can page out cold memory or migrate hot and cold pages between NUMA nodes.

![Slide 31 of the DAMON overview deck: DAMOS](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-31.png)

This makes DAMON more than an access monitor. It is also an engine for access-aware system operations.

### The DAMON community

DAMON has a public mailing list and a project website. You can also email me directly. Every two weeks, the community holds an informal online DAMON Beer/Coffee/Tea chat. If you have a question or want to discuss a patch, please use any of those channels.

![Slide 37 of the DAMON overview deck: DAMON community channels](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-37.png)

## The future: from data accesses to data attributes

In one sentence, DAMON today is a Linux kernel subsystem for efficient data access monitoring and access-aware system operations. The next step is to make it an efficient data-attribute monitoring and operations engine.

![Slide 48 of the DAMON overview deck: DAMON’s extensible layers](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-48.png)

Users are already asking for more detailed information. Some want to monitor only a particular cgroup, CPU, or thread. Others want reads and writes distinguished. Those requirements go beyond access frequency and point toward more general data attributes.

![Slide 53 of the DAMON overview deck: extending DAMON beyond snapshot access patterns](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-53.png)

Some users also want more powerful monitoring primitives. DAMON currently relies mainly on page-table Accessed bits. They work well for memory-level monitoring but provide limited detail and precision. Hardware facilities such as AMD IBS, Intel PEBS, and Arm SPE can provide more specialized data.

There have been proposals for page-fault-based monitoring, including read-only, write-only, and per-CPU monitoring, as well as proposals to extend DAMON to perf events. More specific work for Arm SPE, AMD IBS, and Intel PEBS is also under discussion. Lian’s recent Arm-oriented proposal is one example.

![Slide 54 of the DAMON overview deck: monitoring primitives beyond PTE Accessed bits](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-54.png)

Our plan has two stages. First, we want to evolve DAMON from a data-access monitoring and operations engine into a general data-attribute monitoring and operations engine by the end of this year. We then plan to build support for hardware facilities such as AMD IBS, Intel PEBS, and Arm SPE on top of that foundation during the first half of next year.

That is all for my presentation. Thank you.

![Slide 38 of the DAMON overview deck: summary](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-kernel-slide-38.png)

## Starting from a production problem

![Slide 1 of the production deck: DAMON in Production](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-1.png)

**Lian Wang:** Kunwu Chan and I have been working on this together. While developing the feature, we discussed DAMON’s longer-term direction with SJ, so we invited him to begin with the broad picture. I will continue from our patch series and talk about the production problem we found and where we want to take the work next.

DAMON already serves many use cases well. This work began with a production issue reported by Sangfor: after THP was enabled, DAMON’s hot/cold region sampling became inaccurate. We added mTHP support around that problem, ran a number of experiments, and posted the patches to the mailing lists. We are now preparing the next revision based on review feedback.

![Slide 2 of the production deck: two gaps, the fixes, and SPE assistance](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-2.png)

## A quick DAMON refresher

DAMON traditionally uses the page-table Accessed bit, or AF, to tell whether memory was accessed. It adaptively divides the address space into regions, samples them, and tracks access frequency and `age`. DAMOS can then trigger actions such as `pageout` from an access pattern. Our work adds large-folio `collapse` and `split` capabilities.

![Slide 3 of the production deck: a quick DAMON refresher](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-3.png)

## Two known limitations

### T1: the TLB-reach blind spot

![Slide 4 of the production deck: the TLB blind spot](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-4.png)

The Kunpeng 920 has 2,048 L2 TLB entries. With 2 MiB THPs, that gives a theoretical reach of 4 GiB. If the working set fits within that range, the CPU may keep hitting in the TLB and stop walking the page tables. After DAMON clears the AF, no new page-table walk occurs to set it again. DAMON then keeps seeing `nr_accesses=0`, and schemes gated by the access count do not run.

This is not Arm-specific. `ptep_test_and_clear_young()` does not flush the TLB for this operation on either x86_64 or arm64, so the blind spot is architecture-independent. We treat T1 as a laboratory observation, not as the production incident itself.

A practical workaround is to set `min_nr_accesses` to 0, allowing the scheme to run even when the AF remains clear, and then use an address filter to select the exact target range.

### T2: PMD-granularity inflation

![Slide 5 of the production deck: PMD-granularity inflation](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-5.png)

T2 is the issue observed in production. A 2 MiB THP contains 512 4 KiB subpages but has only one PMD-level AF. An access to any one subpage makes the entire 2 MiB range appear hot to DAMON. DAMON cannot distinguish a uniformly hot THP from one with a single hot 4 KiB page, so cold data can hide inside an apparently hot THP.

We used Arm SPE to analyze a Kunpeng 920 host running KVM and Oracle DB. The result showed that 94.6% of the sampled 2 MiB THPs had fewer than 10% of their subpages accessed. Many THPs were highly uneven internally, and a PMD-level AF substantially inflated the amount of memory reported as hot.

![Slide 6 of the production deck: production data validated with Arm SPE](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-6.png)

## Adding order-aware collapse and split to DAMOS

Our v2 series contains five patches.

![Slide 7 of the production deck: five patches for mTHP collapse and split](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-7.png)

The first three extend the existing `DAMOS_COLLAPSE` action. They add `target_order` to `struct damos` and expose it through sysfs, export `damon_collapse_folio_range()` from `khugepaged`, and wire an mTHP-aware collapse handler into DAMON’s virtual-address operations set.

The final two add `DAMOS_SPLIT`. Through `target_order`, users can choose the order to which a large folio should be split; the implementation calls `split_folio_to_order()`. The early RFC called the action `DAMOS_MTHP_SPLIT`. At SJ’s suggestion, v2 renamed it to the shorter and more consistent `DAMOS_SPLIT`. The `hot_threshold` field was also removed from the kernel interface, leaving hotness policy in user space.

## The kernel provides mechanisms; user space sets policy

![Slide 8 of the production deck: kernel mechanisms and user-space policy](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-8.png)

The design is straightforward: the kernel provides collapse, split, and address-filter mechanisms, while user space decides which addresses should be acted upon.

On a system with Arm SPE, user space records SPE data with `perf record`. `spe_hist` then measures heat within each THP. The physical addresses are translated back to process virtual addresses, and ranges containing sparse THPs are written to the DAMON sysfs address filter. When the filter matches in the kernel, DAMON runs `DAMOS_SPLIT` or `DAMOS_COLLAPSE`.

![Slide 9 of the production deck: Arm SPE versus DAMON AF](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-9.png)

Arm SPE samples physical addresses from the CPU pipeline. It does not depend on the page-table AF and is unaffected by TLB hits, while its granularity can reach the cache-line level. DAMON’s AF path works across architectures and remains lightweight, but it sees only PMD granularity for THPs. The two can complement each other: SPE supplies precise candidate ranges, while DAMON performs generic kernel operations.

One practical limitation is that the SPE PMU is generally a single-consumer resource. If `perf` owns SPE, another consumer cannot use it at the same time, and vice versa. For that reason, we stopped feeding SPE data into the kernel through a direct `debugfs` interface. Collection and policy now stay in user space, leaving room to replace SPE with AMD IBS, Intel PEBS, or another data source later.

## Test status

![Slide 10 of the production deck: validation through the smaps path](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-10.png)

On systems without SPE, user space can scan `/proc/<pid>/smaps`, enumerate THP-backed ranges, and select them through an address filter. We validated this path on an aarch64 VM and a physical Kunpeng 920 machine. In the tests, `sz_tried` covered the complete monitored range, while `sz_applied` accounted only for THPs selected by the address filter. That confirmed the action was bounded to the intended targets.

The SPE path was still being debugged at the time of the talk. A sparse-access test program reproduced and amplified the issue in the lab, but the complete Sangfor KVM plus Oracle DB production chain still required end-to-end validation. These results demonstrate the problem and the mechanism; they should not be read as conclusions for every production workload.

## Where the work goes next

![Slide 11 of the production deck: short-, medium-, and long-term directions](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-11.png)

In the short term, we will revise the collapse/split series based on mailing-list review and finish the user-space SPE path. In the medium term, we hope to work with Arm, Huawei, and other community developers on an SPE-to-DAMON feedback design, while exploring similar capabilities from Intel PEBS and AMD IBS. The longer-term goal is for DAMON to become a standard sensor for Linux memory management: the kernel mechanisms remain stable, while policy and hardware data sources can be composed for each deployment.

Many people helped with this work. We thank SJ for his direction and reviews as the DAMON maintainer; Kefeng Wang, Asier Gutierrez, and our collaborators at Arm and Huawei for the discussions; and Sangfor for the production use case. The Kernel Mastery community and Barry Song also gave us valuable advice on THP.

If you encounter a problem while using DAMON, please continue the discussion on the mailing lists or contact us directly. Thank you.

![Slide 12 of the production deck: three takeaways](../assets/SeongJae-Park-王炼-陈涛-damon-in-production/damon-production-slide-12.png)

## Mailing-list threads and further reading

The links below were used to verify the DAMON design, patch revisions, and roadmap discussed in the talk. The patches are still evolving on the mailing lists; the final upstream interface may differ.

### Official DAMON resources

- [Linux kernel documentation: DAMON](https://docs.kernel.org/mm/damon/)
- [Linux kernel documentation: DAMON design](https://docs.kernel.org/mm/damon/design.html)
- [DAMON project website and community](https://damonitor.github.io/posts/damon/)
- [DAMON mailing-list archive](https://lore.kernel.org/damon/)

### mTHP collapse/split and Arm SPE

- [Lian Wang: `[RFC PATCH 0/6] mm/damon: Add mTHP-aware collapse/split with ARM SPE feedback`](https://www.spinics.net/lists/kernel/msg6273679.html)
- [Lian Wang: `[PATCH v2 0/5] mm/damon: add mTHP collapse and split actions`](https://lkml.iu.edu/2607.0/00984.html)
- [SJ’s response on the v2 patch-status transition](https://lkml.iu.edu/2607.0/01386.html)
- [User-space SPE experiments: `lianux-mm/damon_spe`](https://github.com/lianux-mm/damon_spe)

### Data-attribute monitoring roadmap

- [SeongJae Park: DAMON data-attribute extension project discussion](https://lore.kernel.org/20260525225208.1179-1-sj@kernel.org/)
- [SeongJae Park: `[RFC PATCH v1.2 00/19] mm/damon: introduce data attributes only monitoring` (post-event update)](https://lore.kernel.org/20260709140600.90950-1-sj@kernel.org/)
