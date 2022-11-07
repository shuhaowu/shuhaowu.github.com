.. meta::
   :title: Real-time programming with Linux, part 4: C++ application tutorial
   :authors: Shuhao Wu
   :created_at: 2022-05-23
   :has_code: true

`Real-time programming with Linux </blogseries.html#rt-linux-programming>`__: `Part 1 </blog/2022/01-linux-rt-appdev-part1.html>`__ - `Part 2 </blog/2022/02-linux-rt-appdev-part2.html>`__ - `Part 3 </blog/2022/03-linux-rt-appdev-part3.html>`__ - Part 4

As we have explored thoroughly in this series, building a real-time (RT) system
involves ensuring the hardware, operating system (OS), and the application all
have bounded latency. In the `last post
</blog/2022/03-linux-rt-appdev-part3.html>`__, I described a few sources of
unbounded latency caused by interactions between the OS and the application. In
this post, we will write the necessary boilerplate code to avoid these
sources of latency in C++ with these few steps:

#. Lock all memory pages into physical RAM.
#. Setup RT process scheduling for the real-time thread(s).
#. Run a loop at a predictable rate with low jitter.
#. Ensure data can be safely passed from one thread to another without data
   races.

Since this code is required for basically all Linux RT applications, I
refactored the code into a small RT application framework in my `cactus-rt
repository <https://github.com/cactusdynamics/cactus-rt>`__. All the examples in this
post are also shown in full in that repository, along with a number of
additional examples based on the refactored app framework.

Locking memory pages with ``mlockall``
======================================

As noted in the `previous post
</blog/2022/03-linux-rt-appdev-part3.html#virtual-memory-avoid-page-faults-and-use-mlockall>`__,
code in the RT section needs to avoid page faults to ensure that memory access
latency is not occasionally unbounded. This can be done by locking the
application's entire virtual memory space into physical RAM via the |mlockall|_
function call. Usually, this is done immediately upon application startup and
before the creation of any threads, since all threads in a process share the
same virtual memory space. The following code snippet shows how to do this:

.. code-block:: c++
   :number-lines:

   #include <cstring> // necessary for strerror
   #include <stdexcept>
   #include <sys/mman.h> // necessary for mlockall

   void LockMemory() {
     int ret = mlockall(MCL_CURRENT | MCL_FUTURE);
     if (ret) {
       throw std::runtime_error{std::strerror(errno)};
     }
   }

   int main() {
     LockMemory();

     // Start the RT thread... etc.
   }


.. |mlockall| replace:: ``mlockall(MCL_CURRENT | MCL_FUTURE)``
.. _mlockall: https://man7.org/linux/man-pages/man2/mlock.2.html

This code is straight-forward: line 6 shows the usage of ``mlockall``, along
with some error handling after.

Setting up real-time threads with pthreads
==========================================

By default, threads created on Linux are scheduled with a non-RT scheduler. The
non-RT scheduler is not optimized for latency and thus cannot generally be used
to satisfy RT constraints. To setup an RT thread, we need to inform the OS to
schedule the thread with a RT scheduling policy. As of the time of this
writing, there are three RT scheduling policies on Linux: ``SCHED_RR``,
``SCHED_DEADLINE``, and ``SCHED_FIFO``. Generally, ``SCHED_RR`` should probably
not be used as it is tricky to use correctly\ [#fschedrr]_. ``SCHED_DEADLINE``
is an interesting but advanced scheduler that I may cover in another time. For
most applications, ``SCHED_FIFO`` is likely good enough. With this policy, if a
thread is `runnable <https://tldp.org/LDP/tlk/kernel/processes.html>`__ (i.e.
not blocked due to mutex, IO, sleep, etc.), it will run until it is done,
blocked, or preempted (interrupted) by a higher-priority thread\
[#fschedfifo]_. With `the right system setup
</blog/2022/02-linux-rt-appdev-part2.html>`__, ``SCHED_FIFO`` can be used to
program an RT loop with relatively low jitter (0.05 - 0.2ms depending on the
hardware). This is something that you will know how to do by the end of this
post.

In addition to configuring the thread with an RT scheduling policy, we also
need to give it an RT priority level. If two threads are runnable, the
higher-priority thread will run, even if it means preempting the
lower-priority thread in the process. The priority of a normal Linux thread\
[#fthreadtask]_ is controlled by its `nice values
<https://man7.org/linux/man-pages/man2/nice.2.html>`__ and ranges from -20 to
+19, with the *lower* values taking a *higher* priority. However, these values
are not applicable to RT threads\ [#fnice]_. Instead, the RT priority values of
a thread scheduled by an RT scheduling policy ranges from 0 to 99. Confusingly,
in this system, a *higher* value takes a *higher* priority. Fortunately, nice
values and RT priority values are unrelated and RT threads always have higher
priority than the non-RT threads. The scale for the nice and RT priority values
is illustrated in *Figure 1*.

On a typical Linux distribution with the ``PREEMPT_RT`` patch, there should not
be RT tasks running on the system except for a few built-in kernel tasks.
The kernel interrupt request (IRQ) handlers handle interrupt requests
originating from hardware devices and run with an RT priority value of 50.
These are necessary for communication with the hardware and should generally
not be changed\ [#fprio80]_. Some critical kernel-internal tasks, such as the
process migration tasks and the watchdog task, always run with a RT priority
value of 99. To ensure that the RT application gets priority over the IRQ
handlers, its RT priority is usually set to 80 as a reasonable default.
Userspace RT applications should generally not set its RT to 99 to ensure
kernel-critical tasks can run. These processes are also marked on *Figure 1*.

.. figure:: /static/imgs/blog/2022/04-rt-priority.svg

   *Figure 1*: Diagram depicting the ranges of priority levels on Linux (not to
   scale). ``SCHED_OTHER`` is a non-RT scheduling policy while ``SCHED_FIFO``
   is an RT scheduling policy.

To setup the RT scheduling policy and priority, we can interact with the
``pthread`` API\ [#frtset]_. The C++ standard library defines the ``std::thread`` class as
a cross-platform abstraction around the OS-level threads. However, there is no
C++-native ways to setup the scheduling policy and priority as the OS-level
APIs (such as ``pthread``) are not standardized across platforms. Instead,
``std::thread`` has a ``native_handle()`` method that returns the underlying
``pthread_t`` struct on Linux. With the right API calls, it is possible to set
the scheduling policy and priority after the creation of the thread. However, I
find this to be a bit tedious and prefer interact with the ``pthread`` API
directly so that the thread is created with the right attributes. This code can
then be wrapped into a ``Thread`` class for convenience (`full code is shown
here
<https://github.com/cactusdynamics/cactus-rt/tree/master/examples/blog_examples/basic.cpp>`__):

.. code-block:: c++
   :number-lines:

   // Other includes ...
   #include <pthread.h>

   class Thread {
     int priority_;
     int policy_;

     pthread_t thread_;

     static void* RunThread(void* data) {
       Thread* thread = static_cast<Thread*>(data);
       thread->Run();
       return NULL;
     }

    public:
     Thread(int priority, int policy)
         : priority_(priority), policy_(policy) {}

     void Start() {
       pthread_attr_t attr;

       // Initialize the pthread attribute
       int ret = pthread_attr_init(&attr);
       if (ret) {
         throw std::runtime_error(std::strerror(ret));
       }

       // Set the scheduler policy
       ret = pthread_attr_setschedpolicy(&attr, policy_);
       if (ret) {
         throw std::runtime_error(std::strerror(ret));
       }

       // Set the scheduler priority
       struct sched_param param;
       param.sched_priority = priority_;
       ret = pthread_attr_setschedparam(&attr, &param);
       if (ret) {
         throw std::runtime_error(std::strerror(ret));
       }

       // Make sure threads created using the thread_attr_ takes the value
       // from the attribute instead of inherit from the parent thread.
       ret = pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);
       if (ret) {
         throw std::runtime_error(std::strerror(ret));
       }

       // Finally create the thread
       ret = pthread_create(&thread_, &attr, &Thread::RunThread, this);
       if (ret) {
         throw std::runtime_error(std::strerror(ret));
       }
     }

     int Join() {
       return pthread_join(thread_, NULL);
     }

     void Run() noexcept {
       // Code here should run as RT
     }
   };

   void LockMemory() { /* See previous section */ }

   int main() {
     LockMemory();

     Thread rt_thread(80, SCHED_FIFO);
     rt_thread.Start();
     rt_thread.Join();

     return 0;
   }

The above code snippet defines the class ``Thread`` with three important methods:

#. ``void Start()`` which invokes the pthread API and starts an RT (or non-RT)
   thread.
#. ``int Join()``, which calls ``pthread_join`` and wait for the thread to
   finish.
#. ``void Run() noexcept``, which should contains the custom logic
   that should execute on the RT thread. As this is a demonstration, it is left
   empty. The method is defined with ``noexcept`` as C++ exceptions are not
   real-time safe.

Most of the magic is contained in the ``Start()`` method. The scheduling policy
is set on line 30 and the scheduling priority is set on line 37 and 38. Note
that ``policy_ = SCHED_FIFO`` and ``priority_ = 80`` is set with the
construction of the ``Thread`` object on line 71. The thread is finally started
on line 51. This calls the method ``Thread::RunThread`` on the newly-created RT
thread, which simply calls ``thread->Run()``. This indirection is needed
because pthread takes a function pointer with a specific signature and the
``Run()`` method does not quite have the right signature. Code written within
the ``Run()`` method will be scheduled with the ``SCHED_FIFO`` policy. As
previously noted, this means it won't be interrupted unless preempted by a
higher-priority thread. With this scaffolding (note that ``LockMemory`` is also
included in the example above), we can start writing an RT application.
Since RT applications generally loop at some predictable frequency, we will
look at how the loop itself is programmed for RT in the next section.

If you compile and run `the full code
<https://github.com/cactusdynamics/cactus-rt/tree/master/examples/blog_examples/basic.cpp>`__,
you will likely encounter a permission error when the program starts. This is
because Linux restricts the creation of RT threads to privileged users only.
You'll either need to run this program as root, or edit your user's max
``rtprio`` value in ``/etc/security/limits.conf`` as per `the man page
<https://www.man7.org/linux/man-pages/man5/limits.conf.5.html>`__\
[#flimitconf]_.

.. [#fschedrr] See `56:40 of this talk <https://youtu.be/w3yT8zJe0Uw?t=3400>`__
   for more details about the problems of ``SCHED_RR``.
.. [#fschedfifo] ``SCHED_FIFO`` is a bit more complex than this, but not that
   much more complex especially for a case where there's only a single RT
   process. `See `the man page for sched(7)
   <https://man7.org/linux/man-pages/man7/sched.7.html>`__ for more details.
.. [#fthreadtask] Thread, tasks, and processes are synonymous from the
   perspective of the OS scheduler.
.. [#fnice] Nice values are technically related to the RT priority values.
   However, the actual formula is very confusing. See the `kernel source
   <https://github.com/torvalds/linux/blob/v5.17/include/linux/sched/prio.h>`__
   for details.
.. [#fprio80] In some cases, you need to ensure some IRQ handlers can
   preempt your RT thread, which means you need to set these IRQ handlers'
   priority level to be higher than the application. For example, if the RT
   thread is waiting for network packets in a busy loop with higher priority
   than the network IRQ handler, it may be blocking the networking handler from
   receiving the packet being waited on. In other cases, stopping IRQ handlers
   from working for a long time may even crash the entire system.
.. [#frtset] It is also possible to set RT priority via the `chrt utility
   <https://man7.org/linux/man-pages/man1/chrt.1.html>`__ without having to
   write code, but I find it cleaner to set the RT scheduling policy and
   priority directly in the code to better convey intent.
.. [#flimitconf] If you create the file
   ``/etc/security/limits.d/20-USERNAME-rtprio.conf`` with the content of
   ``USERNAME - rtprio 98``, you may be able to run basic pthread program
   without using ``sudo``. Your mileage may vary, so please consult with the
   man pages for ``limits.conf``.

Looping with predictable frequency
==================================

.. figure:: /static/imgs/blog/2022/04-rt-loop-1.svg

   *Figure 2*: Timeline view of a loop implemented with a) a constant sleep and
   b) a constant wake-up time

If an RT program must execute some code at 1000 Hz, you can structure the loop
in two different ways as shown in *Figure 2*. This figure shows the timeline
view of two idealized loops executing and sleeping, shown with the green boxes
and the double-ended arrows respectively. The simplest way to implement this
loop would be to sleep for 1 millisecond at the end of every loop iteration,
shown in *Figure 2a*. However, unless the code within the loop executes
instantaneously, this approach would not be able to reach 1000 Hz exactly.
Further, if the duration of each loop iteration changes, the loop frequency
would vary over time. Obviously, this is not an ideal way to structure an RT
loop. A better way to structure the loop is to calculate the time the code
should wake up next and sleep until then. This is effectively illustrated in
*Figure 2b* with the following sequence of events:

#. At time = 0, the application starts the first loop iteration.
#. At time = 0.25ms, the loop iteration code finishes.
#. Since the application last woke up at t = 0, it calculates the next intended
   wake-up time to be 0 + 1 = 1ms.
#. The application instructs the OS to sleep until time = 1ms via the
   ``clock_nanosleep`` function.
#. At time = 1ms, the OS wakes up the application, which unblocks the
   ``clock_nanosleep`` function, and the loop advances to the next iteration.
#. This time, loop iteration code takes 0.375ms. The next wake up time is
   calculated by adding 1ms to the last wake-up time, resulting in a new
   wake-up time of 1 + 1 = 2ms. The application goes to sleep until then and
   the loop repeats.

Since this workflow is generic, most of it can be refactored into
``Thread::Run()`` as introduced in the previous section. We can leave a
``Thread::Loop()`` method that actually contains the application logic as
follows (`full code is shown here
<https://github.com/cactusdynamics/cactus-rt/tree/master/examples/blog_examples/loop.cpp>`__):

.. code-block:: c++
   :number-lines:

   // Other includes omitted for brevity
   #include <ctime> // For timespec

   class Thread {
     // Other variables omitted for brevity

     int64_t period_ns_;
     struct timespec next_wakeup_time_;

     // Other function definition omitted for brevity

     void Run() noexcept {
       clock_gettime(CLOCK_MONOTONIC, &next_wakeup_time_);

       while (true) {
         Loop();
         next_wakeup_time_ = AddTimespecByNs(next_wakeup_time_, period_ns_);
         clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &next_wakeup_time_, NULL);
       }
     }

     void Loop() noexcept {
       // RT loop iteration code here.
     }

     struct timespec AddTimespecByNs(struct timespec ts, int64_t ns) {
       ts.tv_nsec += ns;

       while (ts.tv_nsec >= 1000000000) {
         ++ts.tv_sec;
         ts.tv_nsec -= 1000000000;
       }

       while (ts.tv_nsec < 0) {
         --ts.tv_sec;
         ts.tv_nsec += 1000000000;
       }

       return ts;
     }
   }

The ``Run`` method is relatively simple with only 5 lines of code:

#. On line 13, the current time is obtained via ``clock_gettime`` before the
   loop starts. It is stored into the instance variable ``next_wakeup_time_``.
#. On line 15, the loop starts.
#. On line 16, the ``Loop()`` method is called, which should be filled with
   custom application logic (but is empty for demonstration purposes).
#. On line 17, the code add ``period_ns_`` to ``next_wakeup_time_``. Although
   not embedded directly in this post, the `full code
   <https://github.com/cactusdynamics/cactus-rt/tree/master/examples/blog_examples/loop.cpp>`__
   sets ``period_ns_`` to 1,000,000, or 1 millisecond.

   * The addition is performed with a helper method ``AddTimespecByNs``, which
     performs simple arithmetic on the ``timespec`` struct based on its
     definition.

#. On line 18, ``clock_nanosleep`` is called with the argument
   ``TIMER_ABSTIME``\ [#fsleep]_, which instructs Linux to put the process to sleep until
   the moment specified in ``next_wakeup_time_``. When the process is woken up
   again, ``clock_nanosleep`` returns and the code continues execution at line
   15.

It is important to note the usage of ``CLOCK_MONOTONIC`` with ``clock_gettime``
and ``clock_nanosleep``, which gets the current time and sleeps respectively.
These function calls ultimately results in system calls, which are handled by
the OS kernel. The ``CLOCK_MONOTONIC`` argument instructs the kernel to perform
operations based on a "monotonic clock" which increases monotonically with the
passage of time and usually has an epoch that coincides with the system boot
time. This is not the same as the real clock (``CLOCK_REALTIME``), which can
occasionally decrease its value due to clock adjustments such as the
`adjustments made for leap seconds
<https://en.wikipedia.org/wiki/Leap_second>`__. Sleeping until a particular
time with the ``REALTIME`` clock can be very dangerous, as clock adjustments
can cause the sleep interval to change, which may cause deadline misses. Thus,
RT code should only use ``CLOCK_MONOTONIC`` for measurements of time durations.

Trick to deal with wake-up jitter
---------------------------------

In `part 1 </blog/2022/01-linux-rt-appdev-part1.html>`__ and `part 2
</blog/2022/02-linux-rt-appdev-part2.html>`__ of this series, I discussed and
demonstrated how Linux cannot instantaneously wake up your process at the
desired time due to hardware + scheduling latency (a.k.a. wake-up latency). On
a Raspberry Pi 4, I measured the wake-up latency to be up to 130 microseconds
(0.13 ms). This means when ``clock_nanosleep`` returns, it could be late by up
to 130 microseconds. Although the wakeup latency is close to 0 for the vast
majority of the time, RT applications always need to account for the worst
case. This was not considered in the previous example. The more realistic
situation is shown in *Figure 3a*, where the gray boxes now denotes the wake-up
latency. As shown in the figure, the actual start time of the loop iteration
may be delayed by the maximum wake-up latency. This may not be tolerable for RT
systems that cannot tolerate high jitter on the wake-up time.

To reduce this jitter, we can employ the method shown in *Figure 3b*: instead
of sleeping until the next millisecond, the code subtracts the wake-up latency
from the sleep time. The thread thus wakes up at the beginning of the blue box
at the earliest. When the thread wakes up, it busy waits in a loop until the
actual desired wake-up time at t = 1ms, before passing control to the ``Loop``
method. As long as the width of the blue box exceeds the worst-case wake-up
latency, the process should always wake up before the actual desired wake-up
time. In my experience, the actual wake-up time was kept within 10 microseconds
of the target on a Raspberry Pi 4. That said, although the jitter is kept low,
this approach uses significantly more CPU and requires accurate knowledge of
the worst-case wake-up latency\ [#fwakeupadv]_. It is also somewhat more
complex to implement correctly, which means I will not demonstrate the code
directly in this post. Interested readers can look at the implementation of
``cactus_rt::CyclicFifoThread`` in the `cactus-rt repository
<https://github.com/cactusdynamics/cactus-rt/blob/master/include/cactus_rt/cyclic_fifo_thread.h>`__.

.. figure:: /static/imgs/blog/2022/04-rt-loop-2.svg

   *Figure 3*: Timeline view of a loop affected by wake-up latency implemented
   with a) a constant wake-up time and b) premature wake-up and busy wait.

At this point, you basically have everything you need to setup a RT
application. However, I do not recommend using the code snippets presented in
this post directly, as they are very barebone and do not provide a very nice
base to build on. Instead, I recommend you to take a look at my ``rt`` library
as a part of the `cactus-rt repository <https://github.com/cactusdynamics/cactus-rt>`__.
In this library, I define ``cactus_rt::App``, ``cactus_rt::Thread``, and
``cactus_rt::CyclicFifoThread`` similar to the code introduced here. The library has
more features, such as the ability to set CPU affinity, use busy wait to reduce
jitter, and track latency statistics\ [#fadvanced]_. More features may also be
added in the future with further development.

.. [#fsleep] The usage of ``clock_nanosleep`` is preferred over functions like
   ``usleep`` and ``std::this_thread::sleep_for`` as the latter cannot sleep
   until a particular time. The usage of ``std::this_thread::sleep_until``
   might be OK if it is implemented via ``clock_nanosleep`` to ensure that
   high-resolution clocks are used. Personally, I prefer just using
   ``clock_nanosleep`` directly as I know that API is safe for RT.
.. [#fwakeupadv] You also "lose" the CPU time spent in the busy wait
   permanently, which can be an issue.
.. [#fadvanced] Some of these "advanced" configuration will be briefly
   discussed in the appendix below.

Passing data with a priority-inheriting mutex
---------------------------------------------

Most RT applications require data to be passed between RT and non-RT threads. A
simple example is the logging and display of data generated in RT threads.
Since logging and displaying of data is generally not real-time safe, it must
be done in a non-RT thread to not block the RT threads. Usually, the data
generated by a RT thread is collected by the non-RT thread where it is logged
into files and/or the terminal output. Data passing between concurrent threads
are subject to `data races
<https://en.wikipedia.org/wiki/Race_condition#Data_race>`__, which must be
avoided to ensure the correctness of the program behavior. As noted in the
`previous post
</blog/2022/03-linux-rt-appdev-part3.html#cpu-scheduler-avoid-priority-inversion>`__,
there are two ways to safely pass data: (1) with lock-less programming and (2)
with a priority-inheriting (PI) mutex. Although lock-less programming is a very
appealing option for RT, it is too large of a topic to cover now (I will
discuss it in the next post). Instead, the remainder of this post will
demonstrate the safe usage of a mutex in RT, as this is likely good enough for
RT in most situations.

Much like ``std::thread``, C++ defines the ``std::mutex``, which is a
cross-platform implementation of mutexes. Also like ``std::thread``, the
standard C++ API does not offer any ways to set the ``std::mutex`` to be
priority-inheriting. While ``std::mutex`` also implements the
``native_handle()`` that which returns the underlying ``pthread_mutex_t``
struct, the attributes of a pthread mutex `cannot be changed after it is
initialized <https://pubs.opengroup.org/onlinepubs/9699919799/functions/pthread_mutex_init.html>`__.
Thus, unlike ``std::thread``, ``std::mutex`` is completely unusable for
real-time and must be replaced with a different implementation. As a part of my
the ``rt`` library that is defined in the `cactus-rt repository
<https://github.com/cactusdynamics/cactus-rt>`__, I have created ``cactus_rt::mutex``, which
is a PI mutex (`full code is shown here
<https://github.com/cactusdynamics/cactus-rt/blob/master/include/cactus_rt/mutex.h>`__):

.. code-block:: c++
   :number-lines:

   #include <pthread.h>
   #include <cstring>
   #include <stdexcept>

   namespace rt {
   class mutex {
     pthread_mutex_t m_;

    public:
     using native_handle_type = pthread_mutex_t*;

     mutex() {
       pthread_mutexattr_t attr;

       int res = pthread_mutexattr_init(&attr);
       if (res != 0) {
         throw std::runtime_error{std::strerror(res)};
       }

       res = pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
       if (res != 0) {
         throw std::runtime_error{std::strerror(res)};
       }

       res = pthread_mutex_init(&m_, &attr);
       if (res != 0) {
         throw std::runtime_error{std::strerror(res)};
       }
     }

     ~mutex() {
       pthread_mutex_destroy(&m_);
     }

     // Delete the copy constructor and assignment
     mutex(const mutex&) = delete;
     mutex& operator=(const mutex&) = delete;

     void lock() {
       auto res = pthread_mutex_lock(&m_);
       if (res != 0) {
         throw std::runtime_error(std::strerror(res));
       }
     }

     void unlock() noexcept {
       pthread_mutex_unlock(&m_);
     }

     bool try_lock() noexcept {
       return pthread_mutex_trylock(&m_) == 0;
     }

     native_handle_type native_handle() noexcept {
       return &m_;
     };
   };
   }

Most of this code is boilerplate to wrap the pthread mutex into a class that
implements the `BasicLockable
<https://en.cppreference.com/w/cpp/named_req/BasicLockable>`__ and `Lockable
<https://en.cppreference.com/w/cpp/named_req/Lockable>`__ requirements,
allowing it to be used by wrappers such as ``std::scoped_lock``. This makes
``cactus_rt::mutex`` a drop-in replacement for ``std::mutex``. The only line of
interest is line 20, where the priority-inheritance protocol is set for the
mutex. A toy example using the ``cactus_rt::mutex`` is given below (`full code is
shown here <https://github.com/cactusdynamics/cactus-rt/tree/master/examples/blog_examples/mutex.cpp>`__):

.. code-block:: c++
   :number-lines:

   rt::mutex mut;
   std::array<int, 3> a;

   void Write(int v) {
     std::scoped_lock lock(mut);
     a[0] = v;
     a[1] = 2 * v;
     a[2] = 3 * v;
   }

   int Read() {
     std::scoped_lock lock(mut);
     return a[0] + a[1] + a[2];
   }

This just shows two functions that can read and write to the same array ``a``
without data races. As you can see, it is just as easy as ``std::mutex``.

Although ``cactus_rt::mutex`` is safe for RT, simply converting normal mutexes into
``cactus_rt::mutex`` in the code does not guarantee the code to be safe for RT. This
is because the usage of a PI mutex causes the critical sections protected by
the mutex on the non-RT thread to be occasionally elevated to run with RT
priority, and this code may cause unbounded latency due to things such as
dynamic memory allocation and blocking system calls (i.e. everything mentioned
in the `previous post </blog/2022/03-linux-rt-appdev-part3.html>`__). Thus, all
code protected by the PI mutex must be written in an RT-safe way. This is
sometimes not feasible, which means lock-less programming must be employed.

Summary
=======

In this post, I gave a tutorial on how to write an RT application with C++.
Specifically, we went over the following steps:

#. Locking memory with ``mlockall`` on the process level at application
   startup.
#. Manually creating a pthread using the ``SCHED_FIFO`` scheduling policy with
   a default RT priority of 80 using the custom ``Thread`` class.
#. Setting up an RT loop by calculating the next wake-up time and sleeping with
   ``clock_nanosleep``.
#. Safely passing data via a priority-inheriting mutex defined as the class
   ``cactus_rt::mutex``, which is a drop-in replacement for ``std::mutex``.

Along the way, we discussed:

* The importance of using ``CLOCK_MONOTONIC`` as ``CLOCK_REALTIME`` does not
  increase monotonically and therefore could be dangerous for time duration
  calculations.
* The usage of busy wait to minimize wake-up jitter.
* The fact that PI mutexes cause code that are protected by the mutex on the
  non-RT thread to occasionally run with RT priority, which means they need to
  be RT safe and avoid unbounded latency.

All of the examples in this post can be found `here
<https://github.com/cactusdynamics/cactus-rt/tree/master/examples/blog_examples/>`__.
In the next post, I will briefly highlight a few lock-less programming
techniques and hopefully conclude this series.

Appendix: advanced configurations
=================================

One way to further reduce wake-up latency is to use a Linux feature known as
|isolcpus|. This flag instructs the Linux kernel to not schedule any processes
(other than some critical kernel tasks) on certain CPUs. It is then possible to
pin the RT thread onto those CPUs via the CPU affinity feature. This can
further reduce wakeup latency, as the kernel will rarely have to preempt
another thread to schedule and switch to the pinned RT thread. This is
implemented in my ``cactus_rt::Thread`` implementation in `cactus-rt
<https://github.com/cactusdynamics/cactus-rt>`__.

.. |isolcpus| replace:: ``isolcpus``
.. _isolcpus: https://www.kernel.org/doc/Documentation/admin-guide/kernel-parameters.txt

In RT, memory allocation is to be avoided. In other words, all memory must be
allocated before the start of the RT sections. Two additional things may be
considered:

#. Stack memory (where all the local variables live) have a limited size on
   Linux. By default, this is 2MB. Since variables are pushed onto the stack as
   the application code executes, stack overflow can occur during execution if
   the stack variables became too large. This usually results in the process
   getting killed by the kernel, which is obviously undesirable. Since each
   thread has its own private stack, you may need to increase the stack size
   during thread creation via ``pthread_attr_setstacksize``. This is also
   implemented in ``cactus_rt::Thread``.
#. If an O(1) memory allocator implementation is used (i.e. ``malloc`` takes
   constant time excluding the time needed for page faults), it may be OK to
   dynamically allocate memory during the RT sections if the memory allocator
   already reserved the memory from the OS. However, reserved memory may be
   returned to the OS once ``free``'d, which may result in page faults when new
   ``malloc`` calls are made as the total amount of reserved memory is reduced.
   If an O(1) memory allocator is used, you should consider reserving a large
   pool of memory at program startup, and disable the ability for the memory
   allocator to give back memory to the OS. This is currently partially
   implemented by ``cactus_rt::App`` in cactus-rt.

Appendix: References
====================

* `A realtime developer's checklist - LWN <https://lwn.net/Articles/837019/>`__
* `HOWTO build a simple RT application - Realtime Linux Wiki <https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/application_base>`__
* `Memory for Real-time Applications - Realtime Linux Wiki
  <https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/memory#dynamic_memory_allocation_in_rt_threads>`__
* `HOWTO build a basic cyclic application - Realtime Linux Wiki
  <https://wiki.linuxfoundation.org/realtime/documentation/howto/applications/cyclic>`__
* `A checklist for writing Linux real-time applications - John Ogness <https://www.youtube.com/watch?v=NrjXEaTSyrw>`__
* `Challenges Using Linux as a Real-Time Operating System - Michael Madden <https://ntrs.nasa.gov/citations/20200002390>`__

