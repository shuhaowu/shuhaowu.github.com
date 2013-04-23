title: Riak, Leveldb, and the birth of Levelupdb
date: 2013-04-23 16:00

Time for another exam, so that means another blog post, right? Yeah, I thought
so too.

For those of you that may follow me on [github](https://github.com/shuhaowu) 
(ha! yeah right), you probably know me as a long time Riak user. Personally,
I have yet to find a use for something like PgSQL and MongoDB for a lot of
the simple projects I've been working on. So far, most of what I have to deal
with involves just some sort of KV store with a very simple way to index 
(Riak calls this secondary index) the data. Furthermore, I host my own apps and
they are usually hosted on low end boxes with minimal amount of RAM and CPU, so
I want everything to be as lightweight as possible. 

Don't get me wrong, Riak solves most of my problems very well. It offers a KV
store, a secondary index, and it persists to disk (sorry, redis fans). However,
it is not lightweight. Riak is designed to scale to clusters of machines. It
is designed to perform better when you have more physical nodes. My side
projects usually have little to no users, so Riak's scaling features are 
unnecessary for me. 

In fact, I've performed some benchmarks:

    Test performed on a Vagrant box with 512MB of RAM, single core Intel Core
    i5 2410M processor. Disk is SanDisk Extreme SSD 240GB. Riak version is
    1.3.1 2013-04-03 Debian x86_64. Operation system is Debian Wheezy.

    All tests done without indexes or links, document is 8000 randomly generated
    characters.

    All tests done using Riak Python Client

    Average Insertion Speed:
      HTTP: 9 seconds / 1000 documents
      PBC: 4.5 seconds / 1000 documents
    Average Fetch Speed:
      HTTP: 6 seconds / 1000 documents
      PBC: 3 seconds / 1000 documents
    Average Deletion Speed:
      HTTP: 14 seconds / 1000 documents
      PBC: 9.5 seconds / 1000 documents

These are pretty abysmal speeds, even if you are using the PBC interface.
I for one tried to import the StackOverflow data dump into Riak at one point to do
map reduce. Needless to say I gave up that attempt pretty quickly.

It is fairly easy to see why I am getting poor performance with Riak by using
it this way. For one, I do not have the optimal settings for my setup. I am also
using Riak in a way that it is not designed for.

At this point, I began searching for alternatives. Leveldb seemed to be quite a
neat library, so I began writing an app with it (more on that later). I was 
using a version of [riakkit](https://github.com/shuhaowu/riakkit) 
(that will be pushed out later, too) for my app and I ported my application to
leveldb by simply writing [leveldbkit](https://github.com/shuhaowu/leveldbkit).

So far so good. Leveldb handled in process concurrency really nicely. I was 
able to implement a crude secondary index with leveldbkit. The only issue I 
encountered was that I cannot use the auto reloader for my application server
as it didn't properly cleanup the leveldb instance, which resulted in some lock
errors. Manually reloading the application server was annoying, but nothing 
insurmountable.

At this point I got a little bit bored with Leveldb and how little it offers,
I was also a little bit annoyed with the fact that only one process could access
the database at a time. I was planning to write an app with two application 
servers but share a common database.

Then I went ahead and wrote [levelupdb](https://github.com/shuhaowu/levelupdb).
Levelupdb is a database that clones the Riak API but it is designed for low end
boxes. This means it needs to be pretty fast and relatively lightweight. I wrote
it using Go and it is currently in a semi functional state (no riak links, no 
map reduce). It does not (and will not) scale to multinodes (although you could
have multiple processes connecting to the same database now as levelupdb acts
essentially as a governor for the underlying leveldbs).

When you clone an API, everything that is written for that API can be 
automatically used for your thing as well. Since levelupdb cloned the Riak API 
(with the exception of a map reduce interface at the moment, although I will
look into the JS M/R integration soon), all existing clients for Riak should
work with levelupdb as well (I use the python client to test). Applications built
on those clients will work as well, as long as if they do not invoke something
explicitly not included in levelupdb, such as the erlang map reduce. Similarly, 
if you use levelupdb as your side projects' database and if your side project
hits the front page of HN, you can easily spin up a couple virtual machines 
and port your entire application over to Riak without changing a single line of 
code while adding features such as fault tolerance, always available, and replication.

Right now levelupdb is still in its infancy. Hopefully in the coming weeks I
could make it feature complete (completely compatible with Riak) and optimize
where necessary. At that point, I'll try to blog again (and hopefully without 
exams this time!). 
