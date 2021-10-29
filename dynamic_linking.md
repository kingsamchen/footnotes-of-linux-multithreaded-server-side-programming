# Dynamic Linking

“I tend to think the drawbacks of dynamic linking outweigh the advantages for many (most?) applications.” – John Carmack

All the purported benefits of dynamic linking (aka., ‘shared libraries’, which is a misnomer as static linking also shares libraries) are myths while it creates great (and often ignored) problems.

Both performance and security are seriously harmed by dynamic linking, but the damage caused by the huge complexity created by dynamic linking is extensive in almost all areas (the term ‘dll hell’ is just one example of the many hells created in dynamic linking environments).

And versioning symbols only bring you to a deeper level of hell.

> From: Rob Pike <robpike@gmail.com>
> Subject: mmap and shared libraries
> Date: Wed, 5 Nov 2008 17:23:54 -0800
>
> When Sun reported on their first implementation of shared > libraries,
> the paper they presented (I think it was at Usenix) concluded that
> shared libraries made things bigger and slower, that they were a > net
> loss, and in fact that they didn't save much disk space either.  > The
> test case was Xlib, the benefit negative. But the customer expects > us
> to do them so we'll ship it.
>
> So yes, every major operating system implements them but that does > not
> mean they are a good idea.  Plan 9 was designed to ignore at least
> some of the received wisdom.
>
> -rob

---

> From: Geoff Collyer <geoff@collyer.net>
> To: 9fans
> Subject: Virtual memory & paging
> Date: Mon, 4 Feb 2002 02:38:16 -0800
>
> There isn't a copy of the entire C library in every binary.  There is
> a copy of each library routine called, directly or indirectly, by the
> program in question.
>
> Sharing of instructions is done at the granularity of process text
> segments, as in V6 or V7 Unix.  The text segment of a process that
> forks is shared between parent and child by page mapping.  Also,
> running (via exec) a program multiple times concurrently causes the
> (pure) text segment to be shared by page mapping across those
> processes.  So all copies of rc and on a machine should share a text
> segment.
>
> Given that degree of sharing, the low cost of RAM, and the increase in
> OS complexity, slowness and insecurity in the implementations of
> dynamic libraries that I've seen, I don't see a need for dynamic
> libraries.  (Remember that the real impetus for adding them to Unix
> was X11 and its big and badly-factored libraries, which most of us
> aren't blessed with.)  My terminal has 115 processes; all but 4 of
> them share their text segment with at least one other process, usually
> more.  74 of them are instances of rio, Mail, rc, acme, listen,
> plumber and samterm.  A CPU server has 141 processes; all but 2 share
> text.  80 of them are listen, another 21 are rc, exportfs, kfs, dns
> and consolefs.  A quick sampling suggests that Plan 9 programs are
> typically smaller than FreeBSD/386 programs even with shared
> libraries.  Here are some FreeBSD sizes:
>
> : unix; size /bin/cat /bin/ed /usr/bin/awk /usr/X11/bin/sam
>    text    data     bss     dec     hex filename
>   54188    4324    9760   68272   10ab0 /bin/cat
>  122835    8772   81920  213527   34217 /bin/ed
>  135761    4772   15756  156289   26281 /usr/bin/awk
>   52525    1412   53448  107385   1a379 /usr/X11/bin/sam
>
> Of those, awk and sam use shared libraries.  The corresponding Plan 9
> sizes are:
>
> ; cd /bin; size cat ed awk sam
> 15996t + 2208d + 944b = 19148   cat
> 45964t + 4212d + 41232b = 91408 ed
> 114731t + 35660d + 12040b = 162431  awk
> 86574t + 7800d + 66240b = 160614    sam
>
> and the Plan 9 programs cope with Unicode and UTF.

---

> <btdn> I never, for the life of me, understand why people like dynamic linking.
> <aiju> btdn: for the very same reason they believe in god

---

vadaszi:

> I request dlopen() to be added to the 'harmful' list, since it
> breaks the assumptions of static linking (e.g. you expect the
> binary to be portable across linux distributions, but by using
> dlopen() some weird assertion break the binary when using a
> differeng glibc version).  Happened to me when using the ghc
> haskell compiler, even statically linked binaries won't work
> across linux distributions or glibc versions.

---
1. 见书 P424
2. 原文链接 http://harmful.cat-v.org/software/dynamic-linking/
