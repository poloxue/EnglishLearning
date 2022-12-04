Subtitle From: [An Introduction to "go tool trace"](https://www.youtube.com/watch?v=V74JnrGTwKA)
Article From: [An Introduction to "go tool trace"](https://about.sourcegraph.com/blog/go/an-introduction-to-go-tool-trace-rhys-hiltner)

The tool that I'd like to share with you today is goes execution tracer, the command you use to run that is go tool trace to visualize the output. But before we get too far into that let's talk about `golang`, More specifically let's talk about go, the keyword. 

The go keyword is what lets us get concurrency and the low overhead. With which we can create new goroutines and maintain our existing goroutines, allows us to use goroutines and concurrency as a tool to solve problems without really thinking twice about it. In the same way that you might use a closure to solve a programming problem when it seems natural. so the low overhead of goroutines and their concurrency means that we often end up solving problems with concurrency, which means we often have problems with concurrency. When solving problems with concurrency, we may want to use the execution tracer to really see what's going on and so this is kind of a special tool tailored for gos needs because concurrency is so such a natural part of writing go programs. It has more of a need for a tool like this than many other languages. CPU profiling and memory profiling are fairly common, and you might have used them in other languages before, but you may not have seen a tool like this before starting on go. 

It's also for chromium's needs. A lot of UI was was made by chromium for diagnosing latency issues in rendering the browser UI and it's been repurposed for visualizing the code that comes out of the go runtime. The goroutine scheduling as I said earlier is very low overhead, that low overhead is key to the way that we know how to write go programs. This is instrumentation that's built into the runtime that allows us to see the decisions that are made by the goroutine scheduler and so it's a fairly special tool. I don't know that would be possible to write outside of modifying to go over to the the runtime to get this data. So this is a tool that can help you with concurrency and how to manage it and with parallelism and how to exploit it to make sure that your programs are running as as quickly as they possibly can. 

You might start looking at the documentation for the tool here in the runtime trace package, it describes a bit about what it does and points you to the go tool trace command. The doc for that commands are also a kind of brief, it says you run this command, then it opens a browser.

Great! Run command it says I don't know how import your trace for you. This is such a like polite and obedient tool. It's going to be fun to work with, but then you end up with something like this. It's clear that there's a lot going on there. 

```bash
*------------------------*
|                        |
|    this is a picture   |
|                        |
*------------------------*
```

Who knows? 

We're going to get into that today and into some of the other ways that you can look at the data that comes out of the runtime and understand what your programs are doing.

I'm going to do this through three demos, these are all from real twitch programs that solving problems that we
encountered in production.  

- The first demo is on diagnosing a timely dependent bug. 
- For the second one I want to make it clear what it doesn't show. This won't replace all the go tools that you use. This will complement them and allow you to look at your programs and understand what they're doing in a different way. 
- Finally, I want to talk about latency during garbage collection, that might look like in your applications where it might pop up

First of all, a race condition, data races we run our tests with the race detector, we might build race instrumented binaries and run them as Canaries in production we got a data race. We have the full stack traces for for the unsafe memory access, where like one goroutine was writing and then the other one was reading, which can lead to memory corruption and crashing. But that's not at all what went wrong here, this is about a logical race. There was synchronization in place with this program, but the synchronization was not done correctly and so although there wasn't memory corruption. The program didn't behave as we expected. This is a bug about gRPC, HTTP 2 and flow control. 

GRPC is a remote procedure call framework that allows you to make very low overhead requests to other micro services. One of the ways that it does that it has that little overhead is it uses HTTP2 to send many concurrent requests on just a few TCP connections. So they're kind of multiplexed on there. You don't have to create a new connection and do a TLS handshake, all that stuff in order to make a single request. 

Flow control is what makes sure that the receiver is ready to get the data. The percentage doesn't just like barf a bunch of bytes at the recipient and overload its buffers. So flow control allows a receiver of this data to put back pressure on the sender. 

Here is what the execution trace for that program looks like when it's encountering the bug. 

Just in very broad strokes, over here stuff is happening, over there, stuff is not happening, not good. This is for a program the first server that would send very large and very slow responses to
its clients and it was fairly typical for the clients to say and I don't need
that data like please stop sending it and to issue another request so I
borrowed this down into a small reproducible test case ran it with go test
has a - trace flag which allows you to specify a file to write the execution
trace data - and so you might have done this with CPU profile as your memory profile as memory profiles for blocking
profiles for other programs and so this is this is another flag in that same
vein although the data that's stored here is very different so now that I have that file I pass it to go to all
trace and sure enough we get the web UI and so there are a bunch of different
types of data here different type different ways of visualizing this data
and I'm not going to go to all of them but I'm going to go to one of each style because they're only through looking at
all of them is it very clear what's going on with this bug so first of all if you trace this is the UI that I
showed a few moments ago and there's this one-second delay there's a time
line listed across the top time goes from left to right across the screen and so there's this one-second delay and the
fact that there's a delay should be surprising because this is a reducer that is making many arca sees and
canceling them and so it's surprising that it stopped but this the the fact
the pause that we see is one second long shouldn't be surprising because that's what I passed to context with time out
when creating the test and so you can press the question mark key and get a
list of help help text for keyboard shortcuts the most important ones to
start off with our wsad you can kind of navigate around with those keys like a
videogame there's plenty of other stuff here hours of entertainment for your board some afternoon so now that we know
how to zoom in we can look at this 20 millisecond segment going from the top
down there's ribbon for the size of the heap there are ribbons for the number of go
routines that are currently running and that are runnable and the number of OS threads that are running them there are
there's an indication of stimuli coming into the process of network data coming in over the
network just calls finishing timers firing things that can that come kind of from outside the go routines and that
can make them do stuff garbage collection is called out pretty clearly here and then these here are go routines
that are running on two OS threads and that that scheduling of hundreds of
thousands of go routines may be onto a very small number of less threads is fairly key to having the low overhead
that we haven't go for starting go routines for maintaining good routines
so next up is the the synchronous blocking profile and if you've used
others CPU profiles memory profiles you've maybe seen a UI that looks kind
of like this these are call stacks that were active when some cost was incurred
and so that costs for the other profiles might be bytes of memory allocated or number of CPU cycles spent but the thing
that we're spending here is seconds that we're spent waiting so we can zoom into lower right and there's this one point
zero one second segment which is about one second of unexpected delay we've
seen that number before going up to the top we have kind of the goroutine name
at the top of the stack this will show up in a couple other places but we can more or less refer to this go routine as
surf streams func one one okay so go routine analysis is next this is a list
of all the different types of go routines that were running in that program during during those few seconds
that I was recording and listed at the very top there are 10,000 different go routines that all at some point that
we're all started in order to run surf streams bump 1 1 so we can click that
one and get this giant table of each of those 10,000 go routines of how much
time they spent in in different states and so we see synchronous blocking time
here at the top there's a particular go routine that spent just a hair over 1 and in synchronous blocking time their
routine 8006 so we can click on that and then we get another execution trace UI
like this that is only that one go routine and the other girl routines that
it interacted with during its lifetime and so on the left there's a teeny a
little bit that it runs for and on the right there's a teeny bit that it runs for and then there's this giant one second gap in between so we can zoom in
on the one at the beginning and if you click on it then you get the ending stack trace of what that go routine that
was doing that made it to stop and so we see that the stack trace here matches
the one that we saw in the SVG earlier where there's a function called weight that calls select go and then it waits
the the body of that function is one giant select statement it takes a
context it takes a bunch of channels and then it selects waiting to see which one of them is ready first and so that might
be the context timing out that might be the stream closing or there's also this
proceed channel which is the only channel that returns in actual valuing we might use it returns an integer and
in all the other cases we return an error but if proceed triggers then we
return whatever int it gave us and no air wait is called from here in Senko to
pool sorry wait it's called from here we see that there's this thing called 10 code pool which is which shows up a few times
first of all we add we call add on that with zero then we wait and then maybe we
cancel or maybe we add some value back to it and so this is this is flow
control in action here in this program of figuring out how many bytes the
recipient is ready to get figuring out how much of payload we can send and then
deducting that amount from the existing quota and putting the quota back in the pool for for other go routines and
be reusing the same transport a TCP connection the quota pool looks like
this there's a channel of integers and a field that's an integer and so which is
protected by by a mutex and when we create it we see that the the integer
channel has capacity 1 and so it might be empty or it might contain a single integer and we see what looks like might
be the beginnings of a helpful invariant here where if the initial quota is positive that value lives on the channel
and in any other cases if the quota is 0 or 4 the quota is negative then its
value is put on into the field and so you could think of it as if if there is
a value in the channel we could sum that value and the value of the quota field and that we then we would get the total
positive or negative quota value of of that struct so here it did not have
these these annotations at the time but that's that's more or less what we're setting up for the acquire function
pretty trivial just returns the channel great the add function is kind of
interesting it's protected by the lock it
it does arithmetic to figure out how much quota is left over and then it
attempts to send that quota on the channel and if it succeeds then it zeros
it out so that the code is conserved the cancel function is pretty weird though
it pretty much explicitly breaks the invariant that that we might hope was
set up by receiving any value that might be on the channel and if it receives
anything then then adds it adds it to its its local quota storage but when
this function returns the channel is definitely empty and so we might have a
go routine the needs quota that like this first of all it calls ad with with argument zero and what this does is
it tries to put something on the channel to make that channel ready to receive
then it calls wait receive them from that channel hopefully and then later on
it would call ad to like put whatever unused quota back into the pool or we
might have to go routines running more or less concurrently one of them adds
triggering the channel and then receives from the channel and then maybe it encountered an error and so it cancels
and so the channel is now empty the other go routine triggers the channel successfully receives from the channel
because it had just triggered it and then adds quota back to the pool or they might be overlapping a bit more and so
one girl routine could trigger the quota pool another girl routine would attempt tree or it's already triggered whatever
and then first go routine tries to grab some quota calculations puts the rest
back and then the second girl routine grabs the quota from the pool does its
calculations puts it back this is all this is all working code great but if we
look at at our particular go routine the one that's stalled in context we see
that that here's here's our go routine 8006 which is about to stall for one
second and overlapping completely with that run is another guru team that runs
the same function and so might plausibly be running that same code and they
overlap by 36 microseconds 36 microseconds is a tiny tiny tiny amount
of time if you're using if you're used to using CPU profilers by a default one
and go we'll give you a hundred samples for every second of CPU spent but this
is a much much smaller timescales and is pretty much all in the day's work for
this tool so we might hypothesize the
the overlap is like this where they both a temperature pool it is triggered great
one of them attempts to get quota but encounters some sort of error and so
that might be the client saying I don't want your data close the stream please don't send me anymore and in that case it calls cancel and
cancel empties the pool then the other guru team tries to wait and just waits
forever because nothing is going to come along to to trigger that and so then at
some point that request times out and it also cancels but this is maybe the the
source of the stalls that we see and so the cancel function is basically toxic
and we just need to delete that entire thing and never call it and the add function we can change so that it
maintains the invariant that we were hoping for and so with the modifications
the add function will always will always clear the channel always come up always
calculate the total amount of quota available and then if it is positive always put it back onto the channel so
that if so that if there is any quota available that it is ready to receive so
with with that with that code change then flow control networks and our
application is able to move on so I want to make it clear though that this is
just one tool and if you have questions about when a go routine that starts or
when it stops or if you have questions about why that goroutines started or what or why that go routines stopped
then this tool has that information if you have questions about anything else you'll probably need a completely
different tool and so it's a 4/4 that very narrow set of questions about go
programs um this is a wonderful tool to
loop back for a moment there are three ways to get data out of your programs and in first example I showed one from
go test where you can pass the fly idea and and specify a file
so you can also do this for for CPU profiles memory profiles etc you can use
the runtime trace package directly it takes an i/o writer you tell it when to start when to stop
fairly similar to the way that you might directly get a CPU profile and also if
you import the net HTTP PDF package then you'll have a handler installed
automatically and so you might right now be running applications that are ready to have to give this data to you and
it's just waiting for you to make that HTTP request to get it and to look at so
this is an execution trace from another program that I looked into recently and
there are few things that kind of pop out to me here first of all there are these suspicious gaps every 5 seconds it
looks like nothing comes in over the network and no sis calls return and no
timers fire during that period which is pretty odd because they happen all the
other times and we see the garbage collection happens it's pretty clear
that like there's a lot of activity during that time and also there's this
other type of goroutine that looks pretty much unlike any of the others that might be interesting to look into
so first of all zooming into one of the sections that one of the quiet periods
where nothing really happened there's this single go routine that runs in
parallel with nothing else for 90 milliseconds uninterrupted and so looking at the code that this go routine
runs it's a call to runtime read mem stats and because because we have this
particular server is running on go one point eight or any earlier version and
because it has an enormous heap it's 40 gigabytes for this one we get these long pauses the implementation of runtime
read mem stats had for a long time just looked at basically all the memory of
the program in order to give its answers as to how much memory use it use this is fixed and going
you probably just have to upgrade if this bug affects you looking at the
other kind of go routine what does that one do how does it spend its CPU time can it possibly run more efficiently
like is it using its CPU cycles well and can we maybe paralyze it to make it
finished sooner those are interesting questions they're not about goroutines
starting or stopping or why go routines started or stopped and so this tool does not have those answers because it runs
for for a couple seconds at a time and and runs about half of the time this
would be pretty easy to spot on a CPU profile so we'll use that tool and probably get answers for that looking at
the other type of go routine that usually runs in this program is from net
HTTP to serve incoming HTTP requests so we might have questions on this one like
what was the request to they've served which actual like what piece of code ran
in response to that request and did we were we able to give the client a good
response to the query that they sent us and again the execution tracer does not
have these answers it might be interesting if we were able to annotate
execution traces with like hey I returned a 200 or hey this was a request
for like slash or it might be interesting to have to have a particular
key that we can that we can use to match up the logs that we have from our
application and the logs did we get from here to correlate them and see what was on what was going on but we currently
don't have that capability the the final
final section that I want to discuss is about garbage collection those garbage
collector is still improving it has come so far but it is not yet perfect and so
here's execution trace from one of our applications during garbage collection
this is a long-running server I may disappear Qwest to the endpoint got some data and
so we see during the garbage collection the number of go routines that are ready to run increases it doesn't increase
with that bound it decreases sometimes but it's still very different than very
different than the surrounding times there's also some patchiness in the inbound network and so when the garbage
collection is not active we generally have Network requests like data coming
over that we're basically all the time but then it looks like the clients just
went away and I find it hard to believe that the clients would all figure that out and all colludes not send data to
this program and so there's something fishy going on here probably with with
garbage collection and the effect that it has on our programs and so we see
that some behavior is clearly different during GC it's probably bad that it's
different but it's unclear also how how perfect we should expect the GC to be and how and clearly we'll have to have
some overhead for automatic memory management but if not clear how and when
we should pay that cost in our programs so to give a history of brief history of
garbage collection ago starting in go one the programs would completely stop
and like do nothing else like garbage collection would happen and then at the
end of garbage collection the program would resume and so any requests that were outstanding for that time would
just have no interaction until GC was complete in go 1.1 the
garbage collector started using parallel threads and so it had been that there was a single thread that would look
through the entire heap all by itself but one point one changed that so the positives got faster in go one point for
the garbage collector became precise and so you could no longer trick it into retaining extra memory by having a large
integer presence somewhere in your program it would know that that's not a pointer and that it shouldn't retain
that in 1.5 huge changing the garbage collector it now allows the program to
run concurrently instead of forbidding it from running until the GC is completely done and the runtime team
announced that their goals for stop the world plazas where nothing was making progress that their goal for that was to
have those pauses be less than 10 milliseconds and through incremental
improvements in in several releases since in bill 1.8 the the goal is now a
100 microsecond causes and those are positives not of the entire application but of individual goroutines the
individual go routines shouldn't be delayed more than necessary more than 100 microseconds more than necessary so
that's a much more beneficial garbage collection latency goal for us as as
engineers and users of this runtime visually go on garbage collection looks
like this the user code the mutator is running on the left in the right in the
middle in between those times garbage collection happens and the entire program stopped for that time
1.12 stop is shorter in 1.5 we see that
some cpu is dedicated to the garbage collector but that a lot of it is available to run our program
concurrently and that the world is only stopped for for those two little slices
on either end and then in go on point 8 those slices are even smaller so as far
as what what these sorts of pauses look like in the execution tracer and things that you can do to make your programs
behave poorly first of all stop the world pauses when the GC begins and ends
the mark phase it has to do some bookkeeping and in order to do that bookkeeping it needs all user codes to
not be doing anything and so it has to stop all of those for a moment but
bringing all of the go routines to a stop take some time and so here's one of
our programs that did not do that very well we see that the garbage collection
begins at that but then there are these three straggler go routines that just keep doing
whatever it is they're doing that they didn't get the message that they should stop and the GC only really begins doing
work once they're all done and there's a three point three point six millisecond
delay in this particular example between between nearly all the go routines
stopping their work and that end them all being able to resume at the end and
so we see that there's this unused CPU during that time and so this is running on a 36 course server and so while so
there are 33 cores that are sitting idle while those last three go routines finish their work this this machine
isn't doing any other work other than just waiting for those and so that's wasted CPU and so go routines are able
to stop the the runtime is able to to preempt them more or less at particular
points where where they check in and so that might be when they're allocating memory when they're calling most
functions or when they're communicating maybe by sending or receiving from channels or grabbing a lock but what is
not on this list is crunching numbers in a loop and so if you crunch numbers in the loop for a long long time you're
probably going to have this problem you if you use encoding base64 or encoding
Jason or protobuf you might see this problem you might see it with one
megabyte chunks one may get up one megabyte is a bunch um but this is exactly what was going on in that
program it was using those packages with that size and seeing seeing this reduced
throughput because of it maybe it affects you maybe it matters
you can seek it out if you like but if you don't already know it's a problem
maybe maybe just wait you can if you click on the go routines
that are laid you can see the end stacktrace of the preemption point that they finally got to and if you think back a
few microseconds to what code those were probably running right before that point if you find yourself
looking at a numeric loop then you might have this problem it doesn't come across
very well on a slide I'm sorry you can also write your own type loop this one counts up to a billion it will mess with
your GC don't put that code there's an issue open for this it's been around for
a while the go 110 compiler should have a general permanent fix and so if you
don't already know that this affects you you can probably just wait and your programs will get better there is a
workaround available now in both go 1 8 + + 1 9 I wouldn't recommend enabling it
unless you know that you have this problem or are interested in debugging it and so with that with that a little
flag enabled with the same program same go version we knew we now see that the
pauses are very quick and are wasting less CPU so other other ways that the
garbage collector can make your program have awkward pauses so to recap a bit go has a mark sweep
garbage collector first of all the mark phase finds all the in use memory and
then sweep reclaims all of the other memories that wasn't marked the garbage
collector needs to make progress this was very easy in older versions of go where all the user code would stop the
GC would make progress until it was done and then it would resume everything else but because it's allowing user go
routines to run at the same time it needs to know that the it will finish
the garbage collection before before the application allocates too much memory and gets itself killed by the operating
system user code is working in direct opposition to this the garbage collector
is trying to find all the reachable memory the user code is trying to make there be more reachable memory and so
these are kind of bad odds but the garbage collector is part of the runtime and the runtime can make things
do whatever it wants and so it can force the user code to help out and so to zoom
in on a few lines of you shan't race for what that looks like there's on each of these ribbons there's
a top half in the bottom half and the top half is user code running in goroutines the bottom half is work that it's doing
on on behalf of the runtime which comes in a couple flavors first of all you may
not be able to see it yet but when I put a box around it you might see that there's this tiny little like half pixel
wide sliver that is a reed syscall that that began you can try clicking on those
little half pixel wide things you can also hold shift and draw a selection box it's much easier I was very happy when I
learned that and there's also a mark assist and and also sweep that shows up
as longer longer spans because there's work to be done and so that's what it looks like when you're go routines are
helping the garbage collector and so in at this particular point here's an
assists that ran for 4.4 milliseconds that's that's a lot it's there are also
a bunch of much much faster assistance bands on here that I didn't highlight
and so many assists are very very fast some of them are kind of slow here's one
that ran for 15 milliseconds and it didn't finish and it didn't it didn't even start then it's resuming one that
it didn't finish earlier and so that go routine probably allocated like 10 megabytes of memory and so it you know
deserved what I got most of these most of the assists are pretty well deserved
and it works pretty well but they also start suddenly and so you might have a program that has a ton of available CPU
overhead that you're handling just a few requests per second you might over provision CPU but then as soon as
garbage collection hits you immediately have to start paying the price of assists and those can be pretty
expensive so sweeping requires assists
as well that when right after garbage collection finishes something has to
find the the memory that wasn't marked and that might be your go routines if they do it soon enough so you might
see those as well and so I guess my advice here is to not allocate in critical paths but that can be really
convenient and tempting and maybe shouldn't be punished so hard but I hope
I hope you know you now know what that looks like and can see if that's affecting you so please don't take my
word for it the the command you're looking for is go tool trace and you
might be able to get your data from your programs right now by by making a request to debug paper off trace to grab
that data to recap how it can help you
it can help you to see timing-dependent issues in your code it complements the
other profiles it doesn't replace them and so you're still going to use see CPU profiles you're still gonna use memory
profile profiles but if you have a problem that looks like it's related to latency then once you've checked those
other those other tools then the execution tracer can be a very helpful one for that and so it can help you find
latency improvements there maybe in the garbage collector maybe in your own code
where it's not acting exactly as you thought when you were writing it and so
my advice is to be prepared and to practice using the tools you probably
have a bunch of free time now and and your sites are not on fire and so this
this is a very good opportunity to take some time and experiment with the tools and and know what your code looks like
when things are going well so you can understand it better once when it's not working well 

Thank you very much! That's what I got you.