.. meta::
   :title: Real-time software development with Linux, part 3: setup a real-time application
   :authors: Shuhao Wu
   :created_at: 2021-04-20 16:20
   :draft: true

`Part 1 </blog/2022/01-linux-rt-appdev-part1.html>`__ - `Part 2 </blog/2022/02-linux-rt-appdev-part2.html>`__ - `Part 3 </blog/2022/03-linux-rt-appdev-part3.html>`__ - Part 4

In the last post

The scaffolding that I am creating in this post uses the `POSIX real-time API
<https://unix.org/version2/whatsnew/realtime.html>`__ to ensure the operating
system knows that the process requires real-time capabilities. While I am only
going to test this on Linux, this scaffolding should in principle also work
with other POSIX-real-time-compliant operating systems (such as QNX, but no
promises).

Setting up a real-time application
==================================

Process setup with ``rt::App``
------------------------------

Memory locking and reservation
------------------------------

Real-time thread setup with ``rt::Thread``
------------------------------------------

Passing data via ``std::atomic``
--------------------------------

Passing data via ``boost::lockfree::spsc_queue``
------------------------------------------------

Writing a basic ``cyclictest``
==============================

Validation
==========
