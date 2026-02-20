.. meta::
   :title: Fast change detection with generation counters in JavaScript
   :authors: Shuhao Wu
   :created_at: 2026-02-20
   :has_code: true
   :draft: true

*Author's note: As I spend a lot of my time working with JavaScript, I noticed a
lot of gaps in that community with respect to high-performance programming.
Today's post is an easy one that really grinds my gears. So I wrote it despite
how simple it is.*

A fundamental part of GUI programming is to re-render the screen when data
changes. This is done in two ways: (1) a push-based system where updating the
data triggers the re-rendering and (2) a pull-based system where the data is
read and re-rendered in some high-frequency loop, usually coinciding with the
monitor refresh rate.

Most JavaScript frameworks operate exclusively with the push-based model. This
is sensible for most JavaScript applications, as they typically re-render due to
infrequent user interactions. However, for applications where data updates are
automatically done by the system, this push-based model can be less than ideal
as the framework controls the timing and execution of the re-render, which can
lead to performance issues that cause `jank`_.  This problem becomes
increasingly acute as the required render rate approaches the monitor's refresh
rate. Video games reside at the high end of the refresh rate spectrum and they
universally employ a pull-based render loop.

.. _jank: https://developer.mozilla.org/en-US/docs/Glossary/Jank

A problem that comes with a naively implemented render loop is that the code
must run even if the data did not change, leading to redundant computation
between frames. This can be especially problematic in a web application as the web
is mainly a form of `retained-mode`_ UI where the browser retains the rendered
elements until they are changed. It is simply not feasible to rebuild the entire
DOM tree (and things like GL contexts) every frame. Thus, a method for change
detection is needed. In the JavaScript ecosystem, it seems like the de facto
standard pattern comes from React (and React-inspired frameworks), which uses
object identity to check for changes by default. This is a very unfortunate way
to detect changes, as it requires allocating a new object and copying the data
from the original. Since JavaScript is a garbage-collected (GC) language, this
can severely increase the GC load. Applications written with this kind of change
detection that also happen to re-render at high frequency can have 30% of their
time spent in the V8 garbage collector, which is remarkable in a bad way.

.. _retained-mode: https://en.wikipedia.org/wiki/Retained_mode

In game programming, this is frequently solved using a `dirty flag`_. It is a
boolean that marks some data to require re-rendering. In pseudo-code, it looks
something like this:

.. code-block:: javascript
   :number-lines:

   const data = {
     value: 0,
     dirty: false,
   };

   function update() {
     data.value += 1
     data.dirty = true;
   }

   function render() {
     if (!data.dirty) {
       return;
     }

     // do rendering

     data.dirty = false;
   }

.. _dirty flag: https://gameprogrammingpatterns.com/dirty-flag.html

The advantages of this are clear: it is efficient as we are reading and writing a
single additional boolean in the update and render code. The downside is
that the state of the dirty flag must be manually synchronized with the update
and render code, which introduces the potential for bugs. For performance, this
trade-off is worth it, especially if you spend 30% of your time in garbage
collection.

However, this pattern is not easily applied verbatim to web applications.
In games, rendering frequently is coded in a single place, allowing a single
entity to flip the dirty state back to false. However, in web applications, the
GUI may be split into reusable components that render independently in their own
contexts. Consider a case of a real-time stock display with two components: (1)
an X-Y chart showing the price of the last few minutes and (2) a text label
showing the current price. While the data can be shared by both components, they
render independently from each other:

.. code-block:: javascript
   :number-lines:

   const data = {
     time: [],
     price: [],
     dirty: false,
   };

   function updateData(currentTime, currentPrice) {
     data.time.push(currentTime);
     data.price.push(currentPrice);
     data.dirty = true;
   }

   function renderPlot() {
     chart.render(data.time, data.price);
     // set dirty here?
     requestAnimationFrame(renderPlot);
   }

   function renderLabel() {
     label.textContent = data.price[data.price.length-1];
     // or set dirty here?
     requestAnimationFrame(renderLabel);
   }

In this structure, there's no obvious place to reset the dirty flag as setting
the dirty flag to false in either render function would cause the other
component not to render. A solution to this problem is commonly known as the
**generation counter**. This is a counter that counts the number of
"generations" the variable has gone through (might be better named
as the revision number). Instead of setting a dirty flag to true when the data is
changed, we can simply increment the counter. Instead of checking if the dirty
flag is true, each component must check if the generation counter for the data
is different from the last time we rendered it (which is called the *observed
generation*):

.. code-block:: javascript
   :number-lines:

   const data = {
     time: [],
     price: [],
     generation: 0, // counter instead of boolean
   };

   function updateData(currentTime, currentPrice) {
     data.time.push(currentTime);
     data.price.push(currentPrice);

     data.generation++; // increment instead of set true
   }

   // Track the value of the counter from the last render iteration
   let plotObservedGeneration = -1;

   function renderPlot() {
     // Check if the counter of the data is different since last iteration
     if (data.generation !== plotObservedGeneration) {
       // Render only if is different
       chart.render(data.time, data.price);

       // Update the observed generation counter
       plotObservedGeneration = data.generation;
     }

     requestAnimationFrame(renderPlot);
   }

This incurs about the same time overhead as the dirty flag solution and uses an
additional integer per component to store the observed generation. I doubt this
will even show up in benchmarks with JavaScript due to how fast JavaScript
engines can be.

There is one minor problem with this code as the generation is a
double-precision floating point number and can only represent integers safely
until ``Number.MAX_SAFE_INTEGER``. To get around this problem, we can create a
low-overhead wrapped increment utility:

.. code-block:: javascript
  :number-lines:

   function incrGenerationCounter(counter) {
     return (counter + 1) >>> 0; // wraps max_uint32 + 1 to 0
   }


This efficiently solves the problem of change detection for pull-based rendering
in JavaScript. The downside remains that it adds some accounting overhead and
requires additional variables to be synchronized. Some structure should be
imposed for each application based on its performance requirements and coding
conventions. For example, one approach is to create a type that holds the data
alongside the generation counter in an object and modify it with a function:

.. code-block:: typescript
  :number-lines:

   type DataWithGeneration<T extends Record<string, unknown>> = {
     data: T;
     generation: number;
   }

   function modifyData<T extends Record<string, unknown>>(
     data: DataWithGeneration<T>,
     f: (d: T) => void
   ): void {
     f(data.data)
     data.generation = incrGenerationCounter(data.generation)
   }

   function hasDataChanged<T>(data: DataWithGeneration<T>, observedGeneration: number): boolean {
     return data.generation != observedGeneration
   }

That said, you should probably create your own structure for this.

Related concepts
----------------

* Kubernetes uses ``metadata.generation`` and ``status.observedGeneration`` to
  perform a similar `change detection <https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/5067-pod-generation>`_ for its operator reconciliation loop.
* Bevy (game engine) uses `a global generation counter
  <https://github.com/bevyengine/bevy/pull/1471>`_ to track changes.
* `Vector clocks <https://riak.com/why-vector-clocks-are-easy/>`_ uses a
  monotonically-incrementing counter to resolve conflicts in distributed
  databases.

