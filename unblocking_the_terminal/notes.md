Hi everyone, my name is Len Woodward, online I go by the handle ProjektGopher, and I've been writing code since about 2003.

I haven't been _good_ at writing code since 2003... I'd say I got properly good maybe about five years ago.

I'm currently starting a two-person agency with my friend Ed Grosvenor, named Artisan Build, so if you or anyone you know has any **tough** problems to solve, then let us know. We'd love to partner up on some interesting projects. We're also announcing a really cool initiative soon, so keep an eye out for that. 

What I'm gonna try to show you **today** is what possibilities open up when we start using the `ReactPHP` event loop in our terminal commands.

Now we're not going to be using it _directly_ inside of our artisan commands though, we're going to be building our own components for Laravel/Prompts using a custom fork that I'll explain in a bit, but first I think it's going to be important for us to define some terms just so that we're on the same page.

By the way, if I get any of this wrong, I'd love for any of you to correct me. This is all from my _current_ understanding, and it's entirely possible that I've misunderstood any number of these things.

So the first term is: `blocking` or `synchronous`:
At it's simplest, this is just an action that we're taking where we're sitting around waiting for the action to complete, and we can't really do anything else while we're waiting.

The next terms are: `non-blocking` or `asynchronous`
This is basically an action that doesn't wait around. These can be callbacks that run when a listener is notified of a future event, or an action that returns a promise that can be resolved later on, or it can even just be an action that returns immediately unless it's got something to deliver.

Next is the `event loop`:
This is just a fancy `while` loop. It's going to keep on running until it's explicitly stopped, or until it has no more listeners registered.

And the last term is: `debouncing`
This is one that I'm pretty confident you're probably all familiar with. It's just triggering an action on a queue, and delaying that actions execution until a certain timeout.

---

So. What problems are we trying to solve, and what tools are we going to use to solve them?

Let's start with the easiest problem in Laravel/Prompts, which is the looping mechanism, and work through the solution.

Every prompt extends this base `abstract Prompt` class, and if we look through the `prompt()` method we see some pretty standing looking setup steps: capturing newlines, setting fallbacks, checking whether we're in an interactive mode, hiding the cursor, doing an initial render, and then we're registering a nice fat callback to handle our keypresses inside of a call to our **looping mechanism**.

In our `runLoop()` method, we're looping over `terminal.read` until it returns `null`. The fun part is that the `terminal.read` method is literally incapable of returning `null`, so it's just going to continue looping until our callback returns an instance of `Result`.

Now the question I had here is why isn't the cpu permanently pegged at 100% if we're just running this `while` loop **constantly**? Well, if we look at the implementation,
{{ click through to `Terminal::read()` }}
we see that we're just calling php's `fread()` function. This is a `blocking` function. It doesn't `return` until something enters the `buffer`. This means that if there's been no keys pressed, the function just waits to return until a key _has_ been pressed. If a key (or multiple keys) _have_ been pressed while the cpu is doing other things, these keypresses are buffered, and they all get returned at once the next time this function is called. This is why the cpu isn't running at 100%: it's literally just sitting there waiting for keypresses. This fundamental limitation means that we can't do _anything_ while the terminal is waiting for input from `STD_IN`.

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
