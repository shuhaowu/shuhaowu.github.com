title: Software is Easy, Hardware is on Medium Difficulty
date: 2013-11-11 11:11

The article [Why Hardware Development Is Hard, Part 2: The Physical World Is Unforgiving](http://danluu.com/hardware-unforgiving/)
just showed up on [Hacker News](https://news.ycombinator.com/item?id=6709925).
I encourage you to read it, as it is a fairly interesting read.

The article gave us some convincing arguments about why hardware is difficult, 
the main one being that hardware development takes time and money. When 
"change consume[s] real [and] physical resources", there is no room for errors and
errors are expensive and sometimes even dangerous. 

As an aerospace engineering student, I can definitely appreciate this point. 
It reminds me of those cases in software when we say that the production environment "blew up" because 
some configuration changed. I suspect that that phrase is not muttered by most
aerospace engineers unless that is literally the case, which is definitely within
the realm of possibilites.

------------------------------------------------------------------------------

One thing that annoys me a lot while studying my engineering courses is the
lack of lack of ability to try things while I learn. Learning programming got me very used to the cycle of learn, apply, 
succeed/fail (mostly fail), and learn again. This cycle is not very practical
with my aerospace training. The second step, apply, does not happen very
often. The amount of labs and experiments is too little compared to the amount in my adventures
with software. Lots of labs and experiments are all mostly done for us, leaving
students with little room for exploration. Even then, we rarely fail. Experiments 
and labs are almost guarenteed to be correct. This only confirms that we were right,
and that means we don't get to learn anything new.

That said, I don't really see any *practical* way for the situation to improve.
The real world is hard and is full of constraints (more on that in a later blog
post, perhaps) and these constraints tend to collide with our imaginations.

Students who study computer science can learn much faster, at least in theory,
than their peers in other fields.
Instead of performing an experiment every two weeks, experiments
can be done in real time, as the lecturer teaches. Instead of costing tens or hundres of
thousands of dollars to get equipments for students, a lot of cool things in CS can 
be accomplished, or at least demonstrated in principle, on machines that cost only 
a few hundred dollars. Instead of fiddling with equipments by flicking the 
bubbles out of a pitot tube and getting confused over why there is a large deviation 
between your experimental results and standard values, 
you can expect relatively the same, or even exactly the same results in CS. 
All of these can be traced back to limitations faced by traditional engineering.

Another interesting point is that in software, you're encouraged to make 
mistakes. The *[fail often, fail early](http://www.codinghorror.com/blog/2006/05/fail-early-fail-often.html)*
mentality works like a charm when your production environment is almost the same 
as your development environment<sup>1</sup>. This kind of mentality will cost
traditional engineering a lot of time, effort, and even in some extreme 
circumstances, lives. Everythings needs to be thoroughly understood and analyzed
before they are even pushed into being built (which I'd like to think as 
compilation). Just imagine programming with pens and papers, except you only get one chance
to compile and run your program until next month (hello there older programmers!). 
The fail often and fail early mentality definitely would not scale well, if I may.

So at this point, perhaps the title "Software is easy" is too careless. Software
is not really easy. Hard software problems are still hard and it takes a lot of time and focus
to solve these problems. Perhaps software is *easy to learn*. As in students get
to learn things faster than traditional engineering students by at least five
orders of magnitude (2 weeks vs 2 minutes). By that logic, engineers with decades
of experiences may have roughly the same of amount of experiences as some college
student who has been programming since a very young age and is now creating her
own startup (I'm probably wrong, however).

Saying that hardware is hard, in the other hand, is probably wrong. Hardware is hard is more like an
opinion, or the lack of willingness to tackle the problem. Limitations are not something that are commonly thought about
by programmers. We tend to think that we can conquer the world with code. We tend
to forget about the fundamental rules and boundaries that we cannot break. All
of this, made worse by the fact that the work we do is essentially writing rules 
for a miniture universe. There is seemingly no limits to what we can accomplish
because we are the lawmakers. If we are faced by restrictions, we tend to think it is hard. It's not that the logic here is
wrong, it's just that other people have accepted the limits while programmers
have not. Hardware is medium difficulty as that is the norm, the norm where being
subject to a lot of restrictions that are difficult work around.

------------------------------------------------------------------------------

<sup>1</sup>: Okay. I hear you scream about how your production servers are so different
from your MacBook Pro. I've had that problem too. I just think that an AutoCAD
rendering of your machine is fundamentally different than a copy of your 
rendering that you can hold in your hand. That is just an opinion, though,

