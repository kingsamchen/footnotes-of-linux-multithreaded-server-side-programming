# The mb() and programming smarts

Roland [writes](http://digitalvampire.org/blog/index.php/2007/06/05/17/):

> However, I still think that using a spinlock around an assignment that’s atomic anyway is at best pretty silly. If you just need a memory barrier, then put an explicit `mb()` (or `wmb()` or `rmb()`) there, along with a fat comment about why you need to mess with memory barriers anyway.

**NOOOOOOOOOOO!**

In theory, it's possible to program in Linux kernel by using nothing but `mb()`. In practice, expirience teaches us that every use of `mb()` in drivers is a bug. I'm not kidding here. For some reason, even competent and experienced hackers cannot get it right.

I have no idea why it is so, and I don't want to go into the ESR's territory of speculation, "curse of the gifted", and such. I just want anyone who writes drivers to burn it into their minds that memory barriers are off limits (there are some very specific exceptions for mailboxes on 10GE cards and things like that, but nonetheless).

Keeping people from writing code which blows up when Red Hat ships it and customers run it on bigger boxes than the developer's laptop is an old hobby of mine. For example, my article about the dangers of read-write locks and spin_lock_bh was written back in 2002, when I blogged at Advogato [link](http://www.advogato.org/person/Zaitcev/diary.html?start=153). But this barrier business is worse, because it affects more than just novices.

---

1. 见书 P31
2. 原文链接：https://zaitcev.livejournal.com/144041.html
