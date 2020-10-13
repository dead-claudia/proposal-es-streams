# Streams for JS

This is a proposal for adding streams to JS.

> These are not your mother's streams, however. These are different.

## Core intuition

Consider this table. In each case, the left and right are duals.

> "Dual" is math speak for being opposites.

| Emit                | React               |
|:-------------------:|:-------------------:|
| Invoke function     | Receive arguments   |
| Return with value   | Handle return value |
| Resolve promise     | Handle resolution   |
| Generate collection | Iterate collection  |
| Send value          | Receive value       |
| Produce value       | Consume value       |

Here's how many of those mirror to JS features:

| Emit                    | React                                                   |
|:-----------------------:|:-------------------------------------------------------:|
| `func(...args)`         | `function func(...args) { ... }`                        |
| `return result`         | `result = ...`                                          |
| `throw error`           | `try { ... } catch (error) { ... }`                     |
| `resolve(value)`        | `.then(value => ...)` or `value = await ...`            |
| `reject(error)`         | `.catch(error => ...)` or `try`/`catch` above + `await` |
| `yield value`           | `for (const value of ...) { ... }`                      |
| `.dispatchEvent(value)` | `.onevent = value => { ... }`                           |
| `.emit("event", value)` | `.on("event", value => { ... })`                        |
| `.next(value)`          | `next(value) { ... }`                                   |

There's two very conspicuous entries in that: the last two lines. They're both callbacks, and neither of them are actually dedicated syntax. They also look very similar and seem to fulfill similar roles. However, there's a twist here: the last one, `next`, is a generator method. And in modules like `co` and `redux-saga`, it's invoked not as `.next()`, but `.next(value)`, and the generator receives that as its input value. [ReactiveX observers also sport a `.next(value)` method](https://rxjs.dev/guide/observer), one with the same exact prototype. And just like generators, there's separate methods for propagating a fatal error (`.error(e)` for observers, `.throw(e)` for generators) and terminating prematurely (`.complete()` for observers, `.return(value)` for generators). There's a pattern here: if you simply change the names, they actually represent the same thing. And yes, they're duals. If you look at these two examples, the only things different are names, yet you can technically stick a generator instance in one of them.

```js
// Taken from RxJS's docs as an example
observable.subscribe({
  next: x => console.log('Observer got a next value: ' + x),
  error: err => console.error('Observer got an error: ' + err),
  complete: () => console.log('Observer got a complete notification'),
});

observable.subscribe({
  next: x => console.log('Observer got a next value: ' + x),
  throw: err => console.error('Observer got an error: ' + err),
  return: () => console.log('Observer got a complete notification'),
});
```

Streams and coroutines (ES iterators) are literally duals of each other - streams send, coroutines receive. Streams consume, coroutines yield. And it's this core intuition that allowed me to figure out what the proper abstraction was and also what the best syntax for it was.

## Core Proposal

This comes in three parts:

- Streamable + stream protocols
- `for ... from` for streams, to mirror `for ... of` for iterators
- Emitters for streams, to mirror generators for iterables

> I'm intentionally stopping short of proposing stream helpers, as that can be addressed in a follow-on proposal similar to [what was done with iterators](https://github.com/tc39/proposal-iterator-helpers).

### Streamable + stream protocols

- Streamables are objects with a `Symbol.stream` method similar to the `Symbol.iterator` method of iterables.
- Streams are objects with a single method `.connect(sub)`, where `sub` is an iterable.
  - If `sub.next(value)` returns a truthy `result.done`, the stream should cease any subsequent emits.
  - To terminate, invoke either `sub.throw(err)` or `sub.return(value)`.
- Async streamables are objects with a `Symbol.asyncStream` method similar to the `Symbol.asyncIterator` method of async iterables.
- Async streamables are objects with a single method `.connect(sub)`, where `sub` is an async iterable.
  - If `await sub.next(value)` returns a truthy `result.done`, the stream should cease any subsequent emits.
  - To terminate downstream, invoke either `sub.throw(err)` or `sub.return(value)`.
- A `%StreamPrototype%` would exist, corresponding to `%IteratorPrototype%`, and an `%AsyncStreamPrototype%` would exist similarly to `%AsyncIteratorPrototype%`.

Here's a real-world example of an implementation of both of those protocols together using an event listener adapter:

```js
function fromEvent(eventTarget, name, opts) {
  return {
    [Symbol.stream]() {
      return {
        connect(o) {
          const iter = o[Symbol.iterator]()
          const handle = event => {
            try {
              const {done, value} = iter.next(event)
              if (done) eventTarget.removeEventListener(name, handle, false)
            } catch (e) {
              eventTarget.removeEventListener(name, handle, false)
              iter.throw(e)
            }
          }
          eventTarget.addEventListener(name, handle, opts)
        }
      }
    },
    [Symbol.asyncStream]() {
      return {
        connect(o) {
          const iter = o[Symbol.asyncIterator]()
          const handle = async event => {
            try {
              const {done, value} = await iter.next(event)
              if (done) eventTarget.removeEventListener(name, handle, false)
            } catch (e) {
              eventTarget.removeEventListener(name, handle, false)
              iter.throw(e)
            }
          }
          eventTarget.addEventListener(name, handle, opts)
        }
      }
    },
  }
}
```

### Stream iteration

A simple construct `for ... from` would exist like so, usable only in async and stream contexts (as they need awaited):

```js
// Sync
for (let foo from stream) {
  // ...
}

// Async
for async (let foo from stream) {
  // ...
}

// Do things on error and/or completion
try {
  for (let foo from stream) {
    // ...
  }
} catch (e) {
  // ...
}
```

> Why "from"? Streams *send* values, so the values could pretty easily be seen as being *from* that stream.

The body of the loop would survive the result of the function, and things like generator `yield`s would not be permitted. However, with `for async ... from`, you can `await` from within the loop body and the `await` is from the context of that body specifically. These are all independent of the surrounding context - `for async ... from` is valid in all contexts, not just `async` contexts. Note that you can't `yield` from inside the body in async generators, and you can only `await` from inside the loop body of *async* streams - the corresponding invariants are in reverse.

It'd be roughly equivalent to the following within async functions:

```js
// Sync
await new Promise((resolve, reject) => {
  let done = false
  stream[Symbol.stream]().connect({
    next: foo => {
      if (done) return {done: true, value: undefined}
      // ...
      return {done: false, value: undefined}
    },
    throw: e => {
      if (done) return {done: true, value: undefined}
      done = true
      reject(e)
      return {done: true, value: undefined}
    },
    return: v => {
      done = true
      resolve()
      return {done: true, value: undefined}
    },
  })
})

// Async
await new Promise((resolve, reject) => {
  let done = false
  stream[Symbol.stream]().connect({
    next: async foo => {
      if (done) return {done: true, value: undefined}
      // ...
      return {done: false, value: undefined}
    },
    throw: async e => {
      if (done) return {done: true, value: undefined}
      done = true
      reject(e)
      return {done: true, value: undefined}
    },
    return: async v => {
      done = true
      resolve()
      return {done: true, value: undefined}
    },
  })
})

// Do things on error and/or completion
try {
  await new Promise((resolve, reject) => {
    let done = false
    stream[Symbol.stream]().connect({
      next: foo => {
        if (done) return {done: true, value: undefined}
        // ...
        return {done: false, value: undefined}
      },
      throw: e => {
        if (done) return {done: true, value: undefined}
        done = true
        reject(e)
        return {done: true, value: undefined}
      },
      return: v => {
        done = true
        resolve()
        return {done: true, value: undefined}
      },
    })
  })
} catch (e) {
  // ...
}
```

You can also `break` from these loops, corresponding to returning `{done: true, value: undefined}` from `next` (and terminating the loop), and you can `continue` from them, too, corresponding to returning `{done: false, value: undefined}`.

There would initially not be a sync-to-async adapter for streams as [there would first need to be a way to close a stream](#why-is-there-no-way-for-the-caller-to-close-streams). However, adapters would exist for making iterables (sync and async) streamable in both ways, that'd work roughly like this:

```js
function coerceToSyncStream(value) {
  if (typeof value[Symbol.stream] === "function") {
    return value[Symbol.stream]()
  } else if (typeof value[Symbol.iterator] === "function") {
    return {
      connect(o) {
        const outputIter = o[Symbol.iterator]()
        const valueIter = value[Symbol.iterator]()
        while (true) {
          let done, value
          try {
            ({done, value} = valueIter.next())
          } catch (e) {
            o.throw(e)
            return
          }
          if (done) {
            o.return(value)
            return
          } else {
            const {done, value} = o.next(value)
            if (done) iter.return()
          }
        }
      },
    }
  } else if (typeof value[Symbol.asyncIterator] === "function") {
    return {
      connect(o) {
        const outputIter = o[Symbol.iterator]()
        const valueIter = value[Symbol.asyncIterator]()
        ;(async () => {
          while (true) {
            let done, value
            try {
              ({done, value} = await valueIter.next())
            } catch (e) {
              o.throw(e)
              return
            }
            if (done) {
              o.return(value)
              return
            } else {
              const {done, value} = o.next(value)
              if (done) iter.return()
            }
          }
        })()
      },
    }
  } else {
    throw new TypeError("value is neither iterable nor streamable")
  }
}

function coerceToAsyncStream(value) {
  if (typeof value[Symbol.asyncStream] === "function") {
    return value[Symbol.asyncStream]()
  } else if (typeof value[Symbol.iterator] === "function") {
    return {
      connect(o) {
        const outputIter = o[Symbol.asyncIterator]()
        const valueIter = value[Symbol.iterator]()
        ;(async () => {
          while (true) {
            let done, value
            try {
              ({done, value} = valueIter.next())
            } catch (e) {
              return o.throw(e)
            }
            if (done) {
              return o.return(value)
            } else {
              const {done, value} = await o.next(value)
              if (done) iter.return()
            }
          }
        })()
      },
    }
  } else if (typeof value[Symbol.asyncIterator] === "function") {
    return {
      connect(o) {
        const outputIter = o[Symbol.asyncIterator]()
        const valueIter = value[Symbol.asyncIterator]()
        ;(async () => {
          while (true) {
            let done, value
            try {
              ({done, value} = await valueIter.next())
            } catch (e) {
              return o.throw(e)
            }
            if (done) {
              return o.return(value)
            } else {
              const {done, value} = await o.next(value)
              if (done) iter.return()
            }
          }
        })()
      },
    }
  } else {
    throw new TypeError("value is neither iterable nor streamable")
  }
}
```

### Emitters

Stream emitters are like generators, but for streams instead of iterables. The basic syntax is this:

```js
function +func(...args) {
  // ...
}
```

> Syntactically, it's nearly identical to a generator, but instead of `function *`, it's `function +`. If someone has a better idea for a sigil, I'm all ears.

The above code returns a stream that works something like this:

```js
return {
  [Symbol.stream]() { return this },
  connect(o) {
    const iter = o[Symbol.iterator]()
    // ...
    iter.return(/* ... */)
  }
}
```

Like generators, they can `yield` values, though this sends rather than receives.

```js
function +fromList(list) {
  for (const item of list) yield item
}

// Rough equivalent
function fromList(list) {
  return {
    [Symbol.stream]() { return this },
    connect(o) {
      const iter = o[Symbol.iterator]()
      for (const item of list) {
        const {done, value} = iter.next(item)
        if (done) return
      }
      iter.return()
    }
  }
}
```

You can even return values from streams and `yield` from inside `for ... from` blocks (unlike with async generators), which is really useful for things like "scan", a form of "reduce" that emits its intermediate values as well:

```js
function +scan(stream, func, initial) {
  let current = initial
  for (const item from stream) {
    current = func(current, item)
    yield current
  }
  return current
}

// Rough equivalent
function scan(stream, func, initial) {
  return {
    [Symbol.stream]() { return this },
    connect(o) {
      const iter = o[Symbol.iterator]()
      let current = initial
      let parentDone = false
      stream[Symbol.stream]().connect({
        next(item) {
          if (parentDone) return {done: true, value: undefined}
          current = func(current, item)
          const {done, value} = iter.next(current)
          if (done) return {done: true, value: undefined}
        },
        throw(e) {
          if (parentDone) return {done: true, value: undefined}
          parentDone = true
          iter.throw(e)
          return {done: true, value: undefined}
        },
        return (v) {
          if (parentDone) return {done: true, value: undefined}
          parentDone = true
          iter.return(current)
          return {done: true, value: undefined}
        },
      })
    }
  }
}
```

You can even `yield*` to other streams and receive their return values, which is useful when you both need to process data and stream results generated from it. (This here is an academic example, as I've not once run into a use case for this that isn't incredibly complex.)

```js
// Logs the following:
// 1
// 2
// done
function +foo() {
  yield 1
  yield 2
  return "done"
}

function +bar() {
  const result = yield* foo()
  console.log(result)
}

;(async () => {
  for (const item from bar()) {
    console.log(item)
  }
})()

// Rough equivalent
function foo() {
  return {
    [Symbol.stream]() { return this },
    connect(o) {
      const iter = o[Symbol.iterator]()
      { const {done, value} = iter.next(1); if (done) return }
      { const {done, value} = iter.next(2); if (done) return }
      iter.return(3)
    }
  }
}

function bar() {
  return {
    [Symbol.stream]() { return this },
    connect(o) {
      const iter = o[Symbol.iterator]()
      let done = false
      foo()[Symbol.stream]().connect({
        next(v) {
          if (done) return {done: true, value: undefined}
          return iter.next(v)
        },
        throw(e) {
          if (done) return {done: true, value: undefined}
          done = true
          return iter.throw(e)
        },
        return(result) {
          if (done) return {done: true, value: undefined}
          done = true
          console.log(result)
          return {done: true, value: undefined}
        },
      })
    }
  }
}

;(async () => {
  const stream = bar()
  await new Promise((resolve, reject) => {
    let done = false
    stream[Symbol.stream]().connect({
      next(item) {
        if (done) return {done: true, value: undefined}
        console.log(item)
        return {done: false, value: undefined}
      },
      throw(e) {
        if (done) return {done: true, value: undefined}
        done = true
        reject(e)
        return {done: true, value: undefined}
      },
      return(v) {
        if (done) return {done: true, value: undefined}
        done = true
        resolve(v)
        return {done: true, value: undefined}
      },
    })
  })
})()
```

And of course, async emitters can exist:

```js
async function +fromList(list) {
  for (const item of list) yield item
}

// Rough equivalent
function fromList(list) {
  return {
    [Symbol.asyncStream]() { return this },
    connect(o) {
      ;(async () => {
        const iter = o[Symbol.asyncIterator]()
        for (const item of list) {
          const {done, value} = await iter.next(item)
          if (done) return
        }
        iter.return()
      })()
    }
  }
}
```

## Potential questions

I've got answers to a few potential questions here, as I know some parts of this are very much *not* obvious.

### Why is this all so verbose? There's got to be a way to tame this...

Iterables and their iterators aren't exactly simple to implement for anything not trivial, either. Also, stream helpers [similar to what's proposed for iterators](https://github.com/tc39/proposal-iterator-helpers) are beyond the scope of this proposal.

### What happens to errors that fall out of async generators?

A new "HostReportError(error)" hook would need added to address that. But in general, it just equates to an always-unhandled rejection, and the hook might be repurposed for that as well.

### What about that first value? Aren't generators *not* going to have access to that?

That's true, but of course, [that's also a known issue that's being addressed independently of this](https://github.com/tc39/proposal-function.sent). And of course, many of the above desugared subscribers could be somewhat simplified as a result. Here's a few examples:

```js
// Do things on error and/or completion
try {
  await new Promise((resolve, reject) => {
    stream[Symbol.stream]().connect((function *() {
      try {
        while (true) {
          const item = yield
          // ...
        }
      } catch (e) {
        reject(e)
      } finally {
        resolve()
      }
    })())
  })
} catch (e) {
  // ...
}

function scan(stream, func, initial) {
  return {
    [Symbol.stream]() { return this },
    connect(o) {
      const iter = o[Symbol.iterator]()
      stream[Symbol.stream]().connect((function *() {
        let current = initial
        let success = true
        try {
          while (true) {
            current = func(current, function.sent)
            yield current
          }
        } catch (e) {
          success = false
          iter.throw(e)
        } finally {
          if (success) iter.return(current)
        }
      })())
    }
  }
}
```

### Why is there no way for the caller to close streams?

Well, that should be handled by [cancellation](https://github.com/tc39/proposal-cancellation) in my honest opinion.

1. People see "unsubscribing" and "cancelling a subscription" as conceptually equivalent in literally all cases.
1. It's one less thing implementers and consumers have to concern themselves with in many cases.
1. Also, the callee can already terminate on its own, so that handles much of the use case already.
1. That proposal could definitely use a big reason to exist, one beyond just aborting network requests and IndexedDB requests.

There's reasons. And rather than inventing *yet another abstraction*, let's actually create that shared abstraction already that'd become useful *today* in so many areas.

### What's the point of allowing subscriptions to return values back to the streams they subscribe to?

It's not a common need, but when it's needed, all the alternatives are far more complicated:

- Backpressure tracking pretty much requires information to be propagated back, so writers don't overload their readers. For this purpose, returning a simple number of bytes it's still reading is sufficient, and beyond that, it's very difficult to overload a subscription.
  - Node's `stream.write` does [iterally this very thing](https://nodejs.org/api/stream.html#stream_buffering), though it handles most the backpressure internally.
- For things like HTTP request handling, being able to receive errors thrown by applications is really valuable and allows a *lot* of logic that would otherwise pollute server handling to be drastically simplified.

### What's the point of async streams?

While yes, most streams in practice might be synchronous, there are use cases for async streams, ones larger than you might expect:

- HTTP server request handling could be modeled as an async stream. File system operations are almost always handled asynchronously, so it makes sense to `await` them. Also, streams can handle uncaught errors and translate them into 4xx (like "file not found") and 5xx (for most non-FS errors) responses as appropriate, with little need for the user to care. And yes, catching async exceptions is also really valuable.
- Any stream that requires backpressure tracking and could potentially do async actions generally need to know when the consumer is free to process more data.

### Why do these streams support return values?

It's similar to why generators are capable of returning values. It's a niche use case, but one where there really isn't any alternative that neither is hacky to the max nor forces you to write the iterator (or in this case, stream) manually. And since all use cases I've ever personally encountered were niche and very complicated, I personally do *not* have a real-world example I can readily point you to. But in general, it's basically like coroutines, but in the opposite direction.

Also, it takes about the same amount of effort to implement either way, and when it's that cheap to implement, even "nice to have" is a sufficient justification for inclusion.
