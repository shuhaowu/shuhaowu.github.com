.. meta::
   :title: Does Wayland fractional scaling work with games in 2025?
   :authors: Shuhao Wu
   :created_at: 2025-11-07

*Author's note: my understanding of fractional scaling is based on some casual
reading and as an user. I'm thus missing a lot of detail and nuance. If you
find a factual error in this post, please email me and I'll correct it. Thank
you.*

Due to the historical choice of `only allowing integer scales in Wayland
<https://dudemanguy.github.io/blog/posts/2022-06-10-wayland-xorg/wayland-xorg.html#:~:text=For%20some%20reason%20unknown%20to%20me%2C%20the%20Wayland%20protocol%20only%20supports%20integer%20scale%20values%2E%20To%20be%20frank%2C%20this%20is%20asinine%20and%20everyone%20pays%20the%20price%20for%20it%2E>`_,
fractional scaling on Linux is commonly implemented via integer overscale then
fractional downscaling technique. The intuition is that for a given screen
height of let's say 1350 pixels and a desired fractional scale of 1.5x, we
really want render GUI elements as if it is a screen that's only
:math:`\frac{1350}{1.5} = 900` pixels tall. Since Linux apps in 2025 generally
can only render at integer scales, we either have to render with 900 pixels or
1800 pixels (neglecting >2x scales). The former would require an upscale of
1.5x, which would be blurry due to the lack of information in the frequency
domain. The latter would require a downscale of 1.5x, which would be generally
acceptable if done well [#f1]_. Thus, most Wayland compositors implements this
technique. However, this technique poses a problem for performance-sensitive
applications like games. Unlike normal GUI applications, games have shaders that
run by default for every pixel. The increase in pixel count due to the integer
overscale could result in significantly more pixel shader runs [#f2]_. All the
work done by the extra pixel shader runs are then discarded during the
downscaling process (neglecting potential supersampling benefits). Effectively,
we just did a bunch of work for nothing and wasted performance.

What further complicates this story is that most games were written for X11 and
must run on top of Xwayland. On GNOME (<47), Xwayland applications were rendered
at 1x and upscaled, which results in a blurry image. Since GNOME 47, `Xwayland
applications can now also be overscaled then downscaled
<https://release.gnome.org/47/#:~:text=GNOME%2047%20includes%20an%20enhanced%20fractional%20display%20scaling%20feature%2C%20which%20provides%20better%20support%20for%20legacy%20X11%20apps%2E>`_
like normal Wayland applications, resulting in a cleaner image. This means games
running on Xwayland may choose the overscaled resolution for rendering, which
would negatively impact performance.

This problem did not go unnoticed during the initial work to make Xwayland scale
fractionally in GNOME. The `GNOME developers state that this is a feature
<https://gitlab.gnome.org/GNOME/mutter/-/merge_requests/3567#note_2191910>`_, as
most applications only render well with integer scales and have a myriad of
potentially incorrect fractional scaling behaviours [#f3]_. This then justifies
the default choice of overscaling to maximize compatibility and usability.
Games, on the other hand, tend to render at the highest available resolution,
which would perform poorly with overscale. It was pointed out by the developers
that most games have ways to configure their output resolution, which can be
used to render the game at the native or a lower-than-native resolution. This
can be used in tandem with "Xwayland's xrandr mode change emulation", which
"will" result in no performance impact. I don't fully understand what Xwayland's
xrandr mode change emulation is. Presumably, if a user configures the game to
output a resolution lower than the physical resolution of the screen, the
compositor has to upscale it to the physical screen's resolution. My concern is
that the compositor will upscale the game's output to the overscaled resolution
before downscaling it to the native resolution even if the game is configured to
render at the native resolution. It is possible that the compositor recognizes
that the game's resolution is identical to the native resolution and skip the
extra scaling, but I don't know for sure.  If anyone knows, email me and I'll
correct this section. Regardless though, even if the overhead is non-existent as
stated, this remains an awful user experience as it requires manual
configuration for every game. Otherwise, the game would likely try to render at
the highest resolution (which is overscaled) and perform poorly. There is also
the occasional game that does not allow for resolution selection and simply
renders at the highest available resolution [#f4]_, which would also perform
poorly.

Further, I believe that mode change emulation doesn't work with windowed mode
games, which is commonly used. This affects both Xwayland and most
Wayland-native games. Games in windowed mode tend to support arbitrary shapes
and simply render to the resolution the assigned to that shape without an option
to change. In fractional scaling, that resolution would be overscaled, which
once again results in performance issues. This can be easily replicated on GNOME
49: Figure 1 is a screenshot showing an overscaled Xwayland-based RetroArch
running in windowed mode on a 4k screen at a resolution higher than 4k.

.. figure:: /static/imgs/blog/2025/01-fractional-scale-retroarch.png

   *Figure 1*: RetroArch's viewport in GNOME is significantly bigger in pixels than the 4k display it is on, despite only occupying a subset of the entire screen space.

So gaming on GNOME with fractional scaling is broken for both full screen and
windowed mode. What about KDE? KDE scales most Wayland applications with the
same overscale-then-downscale technique. For Xwayland applications, KDE has two
system-wide configurable ways to scale:

1. **Apply scaling themselves**: By default, KDE will simply provide the native
   resolution to the Xwayland app and let the application handle scaling.
2. **Scaled by the system**: In this mode, KDE will present the resolution for
   1x scaling to the Xwayland app and then upscale it.

In the first mode, games will pick the right resolution out of the box and not
suffer any performance penalties. Applications running in this mode will need to
fractionally scale themselves given the resolution and DPI. If the application
can only scale with integers, it will likely render incorrectly. In the second
mode, games and applications will be upscaled and blurry, which is not really
usable. Unlike GNOME, KDE does not offer a way to overscale then downscale for
Xwayland applications, so there is no performance footgun here.

Lastly, I should mention the existence of `wp-fractional-scale-v1
<https://wayland.app/protocols/fractional-scale-v1>`_. Wayland applications that
add support for this can allocate a correctly-sized surface to draw on and skip
the overscale-then-downscale process. This is similar to much like how KDE
treats X11 applications by default and requires applications to properly render
the scaling on their own.  Since this was only `merged
<https://gitlab.freedesktop.org/wayland/wayland-protocols/-/merge_requests/143>`_
relatively recently, a whole generation of Wayland code had already been written
without it. I do not expect applications to universally support this any time
soon. That said, Chrome has already `merged support for this
<https://chromium-review.googlesource.com/c/chromium/src/+/4322839>`_ and
Firefox support is available behind the flag
``widget.wayland.fractional-scale.enabled``. Perhaps Proton will support this in
the future, which would allow games to properly run on the right resolution, and
skip all the mess with Xwayland and scaling.

So does Wayland fractional scale work with games in 2025? Assuming you have the
latest GNOME and KDE, we can refer to the following table:

+-------+----------------+---------------------+------------------------------+-----------------+
| DE    | Display server | Scaling type        | Games                        | Application     |
+=======+================+=====================+==============================+=================+
| GNOME | Wayland        | Overscale           | ❌ Bad performance           | 🆗 Usable       |
+-------+----------------+---------------------+------------------------------+-----------------+
| GNOME | Wayland        | fractional-scale-v1 | ❓ Nothing to test with      | ✅ Best         |
+-------+----------------+---------------------+------------------------------+-----------------+
| GNOME | XWayland       | Overscale           | ❌ Bad out-of-box experience | 🆗 Usable       |
+-------+----------------+---------------------+------------------------------+-----------------+
| KDE   | Wayland        | Overscale           | ❓ Not tested                | 🆗 Usable       |
+-------+----------------+---------------------+------------------------------+-----------------+
| KDE   | Wayland        | fractional-scale-v1 | ❓ Nothing to test with      | ✅ Best         |
+-------+----------------+---------------------+------------------------------+-----------------+
| KDE   | XWayland       | Scale themselves    | ✅ Best performance          | 🆗 Depends      |
+-------+----------------+---------------------+------------------------------+-----------------+
| KDE   | XWayland       | Upscale             | ❌ Blurry image              | ❌ Blurry image |
+-------+----------------+---------------------+------------------------------+-----------------+

Given that most games are using Xwayland these days, it seems like KDE is the
best option if you want both fractional scaling and play games. Generally
speaking, if you desktop environment allows Xwayland applications to directly
render to the native resolution and apply scaling themselves, then you should be
good to go. If all the games you play implements wp-fractional-scale-v1, then
you're also good to go. Alternatively, you can go back to using fraction font
scaling with integer resolution scaling, which may not work well with
heterogeneous multi-monitor setup.

For the future, I would like to see three things:

1. Games need to implement wp-fractional-scale-v1. This seems like it will take
   a long time, unless this can be supported in Proton quickly.
2. For XWayland applications, it would be nice to be able to select, on a
   per-application basis, whether to use the scale themselves technique (which
   is effectively the same as wp-fractional-scale-v1) or apply overscale then
   downscale. Having a per-application choice in addition to a system-wide
   default will make things more flexible.
3. GNOME should implement the scale themselves approach for XWayland apps. I
   think the ideological resistance to implementing this for GNOME is that some
   apps will be broken visually if they cannot fractionally scale themselves.
   However, if #2 is implemented, the responsibility of whether or not an
   application can fractionally scale falls on the user instead of communication
   between the application and the compositor. GNOME can then wash its hands on
   this problem and users would be happier.

Another interesting idea would be to use machine learning to upscale from 1x
scaling, like using FSR. This is something that Gamescope can already do, but I
wonder how it would work for applications.

.. [#f1] This is effectively a `supersampling <https://en.wikipedia.org/wiki/Supersampling>`_ effect, although the community seems to be `ideologically against this <https://dudemanguy.github.io/blog/posts/2022-06-10-wayland-xorg/wayland-xorg.html#:~:text=This%20strikes%20me%20has%20more%20ideologically%20motivated%20rather%20than%20anything%20technical.>`_, calling it impossible or blurry.
.. [#f2] For a given scaling factor :math:`s` between 1 - 2, the overhead is :math:`(2\frac{p}{s})^2/p^2 = \frac{4}{s^2}`, which translates to 2.56x, 1.78x, 1.31x the pixel count at 1.25x, 1.50x, and 1.75x scale respectively.
.. [#f3] I'm not convinced that this is fully accurate, but I do not want to litigate this point.
.. [#f4] The resolution selection could also be done by running Gamescope, a microcompositor by Valve. This is still an awful experience as it requires per-game configuration.
