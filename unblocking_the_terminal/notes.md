What problem are we trying to solve

lexicon
	blocking - synchronous
	non-blocking - asynchronous... kind of
	event loop
	debouncing (we'll talk later)

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
