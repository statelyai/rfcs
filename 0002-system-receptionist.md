- **Start Date:** 2022-04-15
- **RFC PR:** (leave this empty)

# Systems and receptionists

## Summary

Actor discovery is an important part of the actor model. This RFC proposes the following for the purpose of making it easier to find discoverable actors within a system using the **receptionist** pattern:

- The creation of an actor system via `createSystem(...)` 
- A system receptionist via `system.receptionist`
- Receptionist actions:
  - Registering an actor ref: `register(key)`
  - Unregistering an actor ref: `unregister(key)`

## Motivation

https://doc.akka.io/docs/akka/current/typed/actor-discovery.html

Naturally, actors form a hierarchical relationship, such that it is easiest for a parent actor to communicate with a child actor, and for a child actor to _sometimes_ communicate with the parent, if it has knowledge of it. However, these communication patterns are more difficult:

- An actor communicating with a sibling actor
- An actor communicating with any arbitrary actor
- A system-level actor (such as a logging or notification actor) being made available to any actor
- Retrieving non-stale data from another actor
- Grouping actors
- Communicating with groups of related actors


## Detailed design

### Technical Background

In both the actor model and XState, any actor ref can communicate with another actor ref as long as it has its `ActorRef` reference via `sendTo(actorRef, ...)`. It is possible to already implement the receptionist pattern in XState, but this involves a lot of wiring "boilerplate" and finding a way to pass through the receptionist actor ref to all spawned/invoked actors. This can be unfeasible for many use-cases, as it involves a lot of restructuring to accommodate the ad-hoc receptionist actor ref.

Instead, we seek a built-in way to define a _system_ which will define a root machine and hold shared data (such as a registry of actor refs) for all of its actors, as well as abstractions to interact with the `system.receptionist` for registering and discovering system-wide actor refs. 


### Implementation


**Creating a system**

The `createSystem(rootMachine, options?)` function will create an `ActorRef` whose behavior is defined by the `rootMachine`, which is a normal machine created from `createMachine(...)`. The purpose of the system is to provide `{ system }` to all machines (including the root) spawned in this system, which contains system-wide resources.

```ts
import { createSystem, createMachine } from 'xstate';

const rootMachine = createMachine({
  // ...
});

// Creating a system
const system = createSystem(rootMachine);

// Starting a system
system.start();
```

**Registering to the receptionist**

The `register(key)` action is made available from `'xstate/actions'`, which will register the executing actor to the system-wide registry under that `key`. Multiple actors can be registered under the same `key` by default, and it should be assumed that each `key` in the registry has 0 or more actor refs.

```ts
import { register } from 'xstate/actions';

const todoMachine = createMachine({
  // Register actor to the 'todo' key
  entry: register('todo'),
  // ...
});
```

This should be roughly equivalent to:
```ts
// PSEUDOCODE
entry: (ctx, e, { system, self }) => {
  system.register('todo', self);
}
```

**Getting listings**

The `system` value is made available in the 3rd argument to action/guard/invoke creators, which contains the `system.receptionist` that extends the `ActorRef` interface.

Listing all `ActorRef` objects registered under a key is done via `system.receptionist.list(key)`, which synchronously returns an array of `ActorRef` objects.

```ts
const todosMachine = createMachine({
  // ...
  on: {
    'todos.logAll': {
      actions: (ctx, e, { system }) => {
        // Lists all todo ActorRefs
        console.log(system.receptionist.list('todo'));
      }
    },
    'todos.completeAll': {
      actions: pure((ctx, e, { system }) => {
        return system.receptionist.list('todo').map(todoRef => {
          return sendTo(todoRef, { type: 'complete' });
        });
      });
    }
  }
});
```

This should be roughly equivalent to:
```ts
// PSEUDOCODE
// system.receptionist.list('todo');
system.receptionist.getSnapshot().registry.get('todo') ?? []; // => ActorRef[]
```

**Single listing**

A common use-case of the receptionist will be to register and find a single `ActorRef` registered to a given `key`. This can be done via `system.registry.find(key)`, which will return `ActorRef | null`.

```ts
const todoMachine = createMachine({
  // ...
  on: {
    complete: {
      // `system.registry.find(key)` returns 0 or 1 ActorRef objects
      // TODO: ensure `sendTo(...)` handles null targets (dead-letter queue?)
      actions: sendTo((ctx, e, { system }) => system.receptionist.find('notifier'), {
        type: 'notify',
        message: 'Todo completed'
      })
    }
  }
});
```

This should be roughly equivalent to:
```ts
// PSEUDOCODE
// system.receptionist.get('notifier');
system.receptionist.getSnapshot().registry.get('notifier')?.[0] ?? undefined; // => ActorRef[]
```

**Subscribing to listings**

The registry may change over time, as actors get registered and unregistered from certain keys. The `system.receptionist.listener(key)` returns an `ActorRef` (subscribable by nature) which can be subscribed to for the specific purpose of listening to changes in `ActorRef` objects registered under a key. The emitted value is the complete `ActorRef[]` array.

```ts
const todosMachine = createMachine({
  invoke: {
    src: (ctx, e, { system }) => fromObservable(() => system.receptionist.listener('todo')),
    onEmit: {
      actions: (_, event) => {
        // ActorRef[] of 'todo' actor refs
        console.log(event.data); 
      }
    }
  }
});
```

This should be roughly equivalent to:
```ts
// PSEUDOCODE
// system.receptionist.listener('todo')
const receptionist = {
  onRegister(key, actorRef) {
    // Register actorRef
    // ...

    observers[key].forEach(observer => {
      observer.next(registry.get(key))
    });
  },

  listener(key) {
    // Return ActorRef that listens for registry updates on that key
    // ...
  }
}
```


**Unregistering**

At any time, an actor may unregister itself from the system-wide registry. This should notify listener `ActorRef`s that the list of actors under that key has changed.

```ts
const todoMachine = createMachine({
  // ...
  on: {
    delete: {
      actions: unregister('todo')
    }
  }
});
```

This should be roughly equivalent to:
```ts
// PSEUDOCODE
// unregister('todo')
exit: (context, event, { system, self }) => {
  system.unregister('todo', self);
}
```

## How we teach this

**New concepts:**
- Actor system
- System registry
- Receptionist
- `onEmit` for invocations (TODO: describe)

**Existing concepts:**
- Actor refs
- Actions
- Subscriptions

No guides will need to be re-organized nor altered. This should be taught after actors (spawning/invoking). There would be two new sections to teach:

- Actor systems (conceptually just a root-level machine that provides shared resources to actors)
- Actor discovery

This is _not_ required knowledge.

**Teaching:**

- Blog post (discovering actors with receptionists)
- Actor discovery guide


## Use cases

These examples may be contrived, but they should each serve a purpose to highlight challenges that the receptionist feature directly solves.

- An orders actor is registered, and another actor subscribes to the `ORDERS_UPDATED` event. Rather than sending the giant payload of all orders through with the event, a smaller payload such as just the changed item could be sent. If an actor needs more data, `system.receptionist.find('orders').getSnapshot().context.list` could be used. 
- Common info like auth or user profile data is often required across multiple machines, so referring to the auth/user actor context directly will be a direct line to the data. This should be more efficient than storing it all over the place and would keep queries up to date.

## Drawbacks

This introduces two "new" concepts to XState - systems and receptionists - but it solves many common pain-points in managing and communicating with actors in a straightforward way. Both of these concepts are not new to the actor model, but the additional concepts can feel burdensome to devs who are trying to learn all of XState.

To mitigate this, we should inform devs that XState can be used successfully without systems (orphaned actors by default) nor receptionists, and that the purpose of these two concepts is to _simplify_ common use-cases that are otherwise difficult and complicated to implement.

## Alternatives

TODO

## Unresolved questions

- System options: `createSystem(machine, options?)` (what should go here?)
- Default system? I.e. should `{ system }` always be defined when a machine is created without `createSystem(...)`?
  - `createMachine(...)` can be thought of as `createSystem(...)` without any shared resources
- Need to further explain the role of `createSystem`, especially how it can also solve other common problems such as specifying system-wide implementations to spawned actors without using factory functions
