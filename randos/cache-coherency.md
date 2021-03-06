## SMP / Uniform Memory Access

SMP and Uniform Memory Access are basically synonyms. They mean
multiple compute units (either cores or processors), attached via a
shared bus to RAM. A major problem is that the shared bus limits
memory bandwidth.

Since memory latency is poor, it's typically for CPUs to have big
caches, that's another part of the question. How do we keep those
caches coherent? That is, if CPU1 writes a memory address, CPU2 should
see this write (eventually).

## Snooping

All CPUs listen to the bus for requests to memory. Every cacheline is
in one of the three states:

* Invalid
* Shared
* Exclusive/Dirty

On a CPU1 read miss, any CPU2 with an exclusive version needs to write
it out. CPU1 can see this version. Now they both have a "shared"
version.

On a write hit to a valid cacheline, CPU1 tells everyone to mark their
version as invalid. CPU1 now has an exclusive/dirty version. Further
writes to a dirty version do not need to be broadcast.

A write miss invalidates anyone else's copy. Now that CPU has it
exclusively. In this way, future writes won't have to broadcast and
just hit cache.

This has poor scalability, since we have to lock the bus to
communicate any of these situations.

## NUMA

This is *non uniform memory access*. The basic idea is that some
memory is closer to some processors. This means not everyone is using
a single shared bus, reducing contention. Of course, this only
improves things if different tasks are working mostly with a subset of
the data.

Another approach to this problem was adding multi-channel memory,
which gave some ability to load more than one thing from memory at a
time. But that doesn't really scale.

The idea is that, if you're sharing a single memory bus, you prolly
don't need more than one processor. That's how extreme the starvation
for data is.

Cache coherence becomes harder to maintain, now that not everyone
shares a bus. Effectively: you can't just do snooping
now. Theoretically you don't need to maintain cache consistency, but
programming without it is very complicated, so this is a required
feature. We call NUMA systems with cache consistency ccNUMA; all NUMA
systems you'll ever see are ccNUMA.

Basically, you can see NUMA as a tightly-coupled form of cluster
computing.

## Directory-Based Cache Coherence

In NUMA systems you typically use this cache coherency strategy. Each
CPU has its own directory. It keeps track of who has each
cacheline. All communication is done point-to-point.

Slower than snooping when there's enough bandwidth, but scales better.

I should find a better resource, I don't actually know 100% how this
works.

## Volatile, Memory Ordering

With NUMA, you're always going to have a cache consistent view of the
data. When memory isn't shared in two caches, then updates don't
require communication.

We do know that your program may not check the cache: it might use a
register. To make sure it hits the cache, you have to use the
`volatile` keyword. Basically, the language is made aware that the
variable could be shared across threads. It won't make assumptions
that it has sole ownership of this variable. It won't use registers,
and it won't re-order.

But the hardware still can! That's why you need memory barriers. Any
read/write operation from before the barrier is guaranteed to not be
moved across the barrier, and vice versa!

Java sort of wraps these up together: if you declare a variable
`volatile`, then it also ensures that there is a "happens before"
relationship between your operations to this variable: they will not
be reordered. Presumably it does this by doing a bunch of fencing.

## Associativity

Caches often have *associativity*. A *fully associative* means that
any cacheline can be replaced on a miss. Direct-mapped means exactly
one line could be replaced. `k`-way associativity means that any of
`k` lines might be replaced. Basically, you have a limited number of
cache line slots; if you can only place a cacheline in one slot, then
this is direct-mapped.

If you have associativity, you need to check more places for a
cacheline. But this way you don't necessarily have to replace a hot
cacheline just because of a collision. Remember that collisions can be
quite common, when the cache is small, and the hot set is of the same
order of magnitude as the cache size.

Basically, associativity will increase the hit rate, at the expense of
retrieval time.

## Cache Coherency and TAS vs TTAS

Say you want to spin, waiting for a value to be set. *Local spinning*
occurs when you have the value in cache, and you just keep checking
until it is ivalidated. In that case, you do not need to talk to the
bus, and you should not interfere with the other processors. This is
the ideal version of spinning.

When you do a TAS, it needs to talk to the bus. These failed requests
can saturate the bus, destroying performance. With TTAS, you do not
make bus requests that you *know* can't succeed.

They make a related suggestion: try to keep a contested lock and its
data on different cachelines, so that attempts to get the lock won't
invalidate the data (and changes to the data don't signal that the
lock might be free).

Source: Art of Multiprocessor Programming

## Memory Consistency

Processors try to keep all their hardware units busy, by doing
out-of-order execution and work in parallel.

Write requests to RAM are typically buffered. This hides latency,
utilizes bandwidth better through batching, and eliminates some
requests due to *absorbtion* (when several writes can be
consolidated).

Consider a mutex algorithm *without* hardware support. It's very
important that everyone have a consistency view of memory. We know
that without cache coherency, we could write a variable on one thread,
and then do a read in a second thread, not seeing the write.

Cache coherency solves the problem introduced by caches. But *there is
still a problem*, introduced by the write buffer (and presumably from
OOE). This is why we have memory fences; someone who issues a fence is
guaranteed to see all other writes from the other processors before
that fence was issued. Fences can take 100s of cycles.

They contrast CAS (which has the ABA problem: update from A to B, then
from B to A, and now they can't see the change). Another possibility
is `load-linked` and `store-conditional` (from PowerPC), where the
`SC` works only if the variable hasn't changed since the `LL`. The
only problem here is you have to be careful what you do between the
`LL` and the `SC`...

Random note: parallelism on a single CPU with multiple cores can be
advantageous because of the shared L2 cache (L1 typically not shared).

Source: Art of Multiprocessor Programming
