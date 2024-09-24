{{ set browser to LenWoodward_com.test }}
Hi everyone, my name is Len Woodward, online I go by the handle ProjektGopher, and I've been writing code since about 2003.

I haven't been _good_ at writing code since 2003... I'd say I got properly good maybe about five years ago.

I'm currently starting a two-person agency with my friend Ed Grosvenor, named Artisan Build,
{{ set browser to https://artisan.build }}
so if you or anyone you know has any **tough** problems to solve, then let us know. We'd love to partner up on some interesting projects. We're also announcing a really cool initiative soon, so keep an eye out for that. 

What I'm gonna try to show you **today** is what possibilities open up when we start using the `ReactPHP` event loop in our terminal commands.

Now we're not going to be using it _directly_ inside of our artisan commands though, we're going to be building our own components for `Laravel/Prompts` using a custom fork that I'll explain in a bit, but first I think it's going to be important for us to define some terms just so that we're on the same page.

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

Let's start with the easiest problem in `Laravel/Prompts`, which is the 'looping mechanism'.

{{ set VSC to `Prompt::class` }}
Every prompt extends this base `abstract Prompt` class, and if we look through the `prompt()` method we see some pretty standard looking setup steps: capturing newlines, setting fallbacks, checking whether we're in an interactive mode, hiding the cursor, doing an initial render, and then we're registering a nice fat callback to handle our keypresses inside of a call to our **looping mechanism**.

{{ scroll VSC to `runLoop()` }}
In our `runLoop()` method, we're looping over `terminal.read` until it returns `null`. The fun part is that the `terminal.read` method is literally incapable of returning `null`, so it's just going to continue looping until our callback returns an instance of `Result`.

Now the question I had here is why isn't the cpu permanently pegged at 100% if we're just running this `while` loop **constantly**? Well, if we look at the implementation,
{{ click through to `Terminal::read()` }}
we see that we're just calling php's `fread()` function. This is a `blocking` function. It doesn't `return` until something enters the `buffer`. This means that if there's been no keys pressed, the function just waits to return until a key _has_ been pressed. If a key (or multiple keys) _have_ been pressed while the cpu is doing other things, these keypresses are buffered, and they get returned all at once the next time this function is called. This is why the cpu isn't running at 100%: it's literally just sitting there waiting for keypresses. This fundamental limitation means that we can't do _anything_ while the terminal is waiting for input from `STD_IN`.

So how do we fix this?...
{{ close `Terminal::class`, open `AsyncPrompt::runLoop()` }}
Now we're in my `Laravel/Prompts` fork - this `abstract AsyncPrompt` class `extends Prompt`, and the beautiful thing here that I'm super proud of is that all we have to do is `#[Override]` the `runLoop()` method with our own implementation. As an aside, the PR I submitted that makes this possible is actually one of the **only** PRs I've made to any Laravel product where Taylor merged it with **zero** changes. Like, not even stylistic ones. But let's take a second to look at what I'm doing here.

I'm setting a `$result` variable, that can only be `null` or an instance of `Result`. This is just a 'sentinel value'. It let's us differentiate between `null` as a return value, and `null` as an indication that we should continue looping. Then I'm assigning a `static $stdin` variable. I'm doing this statically for some testing stuff - we don't really care about that right now. But I'll touch on it later if we have time. The important part is that we're setting it to this `ReadableResourceStream` class that we get from the ReactPHP `streams` package, and we're passing it our `STDIN` stream. This gives us back an object on which we can register some listeners (among other things). We're going to register a listener on the `data` event, which gets passed everything that's in the `STDIN` buffer. We're going to pass in our **fat** 'keypress handling' callback, and we're going to pass `$result` by reference instead of by value because we need to mutate it within this listener, and have that persist so that we can use it later. Once we get a proper result from our callback, our listener explicitly **stops** the event loop.

Now that the listener is defined and registered, we start the `Loop`, and wait until it's stopped by the listener. Now that we've got a return value for our `Prompt`, we remove the listeners, do some checks, and return the `Result`.

Great! We're reading from `STDIN` on the event loop, and we don't do any work until we recieve a `data` event. Perfect. But how is this `ReadableResourceStream` doing this in a way that doesn't block the main process? Well let's take a look.
{{ click through to `ReadableResourceStream:class` }}
We've got some private properties, then in the `constructor` we take the `$stream`, check that it's a `resource`, check that our `$stream` is open in **read mode**, and then here's where the _magic_ happens: **we set the stream to 'non-blocking' mode**. From what I've been able to determine, what this does is makes php's `fread()` function just return `false` if there's nothing in the buffer. There's no waiting around for input. After that we assign some value to properties, then we call the `resume()` method, {{ scroll down }} which just adds our `read stream` to the event loop along with a `handler` and sets `$listening` to `true`.

If we look at the handler
{{ scroll to `handleData` method }}
we set some error handling stuff, then we get the `contents` of our `stream`. We do some more error handling stuff, then if the `contents` of the `stream` isn't an `end of file` or an empty string it `emits` the `data event` that we were listening for in our `runLoop()` method.
{{ close `ReadableResourceStream::class` }}

Alright!
So
We're not being blocked by waiting for keypresses on `STDIN`.
Great! That's awesome! But what does that get us?
Well I think now I can finally show you all some examples
---

{{ window > DocsSearchCommand (echo-terminal) }}
	
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
