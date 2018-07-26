# LISP: All is code

Lako / Jake thoughts

Jake: Possibly nutty idea, but first, a preface: I stumbled across your reference to LISP in the email chain you ccd me on and proceeded to binge some material regarding the language (from high level theoretical, benefits, legacy, paradigm shifting attributes -> mathematical basis -> a little about modern uses) and let it stew for a day or so and this is what came to me.

What if we structure the entire object model in the backend as a LISP data structure (you can do all sorts of fancy stuff to make it so it’s not literally all a monolithic thing: sharding etc).

One of the core ideas of LISP is that it is code that can modify itself, and that the difference between data and code breaks down... sound familiar (contract artifacts and code but also a thing in and of itself)? Parsing and changing of state becomes way more trivial. It can even be done functionally, viewing each node as a persistent data structure pointing to other persistent data structures. This leads to other ideas such on the implementation of networks built with Prose Objects + some sort of enforcement / consensus mechanism (maybe on something like enigma interfacing w/ arbitrary PoW and PoS blockchain or any other consensus systems). For example, each node could keep a cryptographically verifiable history of its own state + maybe some of its neighbors... but I’m getting off track, so I digress.

If the backend is all a LISP parseable data structure then allowing users to implement Turing Complete code, something I was stumped on, becomes way more arbitrary / easy (At least I think... I’ll have to ponder the specifics). This doesn’t change anything about the interface we’ve described although it does update the mental model a bit (but doesn’t have to... the OO mindset can still function on top and the users approach will be wildly affected by the interface). I read in one of the pieces that lisp can be seen as a more succinct XML (a language for representing hierarchical data) and this concept can be extended to graphs as well.

Of course, I’m also only 2 days into encountering the concept that is LISP so I have a lot more thinking to do... planning to read the little LISPer and maybe do a bit of playing around with the language to get a better idea.

Curious to hear your thoughts - will continue to think / refine / research further.

Side note: I also wonder if SSI (self sovereign identity) can be linked into the Prose Object Model concept but I am also aware that I’m open to scope creep bc I like to chase many threads at once... just a thought I’ll be keeping in mind. (edited)

Jake Martin [5:55 AM]
Not only that, but I would bet that lisp would be the easiest language in the world to interpret any other arbitrary language into b/c the space of possible syntaxes knows no bound as far as I can tell
So deva don’t have to write lisp to write contracts in Prose Object Model
Just need a language that has an interpreter to list built and well tested

Jake Martin [6:09 AM]
Of course non functional approaches (including languages) will likely disallow any formal verification of the parts of the graph that they touch
But mission critical subgraphs can be written in a probably secure way
And not need to worry about being tainted by non functional nodes. They are well within the constrain of (for all input, this graph / finite state machine will act this way).

Iako [8:01 AM]
Jake, this is a very productive line of thought.  It will indeed Fuse law and code.  Let’s find a way to do this in as public a way as possible so that we can bring others into the conversation - other students in the group, but also others with whom I’m in conversation.
We want broad perspective.  I think the way to do this will be iteration - and I know that the core mission - legal codification - can be done independently of the technical platform - or to put it another way, the git-based collaboration of repos and files is a perfectly _adequate_ way to achieve this, proven in software collaboration, well-tested, free, continuously improving and expanding (and Microsoft, which already owns the legal environment is buying Github).


Iako [8:12 AM]
I once played with seeing if I could get some LISP that I installed to natively expand strings, but stumbled somewhere.  Getting this to work would, I think, make utterly seamless the thought-process of law-is-code.   A critical design constraint to keep in mind is that (recursively)  i) the vast majority of activity will be “simple” transactions - a few variables with a link to an environment, ii)  the vast majority of non-simple transactions will require a few modifications, and the great majority of those modifications will be tweaks in Prose, iii) the vast majority of what requires more work will be reworking of legal forms and iv) the very few that still need work will be a) done by engineers and b) very likely involve tying into existing computer systems.
But let’s make this public.  As a first step, OK to move to #coding ?  Then maybe start it as a germ of a thought in the git repo?