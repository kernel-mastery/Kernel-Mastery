# Beyond Code: Communication, Consensus, and Getting Patches Merged Upstream | Conference Talk Transcript

- Speaker: Zi Yan (晏子)
- Date: July 5, 2026
- Chinese version: [中文逐字稿](../zh/晏子-linux-upstream-communication-consensus-and-patch-merging.md)

## Opening

**Host:** Our next speaker is Zi Yan.

**Zi Yan:** I will not focus too much on technical details today. Other speakers will cover technical topics. I want to talk about what happens after you send a patch: how to communicate with people in the community, work through feedback, and get your patch merged.

Many people run into this. They send a patch and hear nothing back. Or the feedback is quite strong, and they do not know how to respond. Then there is the people side of things as well. I would like to share a few lessons from my years in the community.

I work at NVIDIA Research in the United States. During my PhD, I worked on a THP migration patch series related to my research. I built on work that was already there. The series went through dozens of versions and more than ten rounds of discussion and changes before it finally went into the kernel. That was exciting, and it made me interested in working with the community. Since then, I have stayed involved in Linux memory management: writing patches, joining discussions, and reviewing other people's work. Along the way, I have learned a few things about communication.

## Sending a patch is only the beginning

After spending a lot of time on a patch and sending it to the mailing list, many people feel that their job is done. In fact, sending the patch is only the beginning.

People send patches upstream for different reasons. Sometimes they want to stop rebasing local changes every time upstream releases a new version. Sometimes they believe their work can help the community. Some also want to show what they can do. Whatever the reason, a patch has a long life.

After it is sent, there will be discussion, questions, explanations, and changes. Maintainers will give feedback. The author can explain their choices or offer a different view. If things go well, the patch will go through several rounds, land in a subsystem maintainer's tree, and then wait for a merge window before it reaches mainline.

The work does not end once the patch is in mainline. A patch may not be fully right from day one. Problems can show up later, and it still needs care. I worked on THP migration in 2017, and even in recent years there have still been things to fix.

So contributors need to stay in the discussion, maintain their patches, and keep an eye on them in mainline. Fixing problems and keeping the code stable over time is also how you build a good relationship with the community.

## Building trust in the community

The community needs to know that there is someone behind a new patch who plans to stay involved and take responsibility for the code. If a patch is merged and its author never appears again, it can become a one-off contribution. Other developers will not have much chance to get to know that contributor.

If you can, make time to review other people's patches as well as your own. You will learn what problems others are trying to solve. You may find work that interests you and that you can join. You can also share your own problems and let the discussion grow from there.

I also had to spend time becoming a familiar face in the Linux memory-management community. At first, people did not know me. They asked why my design took a certain shape and whether it would work. That is normal. The community does not yet know whether you will stay, or whether your patch will be left without an owner after it is sent.

Long-term, steady participation helps people get to know you. It gives people a reason to keep talking and working with you.

## Looking at a patch from the maintainer's side

As a contributor, you often want to solve one specific problem, say problem A. A maintainer has a wider set of questions: Is problem A worth solving? Is there a better way? Can the solution cover more cases?

Maintainers may worry that a solution for one narrow case will add a special case and make the code harder to maintain. When they challenge a patch or suggest another idea, they are often looking beyond the details of that one implementation. They are trying to find a deeper solution that will last.

The same is true for performance work. A patch may improve one workload on Android, but maintainers will also ask whether it works for desktops and servers. Does the gain hold in a wider range of cases?

Even a large change that has been tested and deployed locally may need major changes. A local setup may solve one problem but still conflict with the kernel's wider design. A patch can be on the right track and still need work in other areas.

## Letting a local solution grow into community consensus

I would like to use my work on `CONFIG_READ_ONLY_THP_FOR_FS` in the last development cycle as an example. It was a Kconfig option that allowed `khugepaged` to collapse read-only file-backed pages into THPs. Matthew Wilcox first raised an idea. At first, I took it to mean that this mechanism should simply be removed. After several rounds of discussion, I understood the real goal: remove the special path that only served filesystems without large-folio support, while making the capability available to filesystems that meet the right conditions.

More specifically, the filesystem page cache must support folios of at least `PMD_ORDER`. With that in place, the patch series makes file-backed THP creation independent of `CONFIG_READ_ONLY_THP_FOR_FS`. Later revisions also extended the work to writable files. It covers three creation paths: page faults, `MADV_COLLAPSE`, and `khugepaged`.

This is how a patch changes through discussion. First, make sure you understand the problem and the goal. Then adjust the implementation in the direction maintainers are asking for. Mailing-list discussion runs through the whole process. It helps the community define both the boundary of the problem and the shape of the solution.

## Keep patch series manageable, and respect review time

Another common case is a very large patch series, perhaps with ten or twenty patches. The author may have tested and deployed it as a whole and may not want to make big changes. But that makes review much harder for maintainers.

Maintainers often ask contributors to split the work into smaller pieces that can be understood and reviewed on their own. A series with fewer than ten patches is usually easier to handle.

Maintainers receive hundreds or even thousands of emails from around the world every day. To a contributor, one patch may represent a great deal of work. To a maintainer, it is one item in a large queue. Be patient and explain why the problem needs to be solved, how serious it is, and what other options you considered. That helps maintainers understand its priority and keeps the discussion focused on the real problem.

## Keep going after rejection

I have had many patches sent back too. During my PhD, I submitted 1 GiB THP support early on. From around 2019, it kept coming back for changes and rewrites. Later, a developer at Meta picked up the direction again, but it still has not been merged because the discussion found missing pieces.

If you believe a patch matters to the kernel and to your own use case, do not give up just because it is sent back more than once. Fill in what is missing, keep talking, and keep improving it. It may reach mainline later in a more mature form.

## Questions and answers

### How can newcomers avoid mistakes in the submission process?

**Question:** New contributors may send the same email more than once because of a network problem, or create mailing-list noise because they do not know the process. What would you suggest to someone who wants to make a large architectural change but is new to upstream work?

**Zi Yan:** If you sent the same email a few times by mistake, write a short email to explain and apologize. That usually solves it.

The bigger concern is when a person who has never appeared in the community sends a very large patch series right away. The community does not know that contributor yet, and cannot quickly decide whether to trust the change. For this kind of work, it is better to start with an RFC or a discussion post.

Explain the problem you want to solve first, and invite the community to discuss it. Others who care about the problem can share their views. You can then build a more complete solution instead of sending a large patch series whose details may still need work. Even if the overall direction is right, this kind of discussion makes the later implementation more solid.

### How can English writing and AI work together?

**Question:** What would you suggest to Chinese developers who are not used to writing in English, especially for emails and patch descriptions?

**Zi Yan:** AI is now quite good at making a written English draft sound more natural. English is not my native language either, and I also use AI to check whether a sentence reads well.

Still, it is best to have a basic command of English. Chinese and English organize ideas differently. If you only translate Chinese directly, it can be hard to know whether the English says what you mean. One practical method is to write an English version first, ask AI to polish it, and then translate it back into Chinese if needed to check that the meaning has not shifted.

Technical writing in English can also follow a clear structure: start with the main point or problem, then go into the details, and return to the main point at the end. Begin each paragraph with a sentence that says what the paragraph is about. Even someone who only scans the first sentence of each paragraph can then understand what the patch is trying to do. This is one of the most important things I learned during my PhD.

## Host's closing remarks

**Host:** Thank you, Zi Yan, for the talk. Zi Yan has helped many newcomers in the community and guided people through their first submissions. If people run into problems with patch submission or community work, we can keep collecting them for discussion and continue the conversation by email.

## Mailing list and further reading

- [Zi Yan: `[PATCH v6 00/14] Remove CONFIG_READ_ONLY_THP_FOR_FS and enable file THP for writable files`](https://www.spinics.net/lists/kernel/msg6213546.html)
