.. meta::
   :title: Real-time software development with Linux, part 2: writing a real-time program on Linux
   :authors: Shuhao Wu
   :created_at: 2021-04-01 19:00
   :draft: true

In the `last post <blog/2022/01-linux-rt-appdev-part1.html>`_, I went over the
definition of real-time as well as the sources of latency. In this post, I will
go over the practical side of setting up a small C++ program that can achieve a
maximum latency of 1 millisecond (ms) with the right kernel and hardware
combinations.

Linux, real-time, and ``PREEMPT_RT``
====================================

From the application's perspective, the operating system (OS) must offer an API
to perform real-time operations like setting scheduler policy and priority. The
Linux kernel offers this as a part of its compliance with the `POSIX real-time
extension API <https://unix.org/version2/whatsnew/realtime.html>`_. Although
Linux provides these APIs, it makes no guarantee that it can actually achieve
the deadlines required by the real-time application. Unlike commercial
real-time operating systems (RTOS) such as VxWorks and QNX, Linux was not
originally designed to be a RTOS. As a result, mainline Linux generally cannot
keep its end of the real-time bargain despite the availability of the real-time
API. Despite this, many people have successfully used Linux in real-time
systems, usually via one of the following approaches:

* Two CPU method: deploy the real-time code on a RTOS on one CPU and deploy
  non-real-time code on a Linux on a physically-separate CPU.
* Hypervisor method: run Linux as a non-real-time process on a RTOS and the
  real-time code as a real-time process. `Xenomai <https://xenomai.org>`_ takes
  this approach.
* Patched kernel method: patch the Linux kernel such that it can achieve very
  low latency and behave like a RTOS\ [#f1]_. The `PREEMPT_RT
  <https://wiki.linuxfoundation.org/realtime/start>`_ patchset takes this
  approach. The eventual goal for ``PREEMPT_RT`` is for it to be merged into
  mainline Linux, which by then would mean the kernel is a real-time-capable
  system if configured properly.

The first two approaches can be paired with "hard" RTOSes which can achieve
very good results with good guarantees of low jitter (:math:`O(10\mathrm{\mu
s})`). However, programming these platforms will likely be more complex. The
``PREEMPT_RT`` kernel generally has higher jitter (:math:`O(100\mathrm{\mu
s})`) and less guarantee, but it has significantly wider hardware support and
a simpler programming interface\ [#f2]_. My understanding is that most
"mission-critical" real-time code prefer the first two approaches due to the
perceived (or proven) higher reliability and lower latency. That said, as the
``PREEMPT_RT`` patchset matures\ [#f3]_, I suspect it will continue to gain
momentum as it attracts more developers. This blog series focuses on the
``PREEMPT_RT`` approach.

.. [#f1] To learn more about how this is done, see `this excellent talk
   <https://www.youtube.com/watch?v=-J0y_usjYxo>`_.
.. [#f2] Writing RT code directly on Linux could also be argued to be more
   complex, because you have to ensure all code paths that's called within the
   kernel does not exceed the required deadline. This may involve auditing
   millions of lines of code, with is usually not feasible. That said, if we
   trust the kernel does the right thing, then the code would be more
   straightforward.
.. [#f3] For example, National Instruments' real-time offering are now `moving
   towards Linux with PREEMPT_RT
   <https://www.ni.com/content/dam/web/pdfs/phar-lap-rt-eol-roadmap.pdf>`_.

Configuring Linux for real-time
===============================

Install and configure ``PREEMPT_RT``
------------------------------------

To run a real-time program on Linux

Enable PREEMPT_RT.

Disable RT throttle and lockup/hung task detector.

Test the system
---------------

Disable CPU frequency scaling and SMT.

Check cyclictest, with and without load.

Link to check list video and real-time list.

Setting up a real-time application
==================================

Scheduling
----------

Memory management
-----------------

Measuring monotonic time
------------------------

Dealing with priority inversion
-------------------------------

Some advanced settings
======================

Summary
=======
