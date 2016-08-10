## Story
See [story](https://www.pivotaltracker.com/story/show/125596231)

## Advantage of `context.Context`

1. It is an accepted, standard way of doing things.
1. We force the user to provide a loggregator, when they may not want anything
logged. With context, the user doesn't have to pass loggregator and can just not
specify a value on the `context.Context` object.
1. Concurrency issues can be handled using `context.Context`. Currently, we use `ifrit` to handle concurrency. `context.Context` can be used instead.

## Disadvantage of `context.Context`

1. Implicit arguments. We can place request-scoped variables in the `context.Context` object and retrieve them with `ctx.Value("key")`. It might seem odd to have something passed implicitly
instead of just saying that this function uses this argument. 
However, according to [this guy]
(https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39#.9o93w6x2m),
using `context.Context` for logging is an acceptable paradigm, as the presence/lack
of a logger should not affect our program flow. The obvious candidates for placing in `context.Context` objects are:
* logger
* metadata objects that are not necessary for program flow, such as the `checksum` variable on the `cacheddownloader`
1. Type-safety is a possible concern for passing varibles in `context.Context` object.
1. Need to update to Go 1.7 if we want to use `Context` from the standard library.
Otherwise, we can just import from here `golang.org/x/net/context` pre-1.7.

## What needs to be changed

### Logging
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

### Loggregator/metadata

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

  We can also place use `context.Context` object to place any metadata that do not affect program flow. [This](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39#.reply9bvv) has examples of other metadata objects that can be placed in the `context.Context` object.

### Concurrency

We can use `context.Context` objects for concurrency.

In the `executor`, the `StepRunner` and all the associated steps from the
`Step` interface can be simplified by having the `Perform` function
take a `context.Context` as a variable. This in turn, would allow us
to get rid of the `Cancel` function, because signalling a cancel can be
handled by using the context. The same context, can also be used to push
the logger into each of the steps, hence simplifying the structs that we
use to define each step.

To prove this, we spiked on modifying the `timeoutStep` to take advantage of the
new model, and the code changed as follows:

The struct changed from:

```go
 type timeoutStep struct {
       substep    Step
       timeout    time.Duration
       cancelChan chan struct{}
       logger     lager.Logger
 }
```

to the following:

```go
type timeoutStep struct {
       substep Step
       timeout time.Duration
}
```

Also the `Perform` function was modified from the following:
```go
func (step *timeoutStep) Perform() error {
	resultChan := make(chan error, 1)
	timer := time.NewTimer(step.timeout)
	defer timer.Stop()

	go func() {
		resultChan <- step.substep.Perform()
	}()

	for {
		select {
		case err := <-resultChan:
			return err

		case <-timer.C:
			step.logger.Error("timed-out", nil)

			step.substep.Cancel()

			err := <-resultChan
			return NewEmittableError(err, emittableMessage(step.timeout, err))
		}
	}
}

func (step *timeoutStep) Cancel() {
	step.substep.Cancel()
}
```

to the simplified version below:

```go
func (step *timeoutStep) Perform(ctxt context.Context) error {
	resultChan := make(chan error, 1)

	ctxt, cancel := context.WithTimeout(ctxt, step.timeout)

	go func() {
		resultChan <- step.substep.Perform(ctxt)
	}()

	for {
		select {
		case err := <-resultChan:
			return err

		case <-ctxt.Done():
			cancel()
			err := <-resultChan
			return NewEmittableError(err, emittableMessage(step.timeout, err))
		}
	}
}

```

We no longer have a logger in the struct as that is retrieved by the
`context`, nor a `cancelChan` since a context provides us with a mechanism
for cancelling with `context.WithCancel`.
Instead, we would pass a context object to the `Perform` function of
`timeoutStep`. We also don't have to create timers either, since `context`
already supports `Timeout`.
 
Furthermore, `context` gets passed to the substep, so since these `context`s
are shared, we no longer have code like

```go
func (step *timeoutStep) Cancel() {
  step.substep.Cancel()
}
```


## Other
Found this [discussion](https://news.ycombinator.com/item?id=8103128)
interesting

Overall, the code change involves passing a Context object instead of a
loggregator object around, and it doesn't seem like a difficult change, just a
tedious one.

For `Step` in `executor`, we would follow the simplifications above for all
implementations of interface `Step`.
