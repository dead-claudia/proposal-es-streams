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

Streams and coroutines (ES iterators) are literally duals of each other - streams send, coroutines receive. Streams consume, coroutines yield.

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

> Why is there no way for the caller? Well, that should be handled by [cancellation](https://github.com/tc39/proposal-cancellation), as 1. people see "unsubscribing" and "cancelling a subscription" as equivalent and 2. it's one less thing implementers and consumers have to concern themselves with in many cases. And that proposal could definitely use a big reason to exist, one beyond just aborting network requests and IndexedDB requests.
>
> Also, the callee can already terminate on its own, so that handles much of the use case already.
>
> Why can streams return values? There are a limited few cases where it's useful, like trailers for HTTP requests. Also, the duplex communication means you could do things like yield parsing regexps for the stream to apply and the stream finally emit the constructed result once it completes, a very CSP-like method of doing it. It's admittedly niche, but the use case does in fact exist.

### Stream iteration

A simple construct `for ... from` would exist like so:

```js
// Sync
for (let foo from stream) {
  // ...
}

// Async
for await (let foo from stream) {
  // ...
}

// Do things on error and/or completion
try for (let foo from stream) {
  // ...
} catch (e) {
  // ...
} finally {
  // ...
}
```

> Why "from"? Streams *send* values, so the values could pretty easily be seen as being *from* that stream.

The body of the loop would survive the result of the function, and things like generator `yield`s would not be permitted. However, with `for await ... from`, you can `await` from within the loop body. These are all independent of the surrounding context - `for await ... from` is valid in all contexts, not just `async` contexts.

It'd be equivalent to the following:

```js
// Sync
let done = false
stream[Symbol.stream]().connect({
  next: foo => {
    if (done) return {done: true}
    // ...
    return {done: false}
  },
  throw: e => {
    if (done) return {done: true}
    done = true
    throw e
  },
  return: v => {
    done = true
    return {done: true}
  },
})

// Async
let done = false
stream[Symbol.stream]().connect({
  next: async foo => {
    if (done) return {done: true}
    // ...
    return {done: false}
  },
  throw: async e => {
    if (done) return {done: true}
    done = true
    throw e
  },
  return: async () => {
    done = true
    return {done: true}
  },
})

// Do things on error and/or completion
let done = false
stream[Symbol.stream]().connect({
  next: async foo => {
    if (done) return {done: true}
    // ...
    return {done: false}
  },
  throw: async e => {
    if (done) return {done: true}
    done = true
    // ...
    return {done: true}
  },
  return: async () => {
    if (done) return {done: true}
    done = true
    // ...
    return {done: true}
  },
})
```

You can also `break` from these loops, corresponding to returning `{done: true, value: undefined}`, and you can `continue` from them, too, corresponding to returning `{done: false, value: undefined}`.
