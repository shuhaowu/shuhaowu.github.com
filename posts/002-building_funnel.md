title: Reinventing the wheel: building funnel
date: 2013-03-17 21:20

Hackers nowadays are all moving to static pages for their blogs/portfolios. 
There are several advantages for this approach:
 
 1. Ease of use: No need to edit in a crappy WYSIWYG editor that generates HTML. 
    Edit directly in a text editor.
 2. Ease of deployment: no need to deploy to a special stack, run a monitoring
    service, a reverse proxy, LAMP, and all that nonsense. Just get anything
    that can serve html pages and you're set. *Hell*, Github even provides you 
    with free hosting!
 3. Ease of scaling: it is probably the case that no one reads your blog during
    regular hours. However, when you write that one *brilliant* article and HN
    decides that they're going to knock on your doors, your server will probably
    fall on its knees if you didn't scale properly. With a static html site, 
    scaling becomes much easier, especially if you let Github do it for you.

(There are probably other reasons, but this is not the point of this post.)

I started making a portfolio sometime ago. I remember using different
technologies: PHP, Python, raw HTML, and so forth. I never knew about things
like Jekyll and Pelican until *very* recently. All I knew is that I have some
stuff I wanna create and ship. So I did. I made things that's embarassing, 
used terrible practises, and learned to hate technologies (\*cough\* php).
So about a year ago (Feb 13th 2012), I took a little bit of time and 
wrote [Funnel](https://github.com/shuhaowu/Funnel/commit/b89c79a4891c30a9d474a647cee9ca25c09012ee).

It was about 80 lines of code and it didn't do anything other than reading some
markdown files and json files in order to put up a static site. I then built a 
my portfolio with the help of Funnel and was quite satisfied.

Over the year, I became increasingly dissatisfied with Funnel. I 
want to give blogging a go again, but Funnel does not have the capability for
that yet. I won't go back to self hosted solution or any sort of CMS as I really
like how the github + static pages setup work. At this point, I have learned 
about things like Jekyll and Pelican and started poking it around. 

First I looked at Jekyll. It didn't really pertain to me. The whole relied on
the Ruby stack. While I don't mind Ruby myself, I just don't have extra time
at the moment to learn everything behind it. I wanted to work with something
that I'm already familiar in and can easily customize without jumping around in
documentations. Also, while I first looked up Jekyll, It wasn't clear to me on
how to author static pages using markdown and the package seemed to be geared towards 
blogging, which is the opposite of what I want: something geared towards static
pages but has blogging as a bonus.

Pelican came next. kernel.org is powered by it now
so it seemed like a reasonable choice. While I read through the docs, the 
first thing that jumped out at me is, again, how
blog focused it is. The second heading of the documentation is "Kickstart a 
blog". The docs also wanted me to get a blogroll in my settings file (first of all, 
why a blogroll. Secondly, why in my settings file! If I want one I can just 
insert it into my blog template). It also wanted me to get a tags page and so forth.
I understand that these are features that could be good to include, but why are
the *required*? Can't the generator just graciously fall back if tags.html is
not found?

To top it all off, both Pelican and Jekyll seems bloated. Sure, it may offer a
lot of features (and lack some that I want, as I never found out how to have 
multiple sections in one page for both), but I fail to see the justification
of having something so hefty. I thought that since programmers are the only ones
using these static site generators (after all, they're not too friendly to use
for normal folks), we would want something that we can hack around to our
likings. 

So at the end of the day I just rewrote Funnel to include blogs. It's
definitely [not pretty](https://github.com/shuhaowu/Funnel/blob/master/funnel).
There are a lot of features that's lacking. It can also be bloated as it
relies on a crap load of libraries (Flask, Frozen-Flask and their dependencies).
However the important thing is that I can now do what I want to do with ease 
and don't have to deal with things like `rake post title="My blawg post"`
(real men (or women, or men identified with women... and so forth) don't use
commands to generate a text file, they `touch` it instead?) and tags.html.

On top of all of that, reinventing the wheel was actually quite fun. I rewrote 
funnel and made it feature complete for me in one night, before a midterm. That
kind of atmosphere is exciting and awesome in my view, even though it may promote 
[bad life style](http://chinpen.net/blog/2013/02/hackathons-are-bad-for-you/).
