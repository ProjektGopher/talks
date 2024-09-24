Hi everyone, my name is Len Woodward, online I go by the handle ProjektGopher, and I've been writing code since about 2003.

I haven't been _good_ at writing code since 2003... I'd say I got properly good maybe about five years ago.

What I'm gonna try to show you today is the possibilities that open up when we start using the `ReactPHP` event loop in our terminal commands.

We're not going to be using it _directly_ inside of our artisan commands though, we're going to be building our own components for Laravel/Prompts using a custom fork that I'll explain in a bit, but first I think it's going to be important for us to define some terms just so that we're on the same page.

By the way, if I get any of this wrong, I'd love for any of you to correct me. This is all from my _current_ understanding, and it's entirely possible that I've misunderstood any number of these things.

So the first term is:
`blocking` or `synchronous`:
At it's simplest, this is just an action that we're taking where we're sitting around waiting for the action to complete, or return, and we can't really do anything else while we're doing that.

The next term is:
`non-blocking` or `asynchronous`... kind of


## event loop
## debouncing

This is one that I'm pretty confident you're probably all familiar with. It's just triggering an action on a queue, and delaying that actions execution until a certain timeout.

---

What problem are we trying to solve

discuss what blocking and non-blocking are
	waiting for something in the buffer, or returning false
	
# Http calls
Scout Search
	blocking, explain the network, how is this working
	non-blocking, can get rate limited, can overwhelm the network
	* what is debouncing *
		* we wait to send off a request until the user indicates that they're actually ready to send it. "ima let you finish before I act on this."
	debounce, why this can't work in blocking mode

# Streamed output
ai thing
	blocking streamed output, have to wait for everything before acting
	"is this thing working?" could use a spinner to indicate that stuff is still happening.
	can now render one chunk at a time
	extra credit- does not block user input

# WebSockets
real time chat
