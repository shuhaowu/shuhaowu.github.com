title: Orbital Mechanics 101: Keplerian Orbits
published: no
mathjax: yes

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

    INSERT ORBIT HERE!
    Technical requirements:
     - r vector
     - v vector
     - eccentricity vector
     - orbit with animation

An ideal orbit involving two bodies is very predictable. Naturally, there are several parameters that are constant for that orbit. These values allow us to calculate almost all of the characteristics of an orbit. These values are constant no matter where you are on the orbit and will only change when you change the orbit.

The first of the trio is the **orbital angular momentum**. This is defined as the cross product between the radial vector and the velocity vector and it is denoted as \\( \vec{h} \\):

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

    INSERT ORBIT WITH ADJUSTABLE a AND e.

A spacecraft orbit in 2D is an ellipse with the focal point as the center of mass of the planet. An ellipse, can be defined with two parameters known as the **semi-major axis** (\\( a\\)) and the **eccentricity** (\\(e\\)).

Semi-major axis is half the distance of the widest part of the orbit.
Eccentricity, on the other hand, is the ratio between the distance from the
center of the ellipse to the focal point and the semi major axis. This ratio
varies between 0 and 1, with 0 being a perfectly circular orbit, and 1 being a
parabolic orbit, or an orbit that will escape the gravity of this planet. For
anything in between, this will result in an elliptical orbit.

    INSERT ORBIT WITH LABELED p, b, a, e, rp, ra

There are some additional parameters that will aid us with our calculations later. These values can be derived from a and e (or be used to derive a and e). They are outlined as follows:

- **semi-latus rectum**: \\( p = a(1-e^2) \\)
- **semi-minor axis**: \\( b = a \sqrt{1 - e^2} \\)
- **minimum distance (periapsis)**: \\(r_p = \frac{p}{1-e} = a(1 + e) \\)
- **maximum distance (apoapsis)**: \\(r_a = \frac{p}{1+e} = a(1 - e) \\)

These can be derived from equations governing the basic operations of an ellipse
and hence I won't go into how they are actually derived.

### Kepler's Laws ###

    INSERT ORBIT WITH MOVING BODY.

With most of the tedious mathematical background out of the way, we can now start to formulate Kepler's Law.

