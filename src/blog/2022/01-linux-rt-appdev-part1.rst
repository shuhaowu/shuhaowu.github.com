.. meta::
   :title: Real-time programming with Linux, part 1: What is real-time?
   :authors: Shuhao Wu
   :created_at: 2021-04-01 20:45

Part 1 - `Part 2 </blog/2022/02-linux-rt-appdev-part2.html>`_ - `Part 3 </blog/2022/03-linux-rt-appdev-part3.html>`_

Recently I've been looking at real-time applications from two different angles:
(1) audio applications and (2) advanced robotics. When developing live audio
production applications, the audio data must be written to the sound card at a
constant frequency with low latency. Missing the deadline generally results in
pops and cracks emitted by the speaker. When developing advanced `controllers
<https://en.wikipedia.org/wiki/Control_theory>`_ for a robot, the system also
must read, compute, and write data to the hardware at a constant frequency with
low latency. Failure to do so can result in the controller becoming unstable,
potentially causing damage to property and life.

This is a series where I document my journey on real-time C++ development on
Linux. While there are lots of materials out there, ranging from blog posts and
presentations, to PhD thesis and journal articles, they tend to cover only a
subset of the problem that combines *real-time*, *C++*, and *Linux* together.
As a result, I found it difficult to tie everything I have read together to
create a real-time C++ application for Linux from scratch. This series
hopefully addresses that problem in a way that's coherent and correct.

What is "real-time"?
====================

Before I get started, it's perhaps a good idea to first review what "real-time"
is. From the `Wikipedia definition`_, a "real-time program must guarantee
response within specified time constraints, often referred to as 'deadlines'".
This definition, in my opinion, is vague and perhaps too broad. For example,
is a messaging app a real-time program? Under the above definition, you could
argue that it is, as the message delivery time likely should remain bounded for
the message to be helpful for me and my friends. What about a video game where
each frame should be rendered within 1/60th of a second?

.. _Wikipedia definition: https://en.wikipedia.org/wiki/Real-time_computing

To better distinguish the different types of real-time systems, we can place a
particular piece of software on two independent axes: maximum allowed latency
and consequence severity of missed deadline. With these two axes, I can plot
(subjectively) as follows:

.. figure:: /static/imgs/blog/2022/01-rt-classification.svg

I like this two-axes system, as each application is unique and must be
evaluated individually. That said, this is not how applications are typically
classified in the literature. Instead, there are a lot of references to "hard"
and "soft" real-time systems, which are not consistently defined everywhere.
"Hard" real-time systems tend to be the ones in the top-left corner of the
above chart where you have very-low-latency requirements as well as
safety-related consequences for missing deadlines\ [#f1]_. Some common examples
of these are aircraft or robot control systems and medical monitoring systems.
"Soft" real-time systems tend to the ones on the middle-center of the chart,
where the latency requirement is difficult to achieve on traditional desktop
operating systems and the consequence of an occasional deadline miss is
undesirable but somewhat tolerable. The best example of this would be audio
production, where the latency must not be perceivable to the musicians.

These terminologies, in my opinion, are just mostly arguing semantics (with
perhaps the exception of the mathematically-proven systems). All software
systems are "real-time" to some extent. If you pressed a key on your computer ,
and it didn't respond for a few minutes, you'll likely forcefully reboot it. If
this happens all the time, you'll either throw the computer out, or at least
install a different operating system on which this doesn't happen. Thus, the
answer to the question of "what is real-time" is "it depends". The requirements
of the application dictate what kind of guarantees you need from your code,
the libraries you call, the operating system you use, and the hardware you
deploy on. If you write an application today with off-the-shelf operating
systems and hardware, deadlines are basically guaranteed to be met if they are
large enough (≥10-100 milliseconds) and if the application is written correctly
(e.g. don't ``sleep`` in a time-critical section). However, if the deadline is
within a few milliseconds, and if the consequence of missing such a deadline is
not acceptable, the code likely will have to be "real-time". This means the
developer has to pay more attention into the libraries, operating system, and
the hardware that the application relies on.

.. [#f1] There is a (classic) school of thought where "hard" real-time systems
   are the ones that are mathematically provable. If I had to pick a
   definition, I would say that a hard real-time system is a
   mathematically-proven one where as everything else would be a soft real-time
   system. Under this model, I suspect only a few systems would be hard
   real-time.

Overview of latency sources
===========================

.. figure:: /static/imgs/blog/2022/01-rt-latencies-overview.svg

To be able to develop an application that can meet its deadlines every time,
the developer must be able to predict the worst-case execution time of every
line of code that runs. As shown in the diagram above (which is not to scale),
there are three sources of latency for most applications: (1) the application,
(2) the operating system and its scheduler, and (3) the hardware.

In the simplest case, the hardware executes a single thread of code that you
have written. An example of this would be something like an Arduino. In such a
system, the worst-case latency is purely caused by the application latency
(green in the plot), as there are no operating system or firmware-level code
executing on the CPU. In this situation, the worst-case latency can be
relatively easily deduced by reading through the code in a line-by-line fashion
and performing benchmarks. This architecture becomes less and less viable as
more computations and IO operations are added in the main loop of the program.
Certain operations, such as disk writes, can be very slow. Once added to the
main loop, these long-running operations can block the execution of the
time-critical code and cause it to miss its deadlines. Thus, other program
architectures are required for such complex real-time systems to meet its
deadlines\ [#f2]_.

On the opposite side of the spectrum with respect to the Arduino-like code are
the applications written for a traditional operating system (OS) such as
Linux. Although thousands of `tasks
<https://en.wikipedia.org/wiki/Task_(computing)>`_ may share a few CPUs,
application code written for these OSes are not affected by the presence of
these other tasks. In fact, I would argue that the experience of writing a
single-threaded application on an OS like Linux feels very similar to writing
Arduino-like code. The operating system schedules the execution of tasks via
its scheduler and switches between these tasks as it sees fit\ [#f3]_, generally
without any specialized code within the application. This allows your
computer to both perform a computationally-heavy task such as running a
simulation at the same time as responding to key presses in a timely manner.
The ability to balance between computational throughput and IO latency is one
of the key pieces of "magic" provided by the OS. However, this magic has a
cost that cannot be ignored for real-time systems. The worst-case time cost of
the OS scheduler\ [#f4]_ must be bounded and known to successfully develop a
real-time application. This is what is labeled as "scheduling latency" in the
above figure, and it occurs before the application code executes.

Finally, the hardware itself may introduce additional latency via a number of
completely different mechanisms. The most famous example is the `system
management interrupt (SMI)
<https://wiki.linuxfoundation.org/realtime/documentation/howto/debugging/smi-latency/start>`_,
which can introduce an unpredictable amount of delay as it hijacks the CPU from
both the application and the operating system. Further, a modern CPU usually
has dynamic frequency scaling based on its utilization to provide a balance
between performance and power consumption. This can cause larger-than-expected
delays as the system performance is not uniform with respect to time. Other
factors like `SMT <https://en.wikipedia.org/wiki/Simultaneous_multithreading>`_
(more commonly referred to as hyper-threading) can also impact latency. I have
even seen bad clocks on a single-board computer causing higher-than-expected
latency. I have lumped these, as well as other sources of hardware-related
latency not listed here\ [#f5]_, together as the "hardware latency". The latency of the
hardware must be determined via benchmarks and (possibly) tuned to ensure the
success of a real-time system.

.. [#f2] See chapter 1 of *Siewert, S., & Pratt, J. (2015). Real-Time Embedded Components and Systems with Linux and RTOS* for a more comprehensive review on real-time architectures.
.. [#f3] To learn more about this, see `context switch <https://en.wikipedia.org/wiki/Context_switch>`_.
.. [#f4] The latency comes from more than just the scheduler. To learn more
   about this for Linux, check out `this talk <https://www.youtube.com/watch?v=-J0y_usjYxo>`_. It also includes some examples of hardware-induced latency.
.. [#f5] See `this page <https://rt.wiki.kernel.org/index.php/HOWTO:_Build_an_RT-application#Hardware>`_ as a start for other sources of hardware latency.

Summary
=======

In the first part of this series, we've defined what a "real-time system" is.
We've also summarized the terminology of "soft" and "hard" real-time systems
with the conclusion that the definition is not universally agreed upon.
However, given that present technologies can only achieve deadlines above the
orders of 10 milliseconds, applications such as robotics controllers that
have deadlines of a few milliseconds require the careful examination and
validation of the hardware, operating system, and the application code.

In the `next post </blog/2022/02-linux-rt-appdev-part2.html>`_, I will write a
very simple program that can achieve a maximum latency of 1ms and configure it
to run on Linux.
