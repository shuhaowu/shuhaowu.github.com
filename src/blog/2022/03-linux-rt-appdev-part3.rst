.. meta::
   :title: Real-time programming with Linux, part 3: Sources of latency in real-time applications
   :authors: Shuhao Wu
   :created_at: 2022-04-29 23:30

`Part 1 </blog/2022/01-linux-rt-appdev-part1.html>`_ - `Part 2 </blog/2022/02-linux-rt-appdev-part2.html>`_ - Part 3

A real-time (RT) system consists of three parts: the hardware, the operating
system, and the application. All three parts must meet the latency deadline for
the whole system to correctly function. I have covered how to deal with the
hardware and the operating system (OS) in the `previous post
</blog/2022/02-linux-rt-appdev-part2.html>`__. The natural next question is:
how do we ensure the application behaves correctly under RT constraints? The
answer to this is surprising complicated and requires a few posts to cover the
basics. In this post, I will discuss the sources of latency coming from the
application. From these, best practices for writing an RT application can be
established.

In the last post, I did promise to write some code. This unfortunately must be
delayed for one last time, as this post is too long as is.

RT vs non-RT sections of the code
=================================

Before I begin, it is important to clarify that an RT application has two
sections in its code: a non-RT section and a RT section. Code in the RT section
must execute within the required deadline while code executing in the non-RT
section do not have a (strict) deadline. The program flow of two typical RT
applications are shown in *Figure 1* below. When the program is initializing or
shutting down, it is usually in the non-RT section. During initialization,
multiple threads are typically created. Usually there are at least two threads:
an RT one and a non-RT one. In the robot application example (left), there are
two threads: a non-RT data logger thread and an RT `PID controller
<https://en.wikipedia.org/wiki/PID_controller>`__ thread. The initialization
and shutdown sections of the RT PID controller thread are also not RT. Only the
actual PID controller loop is. A similar architecture exists for the audio
production program (right), and basically every other multi-threaded RT
application. Most of the time, we only need to worry about sources of latency
in the RT sections of the program. Thus, the practices discussed in this post
are only necessary in the RT sections and usually unnecessary in the non-RT
sections.

.. figure:: /static/imgs/blog/2022/03-rt-critical-vs-non-critical.svg

   *Figure 1*: Example of program flow in the robotics and audio contexts.
   Arrow shows the flow of the program. Red blocks indicate RT sections. White
   blocks indicate non-RT sections.

Avoid non-constant worst-case execution time
============================================

It turns out, there is a single rule that you need to follow to be successful
at real-time programming: **do not call code in the RT section if you don't
know its worst-case execution time**. Basically, this is saying that you
cannot use non-constant time algorithms, unless the input size is low enough
that the actual execution time of the algorithm remains acceptably bounded.
Unfortunately, most algorithmic implementations favour throughput over latency,
which usually means they are tailored to the average-case execution time.
Worst-case execution time could be orders of magnitude worse than the average
case and such occasions could occur infrequently (e.g.
``std::vector.push_back``). The worst-case execution times also tend to be
under-documented (if at all). If the RT system cannot tolerate any deadline
misses, specialized libraries and algorithms are usually needed.  However, most
RT applications do not such hard guarantees. In these situations, "normal"
libraries such as the C++ standard library can be safely used if the worst-case
execution time can be acceptably determined. There is a `talk in 2021 by Timur
Doumler <https://www.youtube.com/watch?v=Tof5pRedskI>`__ that presents ways to
avoid function calls with unbounded latency in the C++ standard library that I
highly recommend for anyone working on RT C++ applications.

However, the implementation of algorithms is not the only type of latency. The
performance of the implementation must also be fast enough. Performance is a
vast topic that is outside the scope of the present post. For real-time, one
should be aware of problems such as cold/invalidated cache, excessive context
switching, memory latency issues, and much more. Since this is not covered
here, readers interested in this kind of problems can refer to at least the CPU
and memory chapters of `System Performance, 2nd Edition by Brendan Gregg
<https://www.brendangregg.com/systems-performance-2nd-edition-book.html>`__.

An additional source of latency exist in the interaction between the
application and the OS. Such latency cannot be inferred from reading the
application code by itself. As noted in part 1, programming on Linux and other
general-purpose operating systems is almost like magic. Although the operating
system is juggling between thousands of concurrent tasks with a limited amount
of CPU and memory, the application can be written as if it is the only process
consuming CPU and memory. The way this is accomplished on Linux (and other
general-purpose OSes) is optimized for throughput instead of the worst-case
latency.  Since RT applications need to ensure that worst-case performance
remain acceptable, we must understand the OS-level magic to ensure the
application won't miss its deadline. The two main kernel subsystems that we
need to understand and cooperate with are: (1) the virtual memory system and
(2) the CPU scheduler.

Virtual memory: avoid page faults and use ``mlockall``
======================================================

When an application accesses memory on a general-purpose OS like Linux, it is
almost certainly interacting with `virtual memory
<https://en.wikipedia.org/wiki/Virtual_memory>`__ instead of physical RAM.
This is an abstraction that provides each process with its own private memory
address space such that writes cannot accidentally overwrite values in memory
addresses assigned to other processes. When allocating virtual memory using
``malloc`` in Linux, memory is not allocated until the addresses are accessed
for the first time during a read or a write, due to a feature known as `demand
paging <https://en.wikipedia.org/wiki/Demand_paging>`__. Upon access, a `page
fault <https://en.wikipedia.org/wiki/Page_fault>`__ is generated, which causes
the kernel to synchronously perform the actual memory allocation, which results
in a delay. Kernel memory allocation can take an unbounded amount of time\
[#f1]_ and the RT application may miss its deadlines. Thus, these page faults
must be avoided. One way to avoid such page faults is to avoid dynamically
allocating memory during the RT sections of the code\ [#f2]_.

However, simply not allocating memory in RT will not get rid of all page
faults. In Linux (and other general-purpose OSes), the sum of memory allocated
(via ``malloc``) by all applications running on the system can be greater than
the available physical RAM. This feature is known as `memory overcommitment
<https://en.wikipedia.org/wiki/Memory_overcommitment>`__. When physical RAM is
exhausted, the kernel has two options: (1) kill the process using the
out-of-memory (OOM) killer or (2) move blocks of memory from physical RAM into
a much-slower secondary storage, like the disk. Obviously, the first case is
catastrophic for RT, as a terminated process cannot satisfy latency
requirements. The second case is much more subtle: the swapping of memory from
physical RAM to disk may occur while the application is sleeping if the system
is facing memory exhaustion. When the swapped memory addresses are eventually
accessed again by the application, a page fault is generated, which causes the
OS to load the values from the disk. Since disk latency can be on the order of
milliseconds and such page faults are synchronously processed, it can introduce
a delay to the RT sections of the application. This is completely
unpredictable by the application and catastrophic for real-time. While running
out of memory is something that should be generally avoided for RT
applications, the OS kernel may choose to swap physical RAM into the disk even
in the absence of memory pressure. To avoid all of this, the application can
instruct the OS to lock all its memory addresses in physical RAM, using the
``mlockall(MCL_CURRENT | MCL_FUTURE)`` function call.

In the literature, there are a number of references that claim the need to
"pre-fault" the memory addresses assigned to the RT process. This usually means
writing a value to all memory addresses allocated to the application during
initialization. The reason to do this is to counteract the effects of demand
paging and ensure that all memory addresses are locked in physical RAM.
However, the |man page|_ for ``mlockall`` shows this to be unnecessary. The
following snippet comes from the man page of ``mlockall``:

    MCL_CURRENT Lock all pages which are currently mapped into the address
    space of the process.

    MCL_FUTURE Lock  all  pages  which will become mapped into the address
    space of the process in the future.  These could be, for instance, new
    pages required  by  a  growing heap and stack as well as new memory-mapped
    files or shared memory regions.

If these two flags are passed to ``mlockall``, all existing memory addresses
(pages) will be mapped into physical RAM after the call completes; and all
future memory allocations will immediately lock the memory addresses into
physical RAM. Combining these two options effectively turns off demand paging
for the calling application. Indeed, `experiments
<https://github.com/shuhaowu/rt-demo/blob/7116d52/docs/prefault-experiments/>`_
show that page faults are immediately generated upon memory allocation for both
the heap and the stack. Thus, contrary to the numerous resources available
online, pre-faulting is unnecessary\ [#f3]_.

In summary:

#. To avoid page faults generated during memory allocation, avoid dynamic
   memory allocation in the RT sections entirely. Memory should be allocated
   before the application enters the RT section, or in the non-RT thread
   (something that will be discussed more next time). There are other
   programming techniques, like the `object pool pattern
   <https://en.wikipedia.org/wiki/Object_pool_pattern>`__, that can be used
   instead of dynamic memory allocation.
#. To avoid memory swapping, lock down all the virtual memory addresses needed
   by the RT sections to physical RAM with the ``mlockall(MCL_CURRENT |
   MCL_FUTURE)`` function call. This prevents the operating system from swapping
   the RT application's memory into secondary storage at the OS's discretion.
#. ``mlockall(MCL_CURRENT | MCL_FUTURE)`` also turns off demand paging, at
   least for Linux. There is thus no need to pre-fault the stack, despite
   numerous literature to the contrary.

As a note, code for everything presented here and in the subsequent sections
will be presented as a part of the small RT app framework in the next post.

.. [#f1] For example, the OS may need to free some RAM elsewhere (by possibly
         moving it to the disk) to be able to satisfy your application's memory
         allocation request, which make take a while.
.. [#f2] Technically, it is possible to perform dynamic memory allocation via
   ``malloc`` if you already reserved a block of memory from the OS. In
   practise, most ``malloc`` implementations are not constant time and may
   occasionally take a long time even if free memory is already reserved to the
   application. While it is certainly possible to get a constant-time
   allocator, it's likely better to keep it simple and not perform any dynamic
   memory allocations, thus avoiding this problem alltogether.
.. [#f3] It is also not clear to me if prefaulting works at all. A quick
   reading at some of the code that prefaults the stack suggests that it may be
   optimized out by the compiler, as it has no side effects.
.. |man page| replace:: ``man`` page
.. _man page: https://man7.org/linux/man-pages/man2/mlock.2.html

CPU scheduler: Avoid priority inversion
=======================================

By default, threads created on Linux are not scheduled using a RT scheduler.
The behavior of the default Linux scheduler is quite complex and is not
suitable for RT. Thus, threads that require RT behavior must request the RT
scheduler through the ``pthreads`` API. For brevity, I am not going to present
the code that does this now, as it will be presented in the next post\
[#fpthreads]_. Instead, I want to focus on a much more subtle problem that can
cause unbounded latency involving the interaction between the CPU scheduler and
the application's mutexes.  This bug is famous for `affecting the Mars
Pathfinder Rover
<http://www.cs.cornell.edu/courses/cs614/1999sp/papers/pathfinder.html>`__
despite the fact that the application is deployed on a hard RTOS (VxWorks).

Non-trivial RT applications usually require both RT and non-RT threads that
communicate with each other. Multi-threaded communication require some form of
synchronization to avoid data races\ [#f4]_. In non-RT programming, one simple
solution to this problem is to protect access to the shared variables with a
mutex. In C++, this is usually coded with ``std::mutex`` as defined by the C++
standard library. When such a program runs, access to the shared variable may be
serialized in the following sequence:

#. Initially, the shared variable have the value of *v1*.
#. Thread 1 acquires lock on the mutex and begins reading/write to the shared
   variable with value *v2*.
#. Thread 2 attempts to acquire the lock on the same mutex and is blocked as it
   is held by Thread 1.
#. Thread 1 finishes writing to the variable and releases the lock.
#. Thread 2 is unblocked, reads the shared variable has a value of *v2*.

This is perfectly acceptable for an application without a bounded latency
requirement (i.e. all non-RT apps) as the average latency is likely to be
quite low. However, the worst-case latency is unbounded on Linux (and other
"general-purpose" operating systems). Thus, mutexes are unacceptable for RT.
The root cause for this is the **priority inversion** problem as demonstrated
in *Figure 2* below:

.. figure:: /static/imgs/blog/2022/03-rt-prio-inversion.svg

   *Figure 2*: Diagram illustrating priority inversion with (top) and without
   (bottom) mutex with priority inheritance

The figure depicts three processes with three different priority levels sharing
a single CPU. The colour of the rectangles shows the original priority levels
of the threads. The colour of the lock status line shows the thread that
currently owns the mutex. Each process executes for a duration, which is
denoted by the width of the rectangles. Finally, the vertical axis denotes the
current priority level of the code executing on the CPU. An application is
shown in the top plot which uses a regular mutex (e.g.  ``std::mutex``). The
low-priority thread of this application, shown in green, acquires a lock via
the mutex. Then, the high-priority thread, shown in red, preempts the
low-priority thread (at A) as it is scheduled to wake up. The high-priority
thread attempts to acquire a lock on the same mutex, which blocks (at B). At
this point, the OS scheduler noticed that the high-priority thread is blocked
and thus puts it back to sleep. The scheduler then reschedules the low-priority
thread, allowing it to finish with its work and release the lock.  As this work
occurs, an unrelated thread (or even another process) with a slightly higher
priority level, shown in orange, preempts the low-priority thread (at C) until
it is put back to sleep, which can take an unbounded amount of time. Throughout
this time, the high-priority thread cannot resume as it remains blocked by the
low-priority thread. In effect, the medium-priority thread is able to block the
execution of the high-priority thread indefinitely due to the usage of the
regular mutex. Such unbounded latency is always unacceptable for RT.

One way to solve this problem is via mutexes with priority inheritance. The
bottom plot of *Figure 2* demonstrates this approach. As with the original case,
the low-priority thread acquires a lock. The high-priority thread preempts it
(at A) and tries to lock the same mutex (at B). This blocks, prompting the OS
to put the high-priority thread back to sleep. Noticing that the high-priority
thread is blocked on the mutex currently being held by the low-priority thread,
the OS switches to the low-priority thread with a temporarily boosted priority
level equaling that of the high-priority thread. In effect, the low-priority
thread *inherited* the priority level of the high priority thread, which
forbids the OS from interrupting its execution by the medium-priority thread.
Once the originally-low-priority thread releases the lock, its priority level
is reverted to the original value and the high-priority thread can continue
with its execution (at C). Thus, the overall latency remains bounded as long as
the code in the critical section (i.e. the duration when it held the lock) of
the low-priority thread is bounded.

There are several drawbacks to this approach. Notably, code within the critical
sections protected by mutexes on the low-priority thread might occasionally run
with RT priority. Thus, such code must be treated as if they are RT code. This
requires the code within the mutex's critical sections to follow the best
practices outlined in this article. In many situations, this is not desirable.
Coupled with other mutex-related problems\ [#f6]_, lock-free (or more strongly,
wait-free) programming can potentially be a more appealing way to get around
the need for a mutex. However, this topic is way too big for me to cover now,
so I will defer it to a future post. For the time being, you can look into the
``boost::lockfree`` package and ``atomic`` variables.

In summary:

#. If mutexes are required for RT, always enable priority inheritance. This is
   not possible with ``std:mutex`` and requires the use of pthread mutex
   directly\ [#fmut]_.
#. Investigate into lock-free (wait-free) programming techniques to share data
   between threads. This is something I'll explore in a future post.

.. [#fpthreads] There are also examples of using this API in `this wiki page
   <https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/application_base>`__.
.. [#f4] Data races occur when two or more threads attempt to access the same
   memory location, where at least one thread performs a write. For interested
   readers, I recommend `this series <https://research.swtch.com/mm>`__ on
   memory models across languages and hardware, which goes into these ideas in
   more detail.
.. [#f6] There are some `debates <https://lwn.net/Articles/178253/>`__ over
   whether using priority inheritance is even a good idea, especially since
   mutexes with priority inheritance can be quite difficult to implement
   correctly. Additionally, mutexes can introduce a number of issues, such as
   deadlocks and performance issues involving thread preemption even in the
   absence of priority inversion. Notably, audio production software appears to
   employ lock-free programming heavily, as priority inheritance is not
   availble on Windows until `very recently
   <https://docs.microsoft.com/en-us/windows/iot/iot-enterprise/soft-real-time/soft-real-time>`__.
.. [#fmut] I will try to cover this in a future post. For now, interested
   readers can take a look at my ``rt::mutex`` implementation `here
   <https://github.com/shuhaowu/rt-demo/blob/master/libs/rt/include/rt/mutex.h>`__.

Don't trust the OS? Avoid system calls
======================================

When an application runs, it usually performs a lot of `system calls
<https://en.wikipedia.org/wiki/System_call>`__ to instruct the OS kernel to do
some work on its behalf, usually synchronously during the application's
execution. We have already seen two of them: ``malloc`` (via ``sbrk`` and
``mmap``) and ``mlockall``.  Others may include writing to files and
interacting with USB devices. Most of these system calls are hidden behind
libraries commonly used by applications. Since Linux was not originally
designed to be a RTOS, there are generally no guarantees that a particular
system call won't cause page faults or priority inversion problems internally.
It might even block the process (such as calls like ``accept``) which causes
the process to be scheduled out of the CPU until the call is unblocked.
Further, system calls may result in a full `context switch
<https://en.wikipedia.org/wiki/Context_switch>`__, which is associated with a
small CPU overhead that may be problematic in some situations.

There are a few solutions to these problems:

#. Use an OS where all system calls are documented with worst-case execution
   time.
#. Audit the kernel source code to determine worst-case execution time and
   ensure the calls used do not block. Alternatively, obtains some sort of
   "soft" guarantee from someone else that has audited the code\
   [#fauditkernel]_.
#. Don't trust the kernel, be defensive, and avoid system calls unless
   absolutely necessary (such as for IO, and getting the current time in
   high-resolution).

If you want to write an RT application for Linux, the number of distinct system
calls used should be kept to a minimum, so it is feasible to audit them and
make sure they cannot cause problems. This might feel somewhat shaky, but RT
applications in `robotics <https://github.com/ArduPilot/ardupilot>`__ and `audio
<https://github.com/jackaudio/jack2>`__ domains have been developed for Linux
with (presumably) acceptable performance.

.. [#fauditkernel] I'm not aware of a list of safe and unsafe system calls for
   Linux. Presumably commercial hard-RTOSes have such a list.

Summary
=======

In the third part of this series, we determined a list of potential sources of
latency and came up with the following "best practices":

* Avoid code with non-constant worst-case execution time

  * Avoid non constant-time algorithms
  * Write fast code

    * Profile and optimize the code as necessary
    * Avoid excessive context switches
    * Avoid CPU cache invalidation if the code relies on cache for speed

* Avoid page faults due to either demand paging or swapping by calling
  ``mlockall(MCL_CURRENT | MCL_FUTURE)`` and reserving all memory needed before
  the RT code sections start.

  * There is no need to prefault the stack nor the heap after allocation,
    contrary to numerous online literature.

* Avoid standard mutexes such as ``std::mutex`` by either using
  priority-inheriting mutexes or lock-free programming instead.
* Avoid system calls where possible, in case the kernel suffers from any of the
  three issues mentioned above

In the next post, we will see these in action, with an RT application framework
as well as an example application.

Appendix: References
====================

These are some of the more relevant materials I've reviewed as I wrote this post:

* `Challenges Using Linux as a Real-Time Operating System - Michael Madden <https://ntrs.nasa.gov/citations/20200002390>`__
* `Real-time programming with the C++ standard library - Timur Doumler <https://www.youtube.com/watch?v=Tof5pRedskI>`__
* `System Performance, 2nd Edition - Brendan Gregg <https://www.brendangregg.com/systems-performance-2nd-edition-book.html>`__
* `Code demonstrating that prefaulting is not needed <https://github.com/shuhaowu/rt-demo/blob/7116d52/docs/prefault-experiments/>`__
* `Make multiprocessor computer correctly execute multiprocess programs - Leslie Lamport <https://www.microsoft.com/en-us/research/publication/make-multiprocessor-computer-correctly-executes-multiprocess-programs/>`__
* `Series on memory model - Russ Cox <https://research.swtch.com/mm>`__
