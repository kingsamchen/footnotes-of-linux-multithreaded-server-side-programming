# fsync() on a different thread: apparently a useless trick

fsync() is the kind of system call you love and hate at the same time. In many applications it's nice to know that kernel buffers are flushed to disk (even if this alone does not necessarily guarantees data is actually written to the disk, as the disk itself can have caching layers), but unfortunately fsync() tends to be monkey assess slow.

As I like numbers, slow is, for instance, 55 milliseconds against a small file with not so much writes, while the disk is idle. Slow means a few seconds when the disk is busy and there is some serious amount of data to flush.

With some application this is not a problem. For instance when you save your edited file in vim the worst that can happen is some delay before the editor will quit. But there are applications where both speed and persistence guarantees are required, especially when we talk about databases.

Like in my specific case: Redis supports a persistence mode called Append Only File, where every change to the dataset is written on disk before reporting a success status code to the client performing the operation. In this kind of application it is desirable to fsync() in order to make sure the data is actually written on disk, in the event of a system crash or alike.

Since fsyncing is slow, Redis allows the user to select among three different fsync policies:
- fsync never: just let the kernel doing it when needed. In Linux this usually means that data will be flushed on disk at max in 30 seconds. But you can change the kernel settings to change this defaults if needed.
- fsync everysec: call fsync every second.
- fsync always: call fsync after every write operation against the Append Only File, and before reporting a success status code to the client.

The first option is the faster, the second is almost as fast as the first but much safer, the third is so slow to be basically impossible to use, at the point I'm thinking about dropping it.

The "fsync everysec" policy is a very good compromise and works well in practice if the disk is not too much busy serving other processes, but since in this mode we just need to sync every second without our sync being blocking from the point of view of reporting the successful status code to the client, an obvious thing to do is moving the fsync call into another thread. Doing things in this way, in theory, when from time to time an fsync will take too much as the disk is busy, no one will notice and the latency from the point of view of the client talking with the Redis server will be good as usually.

Sounds cool right? But I started to have the feeling that this would be totally useless, as the write(2) call would block anyway if there was a slow fsync() going on against the same file, so I wrote the following test program:

```c
/* Test if write(2) blocks when an fsync() in the same file is in progress
 * in another thread.
 *
 * Compile with: gcc fsync.c -Wall -W -O2 -lpthread -o fsynctest
 *
 * For feedbacks drop an email to antirez at gmail
 */

#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <string.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/time.h>
#include <stdlib.h>

static long long microseconds(void) {
    struct timeval tv;
    long long mst;

    gettimeofday(&tv, NULL);
    mst = ((long long)tv.tv_sec)*1000000;
    mst += tv.tv_usec;
    return mst;
}

void *IOThreadEntryPoint(void *arg) {
    int fd, retval;
    long long start;
    arg = arg;

    while(1) {
        start = microseconds();
        fd = open("/tmp/foo.txt",O_RDONLY);
        retval = fsync(fd);
        close(fd);
        printf("Sync in %lld microseconds (%d)\n", microseconds()-start,retval);
        usleep(1000000);
    }
    return NULL;
}

int main(void) {
    pthread_t thread;
    int fd = open("/tmp/foo.txt",O_WRONLY|O_CREAT,0644);
    long long start;

    pthread_create(&thread,NULL,IOThreadEntryPoint,NULL);
    while(1) {
        start = microseconds();
        if (write(fd,"x",1) == -1) {
            perror("write");
            exit(1);
        }
        printf("Write in %lld microseconds\n", microseconds()-start);
        usleep(100000);
    }
    close(fd);
    exit(0);
}
```

The program is pretty simple. It starts one thread doing an fsync() call every second, while the other (main) thread does a write 10 times per second. Both syscalls are benchmarked in order to check if when a slow fsync() is in progress the write() will also block for the same time.

The output speaks for itself:

```
...
Write in 11 microseconds
Write in 12 microseconds
Write in 12 microseconds
Write in 12 microseconds
Sync in 40523 microseconds (0)
Write in 30596 microseconds
Write in 11 microseconds
Write in 11 microseconds
Write in 11 microseconds
Write in 11 microseconds
...
```

Unfortunately my suspicious is confirmed. This is really counter intuitive since after all we are talking about flushing buffers on disk. When this operation is started the kernel could allocate new buffers that will be used by new write(2) calls, so my guess is, this is a Linux limitation, not something that must be this way. (Note: you may need to run the program for some minute before the two threads are in sync enough to show that behavior).

## What about fdatasync()?

Since this behavior seemed so strange I started wondering if fsync() actually blocks all the other writes until the buffers are not flushed on disk because it is required to also flush metadata. So I tried the same thing with fdatasync(), that is much faster, unfortunately it just takes some more time to see the same behavior because fdatasync() calls are usually much faster, but from time to time I was able to see this happening again:

```
Write in 13 microseconds
Write in 13 microseconds
Write in 14 microseconds
Write in 14 microseconds
Write in 12 microseconds
Write in 13 microseconds
Write in 12 microseconds
Sync in 48649 microseconds (0)
Write in 47213 microseconds
Write in 13 microseconds
Write in 10 microseconds
Write in 13 microseconds
```

## Conclusions

If you have a Linux write intensive application and are thinking about calling fsync() in another thread in order to avoid blocking, don't do it, it's completely useless with the current kernel implementation.

If you are a kernel hacker and know why Linux is behaving in an apparently lame way about this, please make me know.

Edit: Same test of the above, but instead of the fsync()ing thread the file is opened with O_SYNC:

```
...
Write in 219 microseconds
Write in 253 microseconds
Write in 264 microseconds
Write in 271 microseconds
Write in 246 microseconds
Write in 259 microseconds
Write in 251 microseconds
Write in 232 microseconds
Write in 235 microseconds
Write in 251 microseconds
...
```

Very interesting. Every write takes more than 20 times more time, but it's much faster blocking 2500 us every 10 writes compared to the big stop-the-world-for-40000-us every 10 writes with fsync(). So we have a clear winner here for "fsync always". Still no better solution of the current one for "fsync everysec" but this is working pretty well already.

---
1. 见书 P171
2. 原文链接：http://oldblog.antirez.com/post/fsync-different-thread-useless.html
