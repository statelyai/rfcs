- **Start Date:** (fill me in with today’s date, 2022-05-22
- **RFC PR:** (leave this empty)

# `@xstate/react` Global Hooks

## Summary

This RFC seeks to establish a simple way to create hooks for global actors (instances of XState machines) that can be accessed in any component. This will allow for shared state patterns, similar to Redux, Zustand, Recoil, and others, without the need for React context providers. Shared state is a very common use-case for non-trivial React applications, and this would enable state machines to be used at any level: component and/or app-wide (global) state with ease.

**See the PR:** https://github.com/statelyai/xstate/pull/3401

## Motivation

TODO

Inspiration:

- [Zustand](https://github.com/pmndrs/zustand)
- [Redux (in React)](https://react-redux.js.org/)
- [Recoil](https://recoiljs.org/)

## Detailed design

**Quick overview:**
```js
import { createHooks } from '@xstate/react';
import { authMachine } from './authMachine';
import { interpret } from 'xstate';

// Create the actor
const authService = interpret(authMachine, {/* ... */}).start();

// Create a "hooks object" from the actor
const auth = createHooks(authService);

// ...

const SomeForm = () => {
  // Get full state snapshot and send() function
  const [state, send] = auth.useActor();

  // Get selected state
  const isLoggedIn = auth.useSelector(state => state.hasTag('authorized'));

  // ...
}
```

**Details**

```js
import { createHooks } from '@xstate/react';
import { authMachine } from './authMachine';

const authService = interpret(authMachine, {/* ... */}).start();

const auth = createHooks(authService);

// Returned object has the actor ref instance
auth.actorRef === authService; // true

// Object has two extra methods to be used as hooks:

// Hook for accessing [state, send] from actor
const [state, send] = auth.useActor();

// Hook for accessing selected state from actor
const derivedState = auth.useSelector(selectorFn, comparator);
```

**Actor initialization control**

Developers have full control over when the actor is started:

```js
// ...

// Actor not started yet
const authService = interpret(authMachine, {/* ... */});
const auth = createHooks(authService);

const App = () => {
  // If you must start the actor in a useEffect...
  useEffect(() => {
    auth.actorRef.start();

    // Since it's a global actor, no need for cleanup
  }, []);

  const [state, send] = auth.useActor();

  // ...
}
```

```js
// ...

// Actor not started yet
const authService = interpret(authMachine, {/* ... */});
const auth = createHooks(authService);

// Only start in browser
if (typeof window !== undefined) {
  auth.actorRef.start();
}

const App = () => {
  const [state, send] = auth.useActor();

  // ...
}
```


### Technical Background

This builds on two existing hooks: `useActor()` and `useSelector()`.

### Implementation

```ts
// Simplified
import { useActor, useSelector } from '@xstate/react';

function createHooks(actorRef: ActorRef) {
  return {
    actorRef,
    useActor() {
      return useActor(actorRef);
    },
    useSelector(selector, comparator) {
      return useSelector(actorRef, comparator);
    }
  };
}
```

## How we teach this

TODO

## Drawbacks

- There are fast-refresh/HMR compatibility concerns, similar to those listed [here (Zustand)](https://github.com/pmndrs/zustand/issues/908). However, we can probably still have "good enough" support with the same fast-refresh capabilities as currently exist:

  > this kind of Fast Refresh support isnt that bad - kinda what we are doing at the moment anyway (just blowing up the old state and restarting)
  > @Andarist (https://discord.com/channels/795785288994652170/976863583336022127/976875867655524412)

- This may feel unfamiliar to React devs who would naturally reach for [React Context](https://reactjs.org/docs/context.html) for shared state; however, Zustand has already popularized the notion of "contextual state" outside of React Context: https://github.com/pmndrs/zustand#why-zustand-over-context
  - We can always add a `Provider` API if this becomes a concern:

  ```js
  const auth = createHooks(authService);

  function App() {
    return <auth.Provider>
      {/* ... */}
    </auth.Provider>;
  }
  ```

## Alternatives

- Previously, only passing the machine was considered; however, this breaks separation of concerns now that `createHooks(...)` becomes responsible for 1) interpreting the machine and 2) creating the hooks. The other downside was that it doesn't become easy for `createHooks(...)` to accept _any_ actor (even non-machine actors), which it should theoretically be able to do without issue.

  ```js
  import { createHooks } from '@xstate/react';
  import { authMachine } from './authMachine';

  // Alternative considered: passing in the machine instead of the actor
  const auth = createHooks(authMachine);
  ```

- Another alternative was considered in the [`createActorContext` PR](https://github.com/statelyai/xstate/pull/2783). This is similar to the above approach, but with the same SoC downsides.

## Unresolved questions

- Will fast-refresh support be possible/satisfactory? What should devs expect for this?
- Are there any downsides to providing the actor instead of the machine (original idea)?
