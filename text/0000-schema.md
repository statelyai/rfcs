- **Start Date:** (fill me in with today’s date, YYYY-MM-DD)
- **RFC PR:** (leave this empty)

# Machine Schema

## Summary

The `schema` property in the machine config is used to provide strong types throughout the machine's definition, without having to use generics or `createModel()`. This RFC aims to provide a complete specification of all of the machine parts that can be strongly typed in `schema`.

## Motivation

> Why are we doing this? What use cases does it support?

> Please provide specific examples. If you say “this would be more flexible” then
> give an example of something that becomes easier. If you say “this would be make
> it easier to do X” then give an example of what that looks like today and what’s
> hard about it.

> Don’t assume others recognize the problem is one that needs to be solved
> Is there some concrete issue you cannot accomplish without this?
> What does it look like to accomplish some set of goals today and how could
> that be improved?
> Are there any workarounds that are necessary today?
> Are there open issues on Github where people would be helped by this?
> Will the change have performance impacts? Can you quantify them?

> Please focus on explaining the motivation so that if this RFC is not accepted,
> the motivation could be used to develop alternative solutions. In other words,
> enumerate the constraints you are trying to solve without coupling them too
> closely to the solution you have in mind.

## Detailed design

### Technical Background

> There are a lot of ways XState and Stately tools are used. They’re hosted on 
> different platforms; integrated with different libraries; built with different 
> bundlers, etc. No one person knows everything about all the ways XState and the 
> Stately tools are used. What does someone who knows about XState but hasn’t 
> necessarily used anything outside of it need to know? Are there docs you can 
> share?

> How do different libraries or frameworks implement this feature? We can take
> design inspiration from others who have done this well and improve upon the
> designs or make them better fit XState.

### Implementation

```ts
const machine = createMachine({
  context: {
    // ... initial context
  } as SomeContext,
  schema: {
    events: {} as SomeEvents,
    actions: {} as SomeActions,
    guards: {} as SomeGuards,
    delays: {} as SomeDelays,
    actors: {} as SomeActors,
    children: {
      first: {} as FirstActor,
      second: {} as SecondActor
    }
  },
  // ...
});
```0

## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Stately patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Stately guides must be
> re-organized or altered? Does it change how Stately is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Stately
> users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Stately,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the
> same domain have solved this problem differently.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still to be decided?
