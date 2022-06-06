- **Start Date:** 2022-06-06
- **RFC PR:**

# Computed Context

## Summary

The on-demand computation of context values from other context values, or even the entire state, enables the usage of these computations as first-class citizens in the state machine, as well as the state. This RFC proposes the addition of the `computed: { ... }` property to specify getters for computed values with minimal extra API overhead.

## Motivation

**Vue computed properties**

https://vuejs.org/guide/essentials/computed.html

**Pinia Getters**

https://pinia.vuejs.org/core-concepts/getters.html

<details>
<summary>
View example
</summary>

```ts
export const useStore = defineStore('main', {
  state: () => ({
    counter: 0,
  }),
  getters: {
    doubleCount(state) {
      return state.counter * 2
    },
    doublePlusOne(): number {
      return this.doubleCount + 1
    },
  },
})
```

```html
<template>
  <p>Double count is {{ store.doubleCount }}</p>
</template>
```

</details>

**Zag**

https://zagjs.com/overview/whats-a-machine

<details>
<summary>
View example
</summary>

```js
createMachine({
  context: {/* ... */},
  computed: {
    isInteractive: (ctx) => !(ctx.disabled || ctx.readonly),
    isHovering: (ctx) => ctx.hoveredValue > -1,
  },
  // ...
});

// Reading
const value = state.context.isHovering ? state.context.hoveredValue : state.context.value;
```

</details>

**MobX State Tree derived values**

https://mobx-state-tree.js.org/concepts/views

<details>
<summary>
View example
</summary>

```ts
import { autorun } from "mobx"

const UserStore = types
    .model({
        users: types.array(User)
    })
    .views(self => ({
        get numberOfChildren() {
            return self.users.filter(user => user.age < 18).length
        },
        numberOfPeopleOlderThan(age) {
            return self.users.filter(user => user.age > age).length
        }
    }))

const userStore = UserStore.create(/* */)

// Every time the userStore is updated in a relevant way, log messages will be printed
autorun(() => {
    console.log("There are now ", userStore.numberOfChildren, " children")
})
autorun(() => {
    console.log("There are now ", userStore.numberOfPeopleOlderThan(75), " pretty old people")
})
```

</details>

**MobX Computeds**

https://mobx.js.org/computeds.html

<details>
<summary>
View example
</summary>

```ts
import { makeObservable, observable, computed, autorun } from "mobx"

class OrderLine {
    price = 0
    amount = 1

    constructor(price) {
        makeObservable(this, {
            price: observable,
            amount: observable,
            total: computed
        })
        this.price = price
    }

    get total() {
        console.log("Computing...")
        return this.price * this.amount
    }
}

const order = new OrderLine(0)

const stop = autorun(() => {
    console.log("Total: " + order.total)
})
// Computing...
// Total: 0

console.log(order.total)
// (No recomputing!)
// 0

order.amount = 5
// Computing...
// (No autorun)

order.price = 2
// Computing...
// Total: 10

stop()

order.price = 3
// Neither the computation nor autorun will be recomputed.
```

</details>

## Detailed design

### Technical Background

Currently, `context` is how a state machine stores contextual data for its state. Context values are assigned via `assign(...)` actions, and the assignments can reference other context values:

```js
actions: assign({
  fullName: (context) => `${context.firstName} ${context.lastName}`
})
```

However, this is a manual and error-prone process, as it is easy to forget to do this whenever the relevant context values change. Assigning manually computed values is also not on-demand, and some of these computations can be expensive. 

Another approach is to use a function:

```js
function fullName(context) {
  return `${context.firstName} ${context.lastName}`;
}

const machine = createMachine({
  // ...
  on: {
    someEvent: {
      guard: (context) => fullName(context).trim().length > 100,
      // ...
    }
  }
});
```

This leads to important parts of the logic (computing the `fullName`, in this case) are now completely external to the machine, which can be less portable and is not easy to visualize in the future. Also, caching must be done manually.

### Implementation

The `computed: { ... }` property on the machine definition is a key-value pair of computed context keys to a function that takes in `state` and returns the computed context value from that state.

Computed values _cannot_ be assigned to; they are read-only values.

Computed values _extend_ `context` values, so their keys cannot be the same as any `context` keys.

Just like getters, computed values are computed on _read_. Caching optimizations may be possible in the future: if we are able to determine which context dependencies a computed value has, we can invalidate the cached computed context value whenever any of those dependencies change on `assign(...)`.

**Specifying computed context**

```ts
const machine = createMachine({
  context: {
    firstName: '',
    lastName: ''
  },
  computed: {
    // Computed from non-computed context
    fullName: ({ context }) => `${context.firstName} ${context.lastName}`,
    // Computed from computed context
    isLongName: ({ context }) => context.fullName.length > 20,
    // Computed from state and context
    errorMessage: (state) => {
      if (state.hasTag('error')) {
        return state.context.fullName.trim().length === 0
          ? 'Invalid name'
          : 'Unknown error'
      }

      return null;
    },
    // Returns a function, e.g.:
    // `const greeting = state.context.greet('Hello')`
    greet: ({ context }) => (greeting: string) => `${greeting}, ${context.fullName}`
  }
});
```

**Reading computed context**

```js
// ...

const name = state.context.fullName; // computed from context.firstName and context.lastName

const greeting = state.context.greet('Hello');
```

**Usage in assign**

```ts
const machine = createMachine({
  context: {
    firstName: '',
    lastName: '',
    age: 0
  },
  computed: {
    fullName: ({ context }) => `${context.firstName} ${context.lastName}`,
    // ...
  },
  // ...
  entry: assign({
    firstName: (_, event) => event.value,
    age: (context, event) => {
      // This will be the full name *prior* to the assignment above
      // since it is within the same assign() call
      console.log(context.fullName);

      return event.value;
    }
  }),
  // ...
  exit: [
    assign({ firstName: (_, event) => event.value }),
    assign({
      age: (context, event) => {
        // This will be the full name *after* the previous assignment
        // since it is a subsequent assign() call
        console.log(context.fullName);

        return event.value;
      }
    })
  ],
  // ...
  on: {
    someEvent: {
      actions: assign({
        // ❌ This will be a type error
        fullName: 'something else'
      })
    }
  }
});
```

## How we teach this

- We add a new "Computed Context" section to the existing guide on machine context
- Computed context is introduced as "read-only context" where the computed values are determined from a function that takes in the `state` and returns the value.
- Teach that computed context should not be assigned to
- Teach that a computed context value can be a function, and show examples where this is useful
- Demonstrate when computed context should and should not be used:
  - Computed context should be used in place of normal context when a context value can be computed from other context values

## Drawbacks

- "I'd personally want a stronger reason than "convenience" for expanding the API (ie. adding to the amount of stuff I need to learn)" ([Discord](https://discord.com/channels/795785288994652170/977959576152449034/978117164697534485))
- "This train of thought runs the risk of reinventing MobX" ([Discord](https://discord.com/channels/795785288994652170/977959576152449034/978112797412061225))

More discussion: https://discord.com/channels/795785288994652170/977959576152449034

## Alternatives

- `someValue: (context) => { ... }` instead of `someValue: (state) => { ... }`?
  - Could be slightly more convenient, but loses the ability to compute a context value based on state

Some more alternatives were mentioned here: https://github.com/statelyai/xstate/discussions/2384

## Unresolved questions

TBD
