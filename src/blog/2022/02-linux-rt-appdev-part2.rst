.. meta::
   :title: Real-time programming with Linux, part 2: configuring Linux for real-time
   :authors: Shuhao Wu
   :created_at: 2021-04-11 18:30

`Part 1 </blog/2022/01-linux-rt-appdev-part1.html>`_ - Part 2 - `Part 3 </blog/2022/03-linux-rt-appdev-part3.html>`_

In the `last post </blog/2022/01-linux-rt-appdev-part1.html>`_, I went over the
definition of real-time (RT) and listed a few sources of latency. I noted that
applications with a latency requirement of less than about 10 milliseconds likely
need special handling of the hardware and operating system configurations, as
well as the application code. I mentioned that I will write a small program
this post to demonstrate this. However, this won't be the case, as a lot more
setup is needed before a program can be written. Thus, this post will only
cover the setup of a computer running Linux with the real-time patch, as well
as showing a quick example of identifying a source of latency via ``ftrace``.
The code will come in the next episode.

Linux, real-time, and ``PREEMPT_RT``
====================================

When writing a RT application, the application must let the operating system
(OS) know that it needs real-time capabilities by setting configurations such
as the process scheduling policy and priority. This must be done through an
OS-level API, as the application and OS must cooperate to achieve such low
latency values. Linux offers this as a part of its compliance with the `POSIX
real-time extension API <https://unix.org/version2/whatsnew/realtime.html>`_.
That said, compliance with this API does not mean today's mainline Linux can
actually achieve the low *scheduling latency* requirement of some RT
applications, especially for those that require latency values that are less
than 1-5 milliseconds. This is because Linux was not originally designed to be
a real-time operating system (RTOS) and has not finished its transformation
into one. Despite not being a RTOS, and perhaps due to Linux's popularity,
people still wanted to use it in real-time systems. This is usually
achieved via one of the following approaches:

* *Two-CPU approach*: deploy the real-time code with a RTOS on one CPU and
  deploy non-real-time code with Linux on a physically-separate CPU.
* *"Hypervisor" approach*: run Linux (and all of its non-RT processes) as a
  non-RT process on a RTOS and the RT code as an RT process on the RTOS. This
  is done all on one CPU. `Xenomai <https://xenomai.org>`_ takes this approach.
* *Patched-kernel approach*: patch the Linux kernel such that it can achieve very
  low latency and behave like a RTOS. The |PREEMPT_RT|_ patches takes this
  approach.

  * The ``PREEMPT_RT`` patches are being progressively merged into mainline
    Linux. When all patches are merged, the Linux kernel could become a RTOS if
    properly configured.

.. |PREEMPT_RT| replace:: ``PREEMPT_RT``
.. _PREEMPT_RT: https://wiki.linuxfoundation.org/realtime/start

The first two approaches can be paired with a "hard" RTOS. These kinds of setup
can usually guarantee scheduling latency values on the order of 10 μs. However,
programming these systems is more complex as they likely require specialized
libraries and interfaces to spawn the RT process, as well as to facilitate
communication between the RT and non-RT processes. The ``PREEMPT_RT`` approach
has higher worst-case scheduling latency values which are in the order of 100
μs. Further, it also cannot provide the same level of theoretical guarantees
(i.e. the max latency can only be derived from experimentation) when compared
to other "hard" RTOSes, as there are no mathematical models for the Linux
kernel. However, programming an RT application with ``PREEMPT_RT`` is
significantly easier as the application can rely on the normal Linux APIs for
threading and inter-thread/process communication. Linux with ``PREEMPT_RT``
also supports a much wider range of hardware than specialized RTOS kernels,
which may further simplify development. As the ``PREEMPT_RT`` continues to
mature\ [#f2]_, I suspect it will enable more real-time applications to be
written due to its lower barrier of entry. This may further increase its
popularity, resulting in a positive feedback loop. This blog series will focus
on the ``PREEMPT_RT`` approach\ [#f3]_.

.. [#f2] For example, National Instruments' real-time offering are now `moving
   towards Linux with PREEMPT_RT
   <https://www.ni.com/content/dam/web/pdfs/phar-lap-rt-eol-roadmap.pdf>`_.
.. [#f3] If you're interested in learning more about the comparisons between
   all three approaches, take a look at `this talk
   <https://www.youtube.com/watch?v=BKkX9WASfpI>`__.

Configuring the system for real-time
====================================

For now, ``PREEMPT_RT`` is a set of patches that is supposed to be applied on
top of mainline Linux. Most Linux distributions do not build it by default, and
you will most likely have to do it yourself\ [#f4]_. How this can be done falls
outside the scope of this post, but there are plenty of `guides
<https://docs.ros.org/en/foxy/Tutorials/Building-Realtime-rt_preempt-kernel-for-ROS-2.html>`_
out there. Hopefully in the near future, all of ``PREEMPT_RT``'s functionality
will be merged in to mainline, and Linux distributions will provide RT-enabled
kernels out-of-the-box.

Once you successfully compiled the RT kernel, the default hardware and OS
configurations are usually not tuned correctly for RT. The following hardware
and OS configurations should likely always be checked and tuned:

* Disable `simultaneous multithreading
  <https://en.wikipedia.org/wiki/Simultaneous_multithreading>`__ (SMT, also
  referred to as hyper-threading for Intel CPUs)

  * SMT improves the performance of the CPU but decreases the determinism, thus
    introducing latency. How this works is outside the scope of this post. As
    of this writing, it is recommended for SMT to be disabled\ [#f5]_.

  * SMT is usually configured on the BIOS/UEFI level. How this is done varies
    depending on the system.

* Disable `dynamic frequency scaling <https://wiki.archlinux.org/title/CPU_frequency_scaling>`__

  * Modern CPUs ramp down their clock frequencies while idling and ramp up
    when there is load. This introduces unpredictability as it causes the
    performance of the CPU to vary with time. Anecdotally, I have noticed an
    order of magnitude higher worst-case latency when frequency scaling is on
    compared to when it is off.

  * How this can be turned off varies per system. Usually this involves
    configuring both the BIOS/UEFI and Linux (usually by selecting the
    ``performance`` CPU frequency governor).

* `Disable RT throttling <https://wiki.linuxfoundation.org/realtime/documentation/technical_basics/sched_rt_throttling>`__

  * Before the widespread availability of multicore systems, if an RT process
    uses up all of the available CPU time, it can cause the entire system to
    hang. This is because the Linux scheduler will not run a non-RT process if
    the RT process continuously hogs the CPU. To avoid this kind of system
    lockup, especially on desktop-oriented systems where any process can
    request to be RT, the Linux kernel has a feature to throttle RT processes
    if it uses 0.95 s out of every 1 s of CPU time. This is done by
    pausing the process for the last 0.05 s and thus may result in deadline
    misses during the moments when the process is paused\ [#f6]_.

  * This can be turned off by writing the value ``-1`` to the file
    ``/proc/sys/kernel/sched_rt_runtime_us`` on every system boot.

* Check and make sure no unexpected RT processes are running on your system

  * Sometimes, the base OS can spawn a high-priority RT process on boot as a
    part of some functionalities it provides. If these functionalities are not
    needed, it is advisable to disable the offending RT process. Near the end
    of this post, I will provide an example for this.

  * Sometimes, the kernel can be configured with such a process. See
    documentation on the kernel build variables ``CONFIG_LOCKUP_DETECTOR`` and
    ``CONFIG_DETECT_HUNG_TASK``.

  * Disabling these processes usually involves consulting with the
    documentations of your Linux distribution of choice.

There are other configurations that may be relevant depending on your use case,
some of which are documented in `this talk
<https://www.youtube.com/watch?v=NrjXEaTSyrw>`__ and `this other talk
<https://www.youtube.com/watch?v=w3yT8zJe0Uw>`__. Additionally, quality-of-life
configurations, such the variables in ``/etc/security/limits.conf``, may need
to be tuned as well. I encourage the reader to look at pre-made distributions
such as the `ROS2 real-time Raspberry Pi image
<https://github.com/ros-realtime/ros-realtime-rpi4-image>`__ (which I
incidentally also worked on) for more inspiration. Although providing a
complete checklist for system configuration is outside the scope of this post
(if it is even possible), I included an non-exhaustive checklist `at the bottom
of this post <#appendix-hardware-and-os-configuration-checklist>`__ as a
starting point.

.. [#f4] Debian, notably, has the ``PREEMPT_RT`` kernel as a `package
   <https://packages.debian.org/bullseye/linux-image-rt-amd64>`_ you can
   install.
.. [#f5] Starting from Linux 5.14, there is a `new feature that enables more
   sophisticated scheduling behavior <https://lwn.net/Articles/861251/>`_,
   which may enable an RT application to run on a real core while allowing the
   rest of the system to use SMT. That said, I personally don't think the
   benefit is worth the complexity.
.. [#f6] It can be argued that an RT process probably should not exceed 95% of
   CPU time and the throttler may be a good way to detect that either the
   program is badly optimized, or the CPU performance is not good enough. Also,
   even in modern mixed-used environments, such as an audio workstation (which
   is RT) where the computer may also be used by the user for day-to-day tasks,
   it is perhaps a bad idea to disable RT throttling for the reasons mentioned
   in the main text.

Acceptance testing for Linux and the hardware
=============================================

Say if the advice given above are followed, the latency still may remain high.
We must verify that the hardware and OS combination actually produces
acceptable hardware and scheduling latency in practice, as a number of things
can go wrong within the numerous layers of the system. Anecdotally, I have
observed an out-of-tree kernel driver that caused an additional 3 ms of
scheduling delay even with ``PREEMPT_RT`` applied. `Others have also observed a
~400 μs delay caused by a hardware system management interrupt (SMI) every 14
minutes <https://youtu.be/w3yT8zJe0Uw?t=1536>`__, during which the firmware is
performing tasks related to memory error correction. Thus, it is important
characterize the hardware + scheduling latency to verify that the system on
which the application will be deployed is acceptable.

The typical benchmark used to detect hardware and scheduling latency is `cyclictest
<https://wiki.linuxfoundation.org/realtime/documentation/howto/tools/cyclictest/start>`__.
It roughly implements the following pseudocode:

.. code::

   while (true) {
     t1 = now();
     sleep(interval);
     t2 = now();
     latency = t2 - t1 - interval;
     log(latency);
   }

This code starts by takes the time stamp ``t1``. It will then ``sleep`` for some
small ``interval``, which passes the control back to the kernel until the
kernel wakes up the process after the ``interval`` has passed. At this point,
the code takes another time stamp ``t2``. If the hardware + scheduling latency
is zero, then ``t2 - t1 == interval``. If ``t2 + t1 > interval``, then either
the Linux kernel or the hardware must have taken up the additional CPU time. The
latency is thus calculated via ``t2 - t1 - interval``. Depending on the
command-line flags passed to ``cyclictest``, the distribution for ``latency``
is logged either via their minimum, average, and maximum values, or in a
histogram. This code is repeated in a loop until the process terminates.
``cyclictest`` also typically runs as the only `userspace
<https://en.wikipedia.org/wiki/User_space_and_kernel_space>`__ RT process on
the system. This means the kernel will try to schedule and switch to it as soon
as possible, as RT processes gets picked ahead of all other normal processes
running on Linux. If configured correctly, ``cyclictest`` will measure the
"best-case" hardware + scheduling latency for a given hardware + OS
combination.

Usually, ``cyclictest`` should run simultaneously with a stress test of the
various subsystems (CPU, memory, storage, network, etc.) of the computer being
commissioned, as an idle system is unlikely encounter the conditions that
trigger significant latency. By running the test for a long enough period of
time, one can get a sense of what the worst-case latency can be expected from
the system. Depending on the use case for the RT application, this may provide
a good enough guarantee\ [#f7]_. As a demonstration for this post, I ran
``cyclictest``\ [#f8]_ on a Raspberry Pi 4 running `this RT image
<https://github.com/ros-realtime/ros-realtime-rpi4-image>`__ while it is idling
and while it is under a CPU stress test\ [#f9]_. The data exported by
``cyclictest`` is used to generate the following latency histograms for the
"stock" kernel (``5.4.0-1052-raspi``) and the kernel with ``PREEMPT_RT``
applied (``5.4.140-rt64``):

.. figure:: /static/imgs/blog/2022/02-rt-vs-non-rt-cyclictest.svg

   *Figure 1*: RT vs non-RT ``cyclictest`` latency histograms. Left plot shows
   the system idling. Right plot shows the system under CPU stress.  `Click
   here </static/imgs/blog/2022/02-rt-vs-non-rt-cyclictest.svg>`_ to make it
   bigger.

When the system is idling (left plot), the scheduling latency values observed
under both the RT and non-RT kernel are very similar. However, when a heavy CPU load
is applied (right plot), the ``cyclictest`` experiences significantly higher
maximum latency under the non-RT kernel, at 717 μs. With the ``PREEMPT_RT``
patch applied, the maximum latency under stress is significantly better, at 279
μs. Depending on the requirements of the RT application, the system can then be
accepted or rejected. I was surprised in this case, as it is my understand that
the typical worst-case scheduling latency of the ``PREEMPT_RT`` kernel is
around 100 μs, not 200+. So I decided to investigate further.

.. [#f7] There is always a chance that the benchmark miss some extreme edge
   case which results in higher scheduling latency than the worst-case latency
   observed in the benchmark. See `this presentation
   <https://www.osadl.org/HOT-Heidelberg-OSADL-Talks-on-May-4-an.hot-2021-05.0.html#c15936>`__
   for an example of this.
.. [#f8] I ran cyclictest with the command ``cyclictest --mlockall --smp
   --priority=80 --interval=200 --distance=0 -D 15m -H 400
   --histfile=cyclictest.log``. The test duration was only 15 minutes, which is
   good enough for this demonstration but likely too short for validating a
   system. From what I've seen, people run these for hours to days to gain more
   confidence.
.. [#f9] I also ran the tests under other conditions, as documented `here
   <https://github.com/shuhaowu/rt-demo/blob/56e2ddc/data/cyclictest-rpi4/plot.ipynb>`__.
   I ran these test scenarios under the recommendations of various talks I've
   seen. So far, I'm not aware of a standard set of tests that one should
   perform, and I'm not even sure if that is posssible or appropriate.

Finding latency source with ``ftrace``
--------------------------------------

To determine the source of the latency, I traced the system using `ftrace
<https://en.wikipedia.org/wiki/Ftrace>`__, `trace-cmd
<https://trace-cmd.org/>`__, and `kernel-shark <https://kernelshark.org/>`__\
[#f10]_. Specifically, I used the ``wakeup_rt`` latency tracer, which can
produce a function call trace for the kernel during the event that produced the
maximum scheduling/wakeup latency. This is done via the following command:

.. code::

   $ sudo trace-cmd start -p wakeup_rt cyclictest --mlockall --smp --priority=80 --interval=200 --distance=0 -D 60s

This code starts ``cyclictest`` for 60 seconds under the ``wakeup_rt`` tracer.
I ran this simultaneously with ``stress-ng -c 4``, which puts a high CPU load
on all 4 CPU cores of the Raspberry Pi. After the test is complete, I showed
the result of the test via the command ``sudo trace-cmd show``, which produced
the following (abbreviated) output:

.. code::

   # tracer: wakeup_rt
   # wakeup_rt latency trace v1.1.5 on 5.4.140-rt64
   # latency: 400 us, #345/345, CPU#1 | (M:preempt_rt VP:0, KP:0, SP:0 HP:0 #P:4)
   #    -----------------
   #    | task: cyclictest-12905 (uid:0 nice:0 policy:1 rt_prio:80)
   #    -----------------
   #
   #                    _------=> CPU#
   #                   / _-----=> irqs-off
   #                  | / _----=> need-resched
   #                  || / _---=> hardirq/softirq
   #                  ||| / _--=> preempt-depth
   #                  ||||| / _--=> preempt-lazy-depth
   #                  |||||| / _-=> migrate-disable
   #                  ||||||| /     delay
   # cmd     pid      |||||||| time   |  caller
   #     \   /        ||||||||   \    |  /
   stress-n-12898     1dN.h4..    1us :    12898:120:R   + [001]   12905: 19:R cyclictest
   [omitted for brevity]
   stress-n-12898     1d...3..   57us : cpu_have_feature <-__switch_to
   multipat-1456      1d...3..   58us : finish_task_switch <-__schedule
   [omitted for brevity]
   multipat-1456      1d...3..  382us : update_curr_rt <-put_prev_task_rt
   multipat-1456      1d...3..  383us : update_rt_rq_load_avg <-put_prev_task_rt
   multipat-1456      1d...3..  384us : pick_next_task_stop <-__schedule
   multipat-1456      1d...3..  384us : pick_next_task_dl <-__schedule
   multipat-1456      1d...3..  385us : pick_next_task_rt <-__schedule
   multipat-1456      1d...3..  389us : __schedule <-schedule
   multipat-1456      1d...3..  389us :     1456:  0:S ==> [001]   12905: 19:R cyclictest

While the output can be somewhat difficult to parse (and I'm not an expert at
this point, either), we can see that the maximum scheduling latency observed by
``ftrace`` is 400 μs on CPU #1. This is significantly higher than the earlier
observed 279 μs, which is expected as ``ftrace`` incurs performance penalties for
low-latency processes when it is turned on. On the left, we can see two
columns: ``cmd`` and ``pid``. These correspond to the process command name and
its process ID. In the middle, we see the ``time`` column, which corresponds to
the moment that certain functions are called. The trace starts when the
kernel attempts to wake up ``cyclictest`` at 0 μs. From the three mentioned
columns, we can see that the kernel switched from the ``stress-ng`` process to
the ``multipathd`` process at 58 μs. It then proceed to spend 331 μs in
``multipathd``, before finally switching to ``cyclictest``. This is very
surprising. I would have expected the kernel to switch to ``cyclictest``
immediately, as it is supposed to be the only real-time application running on
the system. This turned out to be the wrong assumption, as a quick ``ps``
showed that ``multipathd`` is a RT process with its RT priority set to 99,
which is higher than the priority of 80 I assigned for ``cyclictest``:

.. code::

   $ ps -e -o pid,class,rtprio,comm | grep 1456
   1456 RR      99 multipathd

Since a process with a higher priority gets scheduled first, it explains why
the latency is higher than I anticipated. At this point, I `filed a
bug against the Raspberry Pi 4 RT image
<https://github.com/ros-realtime/ros-realtime-rpi4-image/issues/30>`_. I then
disabled ``multipathd`` and retested the system's latency. The maximum latency
went from 279 μs to 138 μs, which is more in line with my expectations. The
latency histogram (see figure below) did not change much. This is
understandable, as further tracing\ [#f11]_ showed that ``multipathd`` executes
code for a small period of time about once a second, which means it only
interfered with ``cyclictest`` a small number of times.

.. figure:: /static/imgs/blog/2022/02-rt-vs-rt-no-multipathd.svg

   *Figure 2*: Scheduling latency with and without interferance from ``multipathd``.


.. [#f10] These tools, when used together, can trace various function calls
   within the kernel. The usage of these tools are complex, and I'm not very
   experienced with them yet. In the future, when I gain more experience with
   it, I may consider writing more about them. For now, the reader can refer to
   these articles and conference talks for more details: `(a)
   <https://www.youtube.com/watch?v=Tkra8g0gXAU>`__, `(b)
   <https://lwn.net/Articles/425583/>`__, and `(c)
   <https://www.youtube.com/watch?v=0uu0ElnjLas>`__.
.. [#f11] I traced ``cyclictest`` with ``sudo trace-cmd record -e
   'sched_wakeup*' -e sched_switch cyclictest --mlockall --smp --priority=80
   --interval=200 --distance=0 -D 60s`` and visualized the resulting trace with
   ``kernelshark``.



Summary
=======

In the second part of this series, we briefly surveyed different approaches of
running Linux for a real-time system. We settled for the ``PREEMPT_RT``
patches, as it transforms Linux into an RTOS and therefore simplify application
development and hardware support. Since modern hardware and software are
complex and generally not tuned for real-time out-of-the-box, I presented a few
BIOS- and kernel-level configurations that should always be checked and
configured to ensure consistent real-time performance. To verify that the
tuning actually made a difference, I introduced and demonstrated the usage of
``cyclictest``, a program that can measure hardware + scheduling latency from
Linux userspace. Through this exercise, I found a problem with the Raspberry Pi
4 ROS2 RT image due to a "rogue" RT process that is a part of the base system.
This highlights the necessity of validating both the hardware and the operating
system to ensure good real-time performance, before even writing a single line
of application code.

In the `next post </blog/2022/03-linux-rt-appdev-part3.html>`__, I will
actually talk about where I wanted to get to with this post: setting up a
simple C++ application in RT on Linux + ``PREEMPT_RT``.

Appendix: References
====================

These are some of the more relevant materials I've reviewed as I wrote this post:

* `Understanding a Real-Time System - Steven Rostedt <https://www.youtube.com/watch?v=w3yT8zJe0Uw>`__
* `A Checklist for Writing Linux Real-Time Applications - John Ogness <https://www.youtube.com/watch?v=NrjXEaTSyrw>`__
* `Finding Sources of Latency on your Linux System - Steven Rostedt <https://www.youtube.com/watch?v=Tkra8g0gXAU>`__
* `The Magic Behind PREEMPT_RT - Haris Okanovic <https://www.automateshow.com/filesDownload.cfm?dl=Haris-MagicBehindPREEMPTRT.pdf>`__

Appendix: Hardware and OS configuration checklist
=================================================

This serves as a non-exhaustive starting point on the things to check for the
hardware and OS. The list is constructed based on my survey of the literature
(mostly conference talks, with some internet articles). Remember to always
validate the final scheduling latency with something like ``cyclictest``!

* Disable SMT
* Disable dynamic frequency scaling
* Check for the presence of `system management interrupts <https://wiki.linuxfoundation.org/realtime/documentation/howto/debugging/smi-latency/start>`__; if possible, consult with the hardware vendor (remember to always verify their claims)
* Understand the `NUMA <https://en.wikipedia.org/wiki/Non-uniform_memory_access>`__ of the computer and minimize cross-node memory access within the RT process
* Disable RT throttling
* Disable any unneeded RT services/daemons already running on the OS
* Possibly setup ``isolcpu`` (or use cgroups to accomplish the same thing)
* Look into kernel configurations that may affect RT performance such as
  ``CONFIG_LOCKUP_DETECTOR``, ``CONFIG_DETECT_HUNG_TASK``, ``CONFIG_NO_HZ``,
  ``CONFIG_HZ_*``, ``CONFIG_NO_HZ_FULL``, and possibly more.
* Configure the memory lock and rtprio permissions in
  ``/etc/security/limits.d``.
