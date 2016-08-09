## Story
See [story](https://www.pivotaltracker.com/story/show/125596231)

## Advantage of `context.Context`

1. It is an accepted, standard way of doing things.
1. We force the user to provide a loggregator, when they may not want anything
logged. With context, the user doesn't have to pass loggregator and can just not
specify a value on the `Context` object.

## Disadvantage of `context.Context`

1. Implicit arguments. It might seem odd to have something passed implicitly
instead of just saying that this function uses this argument. 
However, according to [this guy]
(https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39#.9o93w6x2m),
using `Context` for logging is an acceptable paradigm, as the presence/lack
of a logger should not affect our program flow.
1. Type-safety is a possible concern. 
1. Need to update to Go 1.7 if we want to use `Context` from the standard library.
Otherwise, we can just import from here `golang.org/x/net/context` pre-1.7.

## What needs to be changed

To get Context to work with our middleware stuff in bbs, you need to:

1. add an explicit `Context` parameter to each function in bbs (which seems to
be more popular option), as a parameter (seems to be more standard way) or as
an optional config on a request structure.  For the BBS and auctioneer
handlers, it seems simple enough to replace the loggregator parameter with
`Context`.

OR

1. use a package to map http.Requests to `Contexts`
1. I'm not sure of this second option as we'd have to store them all
somewhere to make sure we don't by accident leave `Contexts` in
the map that we don't need. It seems the first option is more popular. There
are packages that do this for you, such as
[gorilla](https://github.com/gorilla/context)

## Will require a minor refactoring of a lot of things

Instead of passing around a loggregator in the handler functions, we'd need to pass around a Context.
These repos are affected:

1. `bbs` (all the handlers need to be changed to have `Context` and not loggregator
and related test code. Our main function will create a `context.Background` and
we can get rid of the middleware.`LogWrap` function.)

1. `auctioneer` (same as `bbs`)

These need to be changed because they use the `bbsClient`.

1. `benchmarkbbs`
1. `cfdot`
1. `route-emitter`
1. `diego-ssh`
1. `inigo`
1. `rep`
1. `vizzini`


## Other
Found this [discussion](https://news.ycombinator.com/item?id=8103128)
interesting

Overall, the code change involves passing a Context object instead of a
loggregator object around, and it doesn't seem like a difficult change, just a
tedious one.
