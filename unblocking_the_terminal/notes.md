So, I only finished writing this talk about an hour ago, which means that you might see me staring uncomfortably at the prompter waiting for it to catch up, or I might speed up my speech if I'm falling behind. And this is literally my _first_ talk. So bear with me.
---

{{ set browser to LenWoodward_com.test }}
Hi everyone, my name is Len Woodward, online I go by the handle ProjektGopher, and I've been writing code since about 2003.

I haven't been _good_ at writing code since 2003... I'd say I got properly good maybe about five years ago.

{{ scroll down }}

I wrote a tool called `Whisky` for managing git hooks, but fully in PHP. It's currently got a little over 200 stars, and 30k downloads. I'm working on a tool called `Conductor` which is basically `npx` but for `composer`.
And I've got a few other open source projects on the go.

I'm currently starting a two-person agency with my friend Ed Grosvenor, named Artisan Build,
{{ set browser to https://artisan.build }}
so if you or anyone you know has any **tough** problems to solve, then let us know. We'd love to partner up on some interesting projects. We're also announcing a really cool initiative soon, so keep an eye out for that. 

{{ set OBS to full cam }}

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

{{ set OBS to RightCam }}
{{ set VSC to `Prompt::class` }}
{{ scroll to `prompt()` }}

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
Well now that we're finally done with my `Laravel/Prompts` fork, I think I can show you all some examples
---

{{ window > composer.json (echo-terminal) }}
{{ scroll down }}
Here you can see that I'm just linking `laravel/prompts` to my local fork.
{{ close buffer, open `DocsSearchCommand::class` }}
This is just a bone stock `SearchPrompt` from `laravel/prompts`. Now I know that `artisan` already has a `docs` command. I'm just using this as an example because it's a convenient endpoint to use. This can be literally any scout powered endpoint for any website that has search.

Now in order to get our list of options for our `SearchPrompt`, we're grabbing the search url and building a search payload using the prompt's input. We're `POST`ing to that endpoint, collecting the results, mapping over them, and returning them. Whatever. Not important.

{{ terminal > art docs:search > reverb }}




This works just fine, but it's a little choppy.
Let's exaggerate that by manually adding some delay in here, cause maybe this endpoint is far away, or it's hosted on some slower hardware.

{{ add `sleep(1)` above `[$url, $payload]` }}

{{ terminal > art docs:search > reverb }}

Now _this_ is a way worse experience. When we type the first character of our search, we're sending out a search request, and as we continue typing none of our next keypresses are showing up in the terminal because we're waiting for that search response to come back, which is a blocking operation. So those keypresses are being buffered, then they all show up at once when that response has come back, and we immediately send out another search request with the updated search string, and again none of our typing shows up until that's done.

So what do we do here? Well, let's just debounce posting this search request.

{{ replace `SearchPrompt` with `ScoutPrompt` }}


{{ add `debounce: 300` under `label` param }}


{{ terminal > art docs:search > reverb }}


Way better. At this point we don't even really care that the HTTP request is a blocking operation since we're basically waiting until we're done typing to send it off. So let's take a look at how we're now using the event loop to `debounce` this input.

{{ open `ScoutPrompt::class` }}

There's a lot of this class that we're not going to bother looking at, since it's basically just a copy-paste of `Laravel/Prompts` `SearchPrompt`.
In our `constructor` we're instanciating our `Debouncer` with a delay, and then we're passing `$this->search()` to the `callback` parameter as a `callable`.

If you've never seen these three dots, this is 'first class callable syntax'. It lets you pass a callable directly instead of having to nest it inside of a function that just calls it and doesn't do anything else.

If we scroll down you can see that our `default` action when typing is this `doThings()` method, which is just running this short list of callables in order. I could probably have kept this as an immediately invoked long function, but whatever.

So what's this `Debouncer` class doing.
{{ set VSC to `Debouncer::class` }}

We've got a `static debounce()` method, which is what we called in our `Prompt`, and this is just a static constructor. We're `promoting` the delay and the callback, then we're getting and assigning the `Loop` from reactPHP.

When we `invoke()` the debouncer we're **cancelling** any timer that's been previously registered by the debouncer, then we register a new timer with the same delay, and the same callback.

It's literally that easy. The event loop gives us this `addTimers` method that allows us to reliably and accurate schedule tasks.

This is a super easy win. We don't really care that our `request` is still a blocking operation here, but next let's look at a usecase where we **do** care.
{{ set VSC to `DemoCommand::class` }}
---

In here, we're just collecting an ai prompt from the user, then passing that to a new `StreamedResponsePrompt`. Let's take a look.
{{ set VSC to `StreamedResponsePrompt::class `}}

This prompt doesn't do anything other than rendering the response we get from OpenAI. We get our API key, we make a `Client`, we request a **streamed** response from the `chat` endpoint, then we loop over the stream, extract the chunk, append it to our output, and render. Easy enough. Let's see how it works in the terminal.
{{ terminal > art ai }}

{{ wait }}





Well, that experience sucked. Why didn't it work?
We're still using a browser implementation that's blocking my nature. We could use a spinner to tell the user that something is going on behind the scenes, but the experience we wanted was to render each `chunk` to the terminal as we get it. ReactPHP offers us a non-blocking browser. Now, there _is_ a package to make this new browser PSR-18 compatible, but for now let's just skip the `OpenAI` package and use the api directly.

{{ comment out bad code, comment in good code }}

Here's our _new_ implementation. First let's see it work, then we'll look at the code again.
{{ terminal > art ai }}

{{ wait }}







Much better. Now let's see how it works.
We set the URL, we set the headers, we set the body, we set the `stream` to `true`, we instantiate react's `Browser`, and we send off a request using this `requestStreaming()` method. This method  sends an request, receives a streaming response _without_ buffering the response body, and returns a `promise`. This let's us work on the streamed response as it comes in.

So we get the response `body`, do some checks, and then we register some listeners.

If our `body` recieves a `data` event, we're going to `process` that because sometimes `chunks` will come in multiple at a time, then we loop over them, take the `content` out, and render the component. We've got some other listeners as well, but those are pretty straight forward.

If we wanted to make this more interactive, we could have this response displayed in a `scrollable` window, with a text input below where we can type out our next message while this one is still coming in, since it's all just being done on the event loop and none of this is blocking anymore.

I don't have that set up in _this_ demo, but I **do** have something similar:
---

I'm gonna pause the prompter, cause it's tough to time this one out.
{{ pause prompter, terminal > art chat, unpause }}

So, full disclosure, there's not much code to examine in this demo, since this one is being faked using timers on our `Loop`, but I'm working on a package called `Resonance` which is essentially just a PHP port of `Laravel/Echo` and brings us the ability to use `websockets` to interact with apps using the `Pusher` protocol, which is what `Reverb` uses, in our terminal.

With all the mechanisms I've got set up already in this component and its renderer, it would actually be super easy to listen for these specific events and get this exact experience once `Resonance` is finished.

Where I think this particular example could shine is adding a `help` command to a tool, so instead of going to a support website and clicking on a 'help bubble', where we'd be connected with a support agent, then having to pass info and context back and forth, we could chat directly in our development environment, where we've got access to the filesystem. We could send our `.env`, a config file, or an entire class, directly to the agent, adding important context and speeding up the whole process.

We can also add a `periodic timer` which is great for 'interval based rendering', which is a fancy way of saying that we can use it control the framerate at which we're rendering terminal output.
---

{{ pause, terminal > art flap, unpause }}

This is being rendered at ten frames per second, but it could easily be moved up or down. Every time we hit that periodic timer we're repainting the entire frame, so we can apply gravity to our bird, flap their wings by alternating between sprites, move the pipes, and update the scrore. If we're diligent about working with multibyte characters, we could use box characters or block characters from unicode to make this look even better. I didn't take the time to do that for this demo, I wrote this one in about three hours, but I did for the next one.
---

{{ terminal > art tetris }}
Here we're rendering again at about ten frames per second. We have a working 7-bag randomizer, we can slide our pieces left and right, we can add garbage to the field,

and we can place our pieces.

{{ place two or three pieces, ESC }}

This one actually took a **lot** of code, and there's still a **ton** to do, but I was able to get this working in about 8 hours worth of work. I've defined some custom console colors, I'm using multibyte block characters, I'm doing lots of fun stuff with `zipping` together collections and mapping over them, if we were using websockets we could _easily_ have a second matrix that's controlled remotely, so we could have real-time multiplayer tetris in the terminal.

This really lets you do some crazy fun stuff if your rendering isn't being blocked by anything.

So, this is all really cool and everything, but there _is_ a problem.
---

{{ switch to full cam }}

To have our console output not 'tear' I've registered a custom `ConsoleOutput` implementation that satisfies our contract that uses reactPHP's `WritableResourceStream` that gets passed our `STDOUT` stream. This makes writing to the console non-blocking as well. And this will work perfectly fine most of the time.
**But**
When Laravel boots, it statically registers it's _own_ `ConsoleOutput` implementation so that it can track the number of `newlines` rendered at the end of the previous output. This lets Laravel track whether the next `Prompt` needs to be _prepended_ with extra spacing or not. This might not sounds like a major concern, but, I mean, it's Laravel. Are we surprised that they're putting in the effort to make things beautiful?

Now, because I'm registering my implementation on _my_ abstract class, the net effect is that we might not get the spacing we would normally expect.

But my ultimate plan is to have this be the default looping mechanism for _all_ prompts. So I've got some more work to do before this will be possible using Laravel's package directly.

For now, if you want to experiment, you'll be able to get all of this goodness by installing `artisan-build/community-prompts`.

This is a package that Ed and I set up to provide a home for the prompts the community's built that Taylor wasn't willing to adopt the maintenance burden of. We don't have a ton of components yet, and we need some help filling in documentation, but if you want to dig into this stuff - not even the async stuff, but just useful components, we'd love some new PRs.
---

I think at this point, I can try to take some questions. Was there anything I covered too quickly, or something that didn't really make sense?

{{ pause prompter }}
