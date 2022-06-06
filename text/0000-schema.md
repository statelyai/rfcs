- **Start Date:** (fill me in with today’s date, YYYY-MM-DD)
- **RFC PR:** (leave this empty)

# Machine Schema

## Summary

The `schema` property in the machine config is used to provide strong types throughout the machine's definition, without having to use generics or `createModel()`. This RFC aims to provide a complete specification of all of the machine parts that can be strongly typed in `schema`.

## Motivation

WIP

## Detailed design

### Technical Background

WIP

### Implementation

**Creating a schema**

```ts
import { createSchema } from 'xstate';

const schema = createSchema({
  context: {} as SomeContext,
  events: {} as SomeEvents,
  actions: {} as SomeActions,
  guards: {} as SomeGuards,
  delays: {} as SomeDelays,
  actors: {
    someSource: {} as SomeActor,
    anotherSource: {} as AnotherActor,
    // ...
  },
  children: {
    first: {} as FirstActor,
    second: {} as SecondActor,
    // ...
  }
});

const machine = createMachine({
  context: {
    // ... initial context
  } as SomeContext,
  schema,
  // ...
});
```

**Actions**

The `actions` types provided in the schema will be available for:

- State `entry: [ ... ]`
- State `exit: [ ... ]`
- Transition `actions: [ ... ]`
- Implementation `actions: { ... }`

**Guards**

The `guards` types provided in the schema will be available for:

- Transition `guard: ...`
- Implementation `guards: { ... }`

**Delays**

The `delays` types provided in the schema will be available for:

- State `after: { ... }`:

```ts
after: {
  // Autocompleted
  namedDelay: {/* ... */}
}
```

```ts
after: [
  // Autocompleted
  { delay: 'namedDelay', /* ... */ }
]
```

**Actors**

The `actors` types provided in the schema will be available for:

- Assignment `spawn(...)` calls:

  ```ts
  actions: assign({   
    thing: (_ctx, _e, { spawn }) => {
      // Autocompleted
      return spawn('namedActor', /* ... */)
    }
  })
  ```

- Invocations (by source):

```ts
invoke: {
  // Autocompleted
  src: 'namedActor',
  onDone: {
    actions: (ctx, event) => {
      // event.data strongly typed from actor schema type
    }
  }
}
```

**Children**

The `children` types provided in the schema will be available for:

- Invocations (by ID):

```ts
invoke: {
  // Autocompleted
  id: 'namedChild',
  // Autocompleted and type-checked to ensure it is compatible
  // with the specified child actor type(s)
  src: 'someSource'
}
```

- Sending to actors:

```ts
// Autocompleted
actions: sendTo('namedChild', (ctx, event) => {
  // Strongly typed to events actor can receive
  return {/* ... */}
})
```

## How we teach this

Documentation

WIP

## Drawbacks

WIP

## Alternatives

- Specifying types in generics

WIP

## Unresolved questions

- Which types are specified as a mapping, and which are not?
