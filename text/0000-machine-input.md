- **Start Date:** 2022-05-23
- **RFC PR:** (leave this empty)

# Machine Input

## Summary

Input data is a very important concept in server-side workflows, and very useful for state machines + statecharts in general. The general idea is that a workflow (represented by a state machine) starts with input data, which can be from some sort of trigger. That data is then used to initialize the workflow with necessary contextual data, or the data itself may be transformed through the workflow.

This RFC proposes to add input data to the built-in `{ type: 'xstate.init' }` event object in XState, in order to avoid the awkward (and potentially unsafe) workarounds that currently need to be done to support this, and to enable better visualization for representing this input data semantically.

## Motivation

Currently, there are 3 ways to implicitly "provide" input data to an XState machine:

**Overriding `context`**

```js
const greetMachine = createMachine({
  context: {
    name: '' // Need to specify "empty" values, which is unintuitive
  },
  entry: (ctx) => { console.log(`Hello, ${ctx.name}!`); }
});

// Need to overwrite context with input values
// WARNING: in v4, this will overwrite context fully, not partially
interpret(greetMachine.withContext({
  name: 'David'
})).start();
```

Problems:
- Empty values need to be specified, since there is an _implicit_ expectation that these context values are available
- No separation between "private" context and "public" context meant _only_ for input
- Too easy to overwrite context values not intended to be overwritten
- Context values may be invalid, especially when machine interpreted in dynamic environments where type checking is not available/forgotten
- No indication of which properties are input properties
- Input values may not need to be long-lived after machine start, yet they reside in `context` persistently

**Machine factory**

```js
const createGreetMachine = (name) => createMachine({
  entry: (ctx) => { console.log(`Hello, ${ctx.name}!`); }
});

interpret(createGreetMachine('David')).start();
```

Problems:
- Machine definition is oblivious to any input properties
- Machine is dynamically created; cannot guarantee that the machine is the same shape every time (even though it should always be)
- Machine definition cannot be serialized; must be run ad-hoc rather than being directly interpreted
- Machine is no longer easily portable between different languages (future goal)
- No standard way of creating a machine factory

**Bespoke input event**

```js
const greetMachine = createMachine({
  on: {
    'GREET.INPUT': {
      actions: (_, event) => { console.log(`Hello, ${event.name}!`); }
    }
  }
});

const greetService = interpret(greetMachine).start();

greetService.send({ type: 'GREET.INPUT', name: 'David' });
```

Problems:
- Since this input event is bespoke, there is no built-in way of identifying input properties
- Input event _must_ be first event sent; this is not guaranteed
- Cannot use input data at semantic start of machine; e.g., not usable in `entry`
- No standard input event shape

-----

**Prior art:**

- [Data in AWS Step Functions (States Language)](https://states-language.net/spec.html#data):

> The interpreter passes data between states to perform calculations or to dynamically control the state machine’s flow. All such data MUST be expressed in JSON.
>
> When a state machine is started, the caller can provide an initial JSON text as input, which is passed to the machine's start state as input. If no input is provided, the default is an empty JSON object, `{}`. As each state is executed, it receives a JSON text as input and can produce arbitrary output, which MUST be a JSON text. [...]

- [Azure Logic Apps](https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-overview) have _triggers_, which are events (with input data) that start a logic apps workflow.
- [Google Cloud Platform Workflows](https://cloud.google.com/workflows/docs/passing-runtime-arguments) enable `params` that let you specify data in a workflow execution request
- [Pipedream](https://pipedream.com/docs/workflows/steps/triggers/#app-based-triggers) has triggers that can contain input data for the start of a workflow.
- [Argo workflow inputs](https://argoproj.github.io/argo-workflows/workflow-inputs/)
- [GitHub Actions inputs](https://docs.github.com/en/actions/creating-actions/metadata-syntax-for-github-actions#inputs)

## Detailed design

**Overall API**

This RFC proposes to extend `{ type: 'xstate.init' }`, which is always the first event that a machine receives, with the optional `data?: ...` property:

```ts
const greetingMachine = createMachine({
  schema: {
    // For type-safety; can be used with JSON Schema
    input: {} as { name: string }
  },
  entry: (_, event) => {
    // event.type is 'xstate.init'
    // event.data is { name: ... }
    console.log(`Hello, ${event.data.name}!`);
  }
});

// Non-interpreted environments
const greetingMachineWithInput = greetingMachine
  .withInput({ name: 'David' });

greetingMachineWithInput.initialState.event.data;
// { name: 'David' }

// Interpreted environments
interpret(greetingMachineWithInput).start();
// logs 'Hello, David!'
```

### Technical Background

The reason this was not implemented before is because SCXML does not define a separation between input data (params) and context data (datamodel). To provide "input data" to a statechart in SCXML, you provide [`<param>` values](https://www.w3.org/TR/scxml/#param) as [children inside of `<invoke>`](https://www.w3.org/TR/scxml/#N10FF5), which will overwrite the values in the `<datamodel>`.

The main problem with this is that there is no difference between datamodel values that should only be present at the initialization of a machine and those that can change (via `<assign>`) via transitions in the machine.

### Implementation

**Input (init) event**

```ts
interface InitEvent<TInputData> {
  type: 'xstate.init';
  data: TInputData
}
```

**`.withInput(...)` method**

`machine.withInput(input)`

Arguments:
- `input`: the input data to include in the `data` property of the `InitEvent`

Returns:
- `machineWithInput`: a cloned `StateMachine` instance with the input data specified

## How we teach this

- As a better alternative to `machine.withContext({...})`, which as been confusing for many
- As a new concept?

TODO

## Drawbacks

- The only trade-off with this would be introducing a "new" concept, although this is _not_ a completely new concept to XState (events with data); it's simply extending the initial event with `data`. This is also not a new concept for workflows in general, but rather a missing feature in XState.

## Alternatives

- This was originally proposed in a different form in this input PR: https://github.com/statelyai/xstate/pull/2779 . The design and goals of this PR were different, larger in scope, and for a different purpose than this proposal, which only seeks to enable providing input data _once_ - at the start of a machine.

## Unresolved questions

- Should `data` be an optional property of `InitEvent`?
- Should we enforce at runtime that input data was passed in? If so, how?
- Should input data instead be provided at interpretation time, rather than machine creation time (see below)?

```js
const greetMachine = createMachine({...});

interpret(greetMachine)
  .input({ name: 'David' })
  .start();

// .input() should _not_ be able to be called after starting service
```
