## Story
See [story](https://www.pivotaltracker.com/story/show/125596231)

## Background
#### Advantage of `context.Context`

1. It is an accepted, standard way of doing things.
1. We force the user to provide a loggregator, when they may not want anything
logged. With context, the user doesn't have to pass loggregator and can just not
specify a value on the `context.Context` object.
1. Concurrency issues can be handled using `context.Context`. Currently, we use `ifrit` to handle concurrency. `context.Context` can be used instead.

#### Disadvantage of `context.Context`

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

## Benefits of Context in Diego Release

### 1. Metadata / Cross Cutting Concerns
Context is useful when we want to pass aspects that are not defining the behavior of the code. This includes cross-cutting
concerns such as logging and security checks. Also meta-data related to the behavior of the an object can be passed in
through context. The benefit is that, metadata is not directly affecting the behavior of the system and over time may grow
or shrink. Passing metadata through context allows the code signature to stay the same and potentially for the complexity of the code to be reduced.

#### Logging
One of the immediate advantages of using context, is to allow for passing the `logger` object through to the handlers, without having the object be a parameter for every endpoint.

There are two options for consideration:

*Option 1:*

This can be easily achieved by modifying the `middleware` in `bbs` where the following change is possible:

```go
func LogWrap(logger lager.Logger, loggableHandlerFunc LoggableHandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		requestLog := logger.Session("request", lager.Data{
			"method":  r.Method,
			"request": r.URL.String(),
		})

		// two new lines below
		ctx := context.WithValue(req.Context(), "logger", &requestLog)
		r = r.WithContext(ctx)

		requestLog.Debug("serving")
		loggableHandlerFunc(requestLog, w, r)
		requestLog.Debug("done")
	}
}
```

And then in the handler, the logger can be used like the following:

```go
// the logger is removed from the function signature below
func (h *PingHandler) Ping(w http.ResponseWriter, req *http.Request) {
  // the logger is read from the context of the Request
  // passing it with pointer, allows for the session information to persist
    logger := req.Context().Value("logger").(*lager.Logger)
	response := &models.PingResponse{}
	response.Available = true
	writeResponse(w, response)
}
```

Similar change can be done to `auctioneer`'s `handlers.go` as well. Alternatively the logger can be taken out of `LogWrap` and context to be passed in.

*Option 2:*

We can modify each handler to have a `context` object passed to it as a first argument. This will be in line with the pattern Google follows as [discussed here](https://news.ycombinator.com/item?id=8103128)

Overall, the code change involves passing a Context object instead of a
`loggregator` object around, and it doesn't seem like a difficult change, just a
tedious one.

In case of the above handler the change would be as follows:
```go
// the logger is removed from the function signature below
func (h *PingHandler) Ping(ctx context.Context, w http.ResponseWriter, req *http.Request) {
  // the logger is read from the context of the Request
  // passing it with pointer, allows for the session information to persist
    logger := ctx.Value("logger").(*lager.Logger)
	response := &models.PingResponse{}
	response.Available = true
	writeResponse(w, response)
}
```

*Option 3:*

Rather than including the `logger` object into the `context`, we can alternatively pass the session information and create a new logger object in each handler. On the negative side, this would result in having multiple `logger` objects created in different functions throughout the flow of code execution, which may not necessarily be desirable. On the positive side, passing session names around results in having immutable objects stored in the `context` which is potentially a better practice.

**Suggestion**: We advocate for the second option, because it prevents unnecessary objects getting created, and also offers a cleaner implementation. Reconstructing the logging object seems to be unnecessary and of little value.

This is however a fairly big change the signature of functions in `bbs` and `auctioneer`. The following repos will be affected"

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

### 2. Concurrency

Functionality offered by `Context` can also be used to simplify some of the logic when dealing with concurrency. We explored a few areas in the code where `Context` can be helpful with simplifying the code.

_For an example, consider the scenario below:_

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

	ctxt, _ := context.WithTimeout(ctxt, step.timeout)

	go func() {
		resultChan <- step.substep.Perform(ctxt)
	}()

	for {
		select {
		case err := <-resultChan:
			return err

		case <-ctxt.Done():		
			err := <-resultChan
			return NewEmittableError(err, emittableMessage(step.timeout, err))
		}
	}
}

```

We no longer have a logger in the struct as it is retrieved by the
`context`, nor a `cancelChan` since a context provides us with a mechanism
for canceling by using `context.WithCancel`. Instead, we would pass a context object to the `Perform` function of
`timeoutStep`. We also don't have to create timers, since `context`
has support for `Timeout`.

More interestingly, `context` gets passed to the substeps, so since these `context`
is shared, canceling the parent `context` will propagate through and to the children,
which exempts us from having to write code below and make sure we call it on the child steps.

```go
func (step *timeoutStep) Cancel() {
  step.substep.Cancel()
}
```

The above is only one possibility for how `context` can be used in `diego release`. There are going to be other possibilities for it as well if we choose to dig deeper.

## Conclusion

There are clear benefits in using `context` and achieving simplicity in the code. However the cost of refactoring could be rather expensive. For example, adding `logger` to the code is a fairly trivial change that can be easily achieved. On the other hand, using `context` to achieve concurrency allows for better code but at the cost of more significant change to the code base.
