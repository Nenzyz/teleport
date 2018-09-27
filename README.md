Original is [here](https://vagabond.github.io/programming/2015/03/31/chasing-distributed-erlang)

# Chasing distributed Erlang

_31 March 2015_

So, the other week, someone in #erlounge linked to an interesting [Reddit post](http://www.reddit.com/r/golang/comments/2y5nc0/fault_tolerance_in_go/cp7m29l) by someone switching from Erlang to Go.

I actually strongly disagree with almost everything he says, but the really interesting part of the thread is when he starts talking about sending 10Mb messages around and the fact that that ‘breaks’ the cluster. Other commentators on the thread rightly point out that this is terrible for the heartbeats that distributed Erlang uses to maintain cluster connectivity and that you shouldn’t send large objects like that around.

And this is where I started thinking. In the Erlang community this is a known problem, but why isn’t there a general purpose solution? Riak’s handoff uses dedicated TCP connections to do handoff, but when reconciling siblings on a GET/PUT? Riak uses disterl for that (this is one of the reasons that Riak recommends against large objects).

So, even Riak is doing what ‘everyone knows’ not to do. Why isn’t there a library for that? I asked myself this one night at 2am before a flight to SFO the next morning, and could not come up with an answer. So, I did the logical thing; I turned my caremad into a prototype library.

After some Andy Gross style airplane-hacking, I had a basic prototype that would, on demand, stand up a pool of TCP connections to another node (using the same connection semantics as disterl) and then dispatch Erlang messages over those pipes to the appropriate node. I even implemented a drop-in replacement for gen_server:call() (although the return message came back over disterl).

The only problem? It was slow. Horrendously slow.

My first guess was that my naive gen_tcp:send(Socket, term_to_binary(Message)) was generating a giant, off-heap and quickly unreferenced binary (and it is). So, I looked at how disterl does it. A bunch of gnarly C later, I had a BIF of my own: [erlang:send_term/2](https://gist.github.com/Vagabond/efb0c1563ef7b94b3b27)

This, amazingly, worked, but with large messages (30+MB) I ended up causing [scheduler collapse](http://erlang.org/pipermail/erlang-bugs/2013-May/003529.html) because my BIF doesn’t yield back to the VM or increment reduction counts. I looked at adding that to the BIF and basically gave up.

So, I left it on the backburner for a couple weeks. When I came back, I had some fresh insights. The first was: what if we had a ‘term_to_iolist’ function that would preserve sharing? So I went off and implemented a [half-assed](https://github.com/Vagabond/teleport/blob/c785e40b03319dd1b8431423465233021c01d20c/src/teleport.erl#L83-L123) one in Erlang, that mainly tries to encode the common erlang types into the Erlang [external term format](http://erlang.org/doc/apps/erts/erl_ext_dist.html) but using iolists, not binaries (for those unfamiliar with Erlang, [iolists](http://prog21.dadgum.com/70.html) are often better when generating data to be written to files/sockets as they can preserve sharing of embedded binaries, along with other things). For all the ‘hard’ types, my code punts and calls term_to_binary and chops off the leading ‘131’ byte.

That worked, but performance was still miserable in my simple benchmark. I pondered this for a while, and realized my benchmark wasn’t fair to my library. Distributed Erlang has an advantage because it is set up by the VM automatically (fully connected clusters are the default in Erlang). My library, however, lazily initalizes pooled connections to other nodes. So I added a ‘prime’ phase to my test, where we send a tiny message around the cluster to ‘prime the pump’ and initialize all the needed communication channels.

This massively helped performance, and, in fact, my library was now in striking distance of disterl. However, I couldn’t beat it, which seemed odd since I had many TCP connections available, not just one. Again, after some thought, I realized that my benchmark was running a single sender on each node, and so there wasn’t really any opportunity for my extra sockets to get used. I reworked the benchmark to start several senders per node, and was able to leave disterl in the dust (with 6 or 8 workers, on an 8 core machine, I see a 30-40% improvement on sending 10Mb binary around a 6 node cluster and then ACKing the sender when the final node receives it).

After that, I thought I was done. However, under extreme load, my library would drop messages (but not TCP connections). This baffled me for quite a while until I figured out that the way my connection pools were initializing was racy. It turns out that I was relying on a registered Erlang supervisor process to be present to detect if the pool for connecting to a particular node. However, the fact that the registered supervisor was running doesn’t guarantee that all of the child processes are, and that is where I was running into trouble. Using a separate ETS table to track actually started pools fixed the race without impacting performance too much.

So, at this point, my library (called [teleport](https://github.com/Vagabond/teleport/)), provides distributed Erlang style semantics (mostly) over the top of tcp connection pools, without impacting the distributed Erlang connections and disrupting heartbeats. A ‘raw’ Erlang message like this:

    {myname, mynode@myhost} ! mymessage
becomes:

    teleport:send({myname, mynode@myhost}, mymessage)
And for gen_server:calls:

    gen_server:call(RemotePid, message)
becomes:

teleport:gs_call(RemotePid, message)
The other OTP style messages (gen_server:cast(), and the gen_fsm/gen_event messages) could also easily be supported. Right now, the reply to the gen_server:call() comes back over distributed Erlang’s channels, not over the teleport socket. This is something that probably should change (the Riak Get/Put use case would need it, for example). Another difference is that, because we’re using a pool of connections, the ordering of messages is not guaranteed at all. If you need ordered messages, this is probably not the library for you.

If you want to compare performance on your own machine, just run

    ./rebar3 ct
The common_test suite will stand up a 6 node cluster, start 6 workers on each, and have them all send a 10mb binary around the ‘ring’ so each node sees each binary. It does this for both disterl and for teleport and reports the individual times in microseconds, and the average time in seconds.

Finally, I’m not actually using this for anything, nor do I have any immediate plans to use it. I mostly did it to see if I could do it, and to see if such a library was possible to implement without too many compromises. Contributions of any kind are most welcome.
