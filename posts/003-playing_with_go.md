title: Playing with Go (as a Python developer)
date: 2013-04-05 02:00

It seems like I have gotten into the bad habit of coding and blogging before exams.
My first exam is in 12 days and I have barely started studying (actually,
I barely *know* the content).

That aside, I've taken the last few days to learn Go. I've heard a lot of good
things from HN and I actually had a reason to use something more efficient than
Python.

So I thought I'd try to make a simple 2D video game in Go as a training exercise
first before I jump into doing actual projects. Something like Pong
or Tank would work well to introduce me to most of the major concepts of a
language.

--------------------------------------------------------------------------------

One of the first thing that jumped out at me while writing code in Go is that
I did not feel like I was writing code with static typing. Most of the time, I
was able to leverage the power of the `:=` operator in Go, which is able to
infer the type of the right hand side expression for you. You do have to specify
the types of all function arguments and returns, but I already do that as
documenting functions in Python usually involve variable types.

I also enjoy all the restrictions that Go poses when compiling programs. I've
seen people say that they don't like the fact that Go won't compile if there is
an unused variable, or an unused import. I for one actually like this limitation
as it allowed me to write cleaner code in general, free of left overs from a
previous iteration of the code, or something like that.

Another thing is that I love is how easy it is to use libraries, whether it be
standard libraries or third party ones. Something like
`import "github.com/jmhodges/levigo"` not only makes referencing the
origin of a package easy, `go get -u` can actually grab all the
dependencies automatically without having something like requirements.txt.
On top of that, if you go into your `$GOPATH` and take a look, you'll see that
all the libraries are in folders exactly as you have specified. This means no
more messy Makefiles or the use of an IDE.

Speaking of which, editor plugins are lacking for Go. If you use vim,
emacs, or Sublime Text, you're in luck. Apparently there are awesome extensions
that does what people want. However, if you're like me and use something that's not
in the above three (even though I'm writing this blog post in vim), you might
have some trouble getting support. I personally use Komodo Edit and its support
for Go is *abysmal* (I use the plugin from [here](https://github.com/trentm/komodo-go)).
All I have is syntax highlighting (kinda). There is no autoindent on `{` and
no inline function lookups. Syntax checking is flaky and only works occasionally.

(For those that wants me to switch to vim, emacs, or Sublime: I don't get more
productive with vim or emacs, especially when there is a lot of files, and I'm
not paying for my editor.)

Documentation is excellent. While its "docstring" format makes the inline
documentations look exactly like regular comments, the generated documents are
beautifully consistent. http://godoc.org is awesome as you can just append
the url of most libraries and get all your documentation needs.
If that's not possible, you could always fall back to `go doc`.
There is no inconsistent styles (for things like pages) like in Python, and
generating docs *just works* (in the sense that you don't need to, as `go doc`
takes care of that for you).

I like the idea of statically linking everything into a single binary. While
this is not possible in all cases, it really simplifies deployment as you just
need to push out one binary and be done with it.

This is probably not everything I've encountered, but these are most of the
stuff that stood out. After writing about ~1k LOC during
times when I should be listening and taking note (yay engineering school~),
I can definitely go on writing more (the reaction was different when
I learned Java).

Some stuff I miss as a Python developer: error handling, tuples,
list comprehensions, ternary operator, default arguments, and real OOP.
Regarding OOP: I'm not a big OOP fan myself, but sometimes it is handy. Right
now initializing a struct is a nightmare as everyone seems to have their own
conventions, such as `obj := NewObject()`,
`obj := new(Object) obj.Init()`, `obj := &Object{} obj.Initialize()`
and so forth.

In case we forgot, I made a game with Go. It's called Go Pong. This is actually
a somewhat difficult task due to the lack of 2D libraries.
You can't run it because I had to use
a custom hack in order to make the game work. I don't have anywhere to host
the linux amd64 binary as of the moment. There is, however, a video of the game
in action:

<iframe width="640" height="360" src="https://www.youtube.com/embed/sWwlvhQ1SdU?feature=player_detailpage" frameborder="0" allowfullscreen></iframe>

The source code is at https://github.com/shuhaowu/gopong if anyone is interested.
It makes use of a `SetRGBA` method. This method is a custom made one as `CopyRGBA`
takes way too long and slows the FPS to something like 20 as oppose to being
locked at 30.

As a side note, the collision detection is horrible, as you can probably already
tell.

So what's next on the radar? I'm creating a nifty database (at least in my
opinion) called [Levelupdb](https://github.com/shuhaowu/levelupdb). The goal
is a high performance, low footprint database server that is API-compatible with
Riak. However, the server is targeting low end boxes (&lt;=512MB of RAM, VPSes
you get from LEB) and side projects that do not have a lot of users.

In the mean while, however, I'm going to bed.

**Edit**: The Go plugin for Komodo Edit just got autoindent. You also need to
config gocode to be in Komodo's path in order for autocompletion to work.
