title: Orbital Mechanics 101: Keplerian Orbits
published: no
mathjax: yes
orbitjs: yes

Orbital mechanics. That thing that Kerbal Space Program does in the background when you put on the Mun. While it is [a lot of fun][ksp-blog-ref] to fool around with in games, how does it actually work? With this post, we will take a look at the basics on Keplerian orbits and gain some insights on how KSP and real life, really works.

The objective of this post is to develop a set of equations that allow us to
calculate the position and velocity at any time during an orbit.

![To the Mun!](/static/img/ksp-moon/05-to-the-mun.png)

[ksp-blog-ref]: /blog/kerbal_space_program_fun.html

Note that if you are reading this on a RSS reader, you will not be served the in browser demo. Mobile may or may not have issues depending on your system. It is recommended that you open this page on a laptop/desktop.

--------------------------------------------------------------------------------

### Quick review on physics ###

This post makes some assumption over your knowledge of physics. Specifically, you should have some basic knowledge of energy, momentum, and forces. Additionally, you should know some basic calculus and vectors. Knowledge of differential equations will help you, but it is not a requirement.

With that said, let's start with gravity. In orbital mechanics, gravity is the primary force that acts on all bodies, whether it is a spacecraft, a moon, or an astronaut. The force of gravity is defined by:

$$\vec{F} = -\frac{G m_1 m_2}{|\vec{r}|^3} \vec{r}$$

Where \\( \vec{F} \\) is the force in its vector form, \\( G \\) is the gravitational constant, \\( m_i \\) is the mass of the \\( i \\)th mass, and \\( \vec{r} \\) is the distance between the center of mass of these two mass. This is [Newton's Law of Gravitation][law-of-gravitation].

[law-of-gravitation]: http://en.wikipedia.org/wiki/Newton%27s_law_of_universal_gravitation

We can see that the force of gravity gets stronger as the masses of the two objects in question gets larger. It also gets weaker as the distance between the two objects grow larger, on the order of a quardratic.

By assuming that our satellite has much smaller mass than the planet we are orbiting around, we can eliminate one of the mass terms. Combined with the gravitational constant, we have an expression for the **standard gravitational parameter**:

$$G m_1 m_2 \approx G m_1$$

$$\mu = Gm_{\mathrm{planet}}$$

This value will pop up everywhere in our calculations later. So make a note! For your reference, this parameter for Earth and Kerbin is respectfully:

$$\mu_{\mathrm{Earth}} = 398600 \mathrm{\frac{km^3}{s^2}} $$

$$\mu_{\mathrm{Kerbin}} = 3531.5 \mathrm{\frac{km^3}{s^2}} $$

### Constants of Keplerian Motion ###

<div id="view1" class="orbitalviewport">
  <canvas class="background" width="500" height="500"></canvas>
  <canvas class="foreground" width="500" height="500">Your browser is not supported</canvas>
</div>
<p class="orbitalcaption">
    This is a highly eccentric orbit.
    <br/>
    The <span class="red">red arrow</span> is the radial/radius vector and the <span class="blue">blue arrow</span> is the velocity vector.
    <br />
    The solid circle is the center body, the hollow circle is the orbiting satellite, and the black ellipse is the orbital path.
    <br />
    This convention will be used later as well.
</p>

An ideal orbit involving two bodies is very predictable. Naturally, there are several parameters that are constant for that orbit. These values allow us to calculate almost all of the characteristics of an orbit. These values are constant no matter where you are on the orbit and will only change when you change the orbit.

The first of the trio is the **orbital angular momentum**. This is defined as the cross product between the radial vector ( \\( \vec{r} \\) ) and the velocity vector ( \\( \vec{v} \\) ) and it is denoted as \\( \vec{h} \\):

$$ \vec{h} = \vec{r} \times \vec{v} $$

We won't go into the details on exactly what this means as it is a slightly more complicated concept and beyond the scope of this post. You can learn more about it [here][angular-momentum].

[angular-momentum]: http://en.wikipedia.org/wiki/Angular_momentum

The second of the trio is the **orbital specific energy**. This measures the amount of energy that your spacecraft has. If you're in an orbit, this value will be negative. If it is zero or positive, your spacecraft has enough energy to escape the gravity of the orbiting body.

Orbital specific energy is defined as:

$$ \epsilon = \frac{v^2}{2} - \frac{\mu}{r} $$

Note that the values here does not have an arrow on top, meaning they are the absolute value of the vector. \\( v \\) corresponds to the speed and \\( r \\) corresponds the distance to the center of the planet.

The last one is the **eccentricity vector**. The eccentricity vector is a vector point to the lowest point of the orbit, known as the **periapsis**. We will elaborate this more when we talk about the orbital characteristics.

The eccentricity vector is defined as:

$$ \vec{e} = \frac{\vec{v} \times \vec{h}}{\mu} - \frac{\vec{r}}{r} $$

### Orbital Elements in 2D ###

A spacecraft orbit in 2D is an ellipse with the focal point as the center of mass of the planet. An ellipse, can be defined with two parameters known as the **semi-major axis** (\\( a\\)) and the **eccentricity** (\\(e\\)).

Semi-major axis is half the distance of the widest part of the orbit.
Eccentricity, on the other hand, is the ratio between the distance from the
center of the ellipse to the focal point and the semi major axis. This ratio
varies between 0 and 1, with 0 being a perfectly circular orbit, and 1 being a
parabolic orbit, or an orbit that will escape the gravity of this planet. For
anything in between, this will result in an elliptical orbit.

<div id="adjustable_view" class="orbitalviewport">
  <canvas class="background" width="500" height="500"></canvas>
  <canvas class="foreground" width="500" height="500">Your browser is not supported</canvas>
</div>
<p class="orbitalcaption">
    Try adjusting the parameters to see the effects yourself!
    <br />
    <label for="adjustable_a">Semimajor Axis (km): </label>
    <input id="adjustable_a" type="range" min="600" max="10000" value="800">
    <span id="adjustable_a_value">800</span> km
    <br />
    <label for="adjustable_e">Eccentricity (km): </label>
    <input id="adjustable_e" type="range" min="0" max="1000" value="0">
    <span id="adjustable_e_value">0</span>
    <br />
    Note: since we only have limited size, the planet rescales with the orbit size. 
    <br />
    Also, if your orbit is inside the planet, that means your satellite will crash!
</p>

There are some additional parameters that will aid us with our calculations later. These values can be derived from a and e (or be used to derive a and e). They are outlined as follows:

- **semi-latus rectum**: \\( p = a(1-e^2) \\)
- **semi-minor axis**: \\( b = a \sqrt{1 - e^2} \\)
- **minimum distance (periapsis)**: \\(r_p = \frac{p}{1-e} = a(1 + e) \\)
- **maximum distance (apoapsis)**: \\(r_a = \frac{p}{1+e} = a(1 - e) \\)

These can be derived from equations governing the basic operations of an ellipse
and hence I won't go into how they are actually derived. The following diagram shows what they mean physically:

<p class="text-center">
<img src="/static/img/orbit101/labeled-orbit.png" alt="labeled orbit" />
</p>


### Kepler's Laws ###

    INSERT ORBIT WITH MOVING BODY.

With most of the tedious mathematical background out of the way, we can now start to formulate Kepler's Law.

<script>
    window.onload = function() {
        var highly_eccentric_orbit = new Orbits.Elliptic2D({
          a: 4200,
          e: (8/10),
          mu: Orbits.constants.mu_kerbin,
          r: 600,
          show_r_vector: true,
          show_v_vector: true,
        });
        var view1canvas = new Canvas("view1", null);
        view1canvas.set_orbit(highly_eccentric_orbit);

        var adjustable_orbit = new Orbits.Elliptic2D({
          a: 800,
          e: 0,
          mu: Orbits.constants.mu_kerbin,
          r: 600,
          show_r_vector: true,
          show_v_vector: true,
        });
        var adjustable_view_canvas = new Canvas("adjustable_view", null);
        adjustable_view_canvas.set_orbit(adjustable_orbit);

        var adjustable_semimajor_axis_control = $("#adjustable_a");
        var adjustable_semimajor_axis_value = $("#adjustable_a_value");
        var adjustable_eccentricity_control = $("#adjustable_e");
        var adjustable_eccentricity_value = $("#adjustable_e_value");

        adjustable_semimajor_axis_control.on("input", function() {
          adjustable_semimajor_axis_value.text(parseInt(adjustable_semimajor_axis_control.val()));
        });

        adjustable_semimajor_axis_control.on("mouseup", function() {
          adjustable_orbit.a = parseInt(adjustable_semimajor_axis_control.val());
          adjustable_orbit.recompute();
          adjustable_view_canvas.set_orbit(adjustable_orbit);
        });

        adjustable_eccentricity_control.on("input", function() {
          adjustable_eccentricity_value.text(parseInt(adjustable_eccentricity_control.val()) / 1000);
        });

        adjustable_eccentricity_control.on("mouseup", function() {
          adjustable_orbit.e = parseInt(adjustable_eccentricity_control.val()) / 1000;
          adjustable_orbit.recompute();
          adjustable_view_canvas.set_orbit(adjustable_orbit);
        });
    };
</script>