- **Start Date:** 2021-08-19
- **RFC PR:** (leave this empty)

# Errors in XState

## Summary

This RFC goes into how unhandled errors should affect the machine, its ancestry tree and even its current state. It also

## Motivation

Proper error handling on the application level requires clear semantics around the libraries with which the given code interacts. Given that the XState owns a good chunk of code execution and it calls to users' code, it's often unclear how error handling should be implement. Or even how XState behaves when it encounters an uncaught error.

This RFC aims to standardize our algorithm and approach for errors in a way that it could be documented for users, so they could know what to expect of XState in this regard.

## Detailed design

### Effect of uncaught/unhandled errors on the XState runtime

I believe that the only reasonable thing to do when we deal with an **uncaught** exception is to **terminate** the program. If there is no handler for such an error then the author has not predicted it to happen (or was in a hurry and didn't implement it). If there is no handler then the correct behavior for this situation has not been modeled and the state of the overall program after this error happens can only be classified as corrupted.

Should a corrupted program be allowed to continue? This is hazardous. It might not manifest at all in a meaningful way or it might lead to serious damage - this is unknown. But the runtime (IMHO) is not in a position to take a risk here and it has to terminate the program.

This is, of course, not ideal for a broad range of applications - errors happen and we can't just shut down the entire system. But this is usually handled by rebooting, defensive programming, or just, well, explicit, granular error handling.

What SCXML says about this? It just tells implementers to raise the `error.execution` event (with the error data contained in it). There is no requirement for actually handling this event though so, in case somebody forgets to handle this (very common in XState and very easy to do) then the error is effectively **swallowed** and we end up with the potentially corrupted state.

Even if we add extra logs for those unhandled events... those are easy to miss (and that doesn't try to address the corrupted state problem). And, in general, I believe that it's the runtime's responsibility to make such errors as easy to be discovered as possible so the correct handling could be implemented by the author.

I have also 2 specific examples from the library land that has switched over to **termination** semantics because swallowing errors, and thus leading to corrupted errors, was too problematic and reported by the community as the problem:

- React: v16 switched to unmounting the whole tree when an uncaught error happens during lifecycle methods ( https://reactjs.org/blog/2017/09/26/react-v16.0.html#better-error-handling )
- Redux Saga: in it stopped swallowing uncaught errors from sync put (Redux's `dispatch`) execution

### Actor hierarchy considerations

Actors form tree-like hierarchies - each actor has a parent (unless it's a top-level actor). If the error gets thrown and the actor didn't deal with it effectively means that it has crashed. For a class of errors (like promise rejections) we already let know the parent about this and try to select the appropriate `invoke.onError` transition).

I propose to treat all errors the same - let the actor crash, notify the parent about it, and let it decide what should happen in such a situation. If it doesn't decide to handle a particular error then let the parent crash and propagate the error to its parent. The process should be performed recursively - up to the very root.

This means that any uncaught/handled error has the potential of killing the whole actor tree and thus it most likely makes the app in question unusable. This sounds a little bit scary but this already works like that in a lot of different systems.

Not every actor gets invoked though. We also support spawned actors and we can't easily define `onError` transitions for them. The only way out of this situation, that I can think of, is to just require "regular" transitions to handle errors of spawned actors. Since an error is an event in XState one can define transitions for descriptors like `error.execution.someid`, `error.*` and so on.

### States hierarchy

I propose to extend states config with `onError`. This would allow for selective error handling - per state. In many cases a single root `onError` would probably be enough but since the root is an implicit state I find it more intuitive to allow this to be defined on any state. Selective handling could potentially be used in creative and neat ways. In a similar fashion to errors bubbling through the actor tree errors would bubble up through the state tree.

According to the SCXML semantics, errors are just events - internal or external ones, depending on the source of an error. This makes them processed like any other event of a given type.

I find this problematic. Consider a simple example like this:
```js
entry: [
    raise('EV'),
    assign(() => { throw new Error('oops') }),
    assign({ counter: 42 })
]
```

According to the SCXML semantics:
- the error event would get queued up after the `EV` event
- the `counter` value after this whole `entry` handler would be 42

The problem that I see with the first thing is that by the time that we get to selecting transitions we can no longer be in a state containing `onError` that was designed to handle that error (after all a transition to a different state could happen based on EV event). It makes the runtime behavior less predictable.

The problem with the second thing is that an error doesn't abrupt the execution which is the most common result of throwing errors. So it, once again, makes the thing less predictable. Since we strictly define the order of actions it may happen that action would get executed when its implementation is depending on the successful execution of all prior actions. Any particular action could depend on a value being successfully assigned in a preceding action.

What I'm proposing is to literally get more in line with a classic and familiar:
```js
try {
    action()
} catch (err) {
    selectTransitions(err)
}
```

This would IMHO solve both of the mentioned problems and would be easier to understand by most of our users.

It's worth mentioning that this would be a substantial change in the core of the SCXML algorithm. It also would have an effect on potentially **abrupting** transitions. Right now the transition, conceptually, runs as a whole unit. If we have a state like this:
```js
{
    id: 'someid',
    initial: 'foo',
    entry: actionA,
    states: {
        foo: {
            entry: actionB,
        }
    }
}
```

then we always end up in the `foo` state (even if temporarily) after selecting a transition targeting the `#someid` state. With the proposed changes that would not always happen because the `enterStates` procedure could get interrupted by an error thrown by `actionA`.

### Communicating intentional errors

There are situations in which one could want to throw an error on purpose, for a transition to be selected by the appropriate `onError` handler. In a way - since errors are just events, one could just raise the appropriate event but there are no means to do that from within actions. Throwing errors seems to be a straightforward pattern that could be leveraged for such purposes. One situation that comes to mind when this might be useful is validation-like logic in actions.

Since often setting the error's message is not enough for handling a particular error it would be nice to have a way to carry some additional information with the raised event. To do that I would propose introducing a new export from XState that would just standardize this. Something like:
```js
throw RaisedError('Message', {
    value: 42
})
```

which would get converted to an error event like:
```
{
    type: 'error.execution',
    data: {
        value: 42
    }
}
```

## How we teach this

A new section should be created in the docs. It would explain how we approach errors - both internal ones and external ones (communicated by other actors). It should also come with practical examples on how one can deal with specific, common, scenarios.

## Unresolved questions

1. How event names for error events should be created? SCXML is really vague about that - especially when it comes to `error.platform.*` events. We should list common scenarios, including errors thrown by a machine and errors "thrown" by a child actor, and decide on the naming scheme that should be used for a given error and situation.
2. Are there any particular patterns that we should incorporate and document when it comes to converting thrown values to error events?
