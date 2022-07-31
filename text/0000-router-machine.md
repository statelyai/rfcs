- **Start Date:** (fill me in with today’s date, YYYY-MM-DD)
- **RFC PR:** (leave this empty)

# Router machines

## Summary

Using XState as a router is one of the most frequently requested use-cases. Whereas routes, by their nature, are freely routable and may not seem like a state machine, statecharts provide both a natural hierarchy for nested routes, guarded routes, and even parallel routes (such as showing 2 views side-by-side on the same page). Also, statecharts can be used to enforce transitions between routes, which are useful for wizard-like multi-step flows.

This RFC seeks to extend state node definitions with meta data for routing, which can be consumed by a 3rd-party routing library, and also used within the statechart itself to specify that a state can be transitioned to from any other state, which is a valid use-case.

```tsx
import { createMachine } from 'xstate';

const routerMachine = createMachine({
  initial: 'home',
  states: {
    home: {
      // A route object signifies that a state
      // can be navigated to via sending the
      // `navigateTo(...)`event creator
      route: { path: '/' },
    },
    login: {
      route: { path: '/login' },
    },
    items: {
      route: { path: '/items' },
      initial: 'all',
      states: {
        all: {
          route: {
            // no path
            schema: {
              params: {} as {
                per_page: number;
                sort_by: 'price' | 'date';
                dir: 'asc' | 'desc';
              },
            },
          },
        },
        single: {
          route: {
            path: '/:id',
          },
        },
      },
    },
    cart: {
      route: { path: '/cart' },
    },
    checkout: {
      route: { path: '/checkout' },
      initial: 'shipping',
      states: {
        shipping: {
          route: { path: '/shipping' },
          on: {
            next: 'billing',
          },
        },
        billing: {
          /* ... */
        },
        review: {
          /* ... */
        },
        receipt: {
          /* ... */
        },
      },
    },
  },
});

const routeData = routerMachine.parse('/items/123?details');
// {
//   value: {
//     items: 'all'
//   },
//   params: {
//     id: 123
//   },
//   query: {
//     details: true
//   }
// }
```

```tsx
import {
  getStateFromRoute,
  getRouteFromState,
  syncWithHistory,
} from 'xstate/router';

const shippingState = getStateFromRoute(routerMachine, '/checkout/shipping');

const shippingRoute = getRouteFromState(shippingState);

// Adding to JSX
<a href={shippingRoute}>Continue to shipping</a>;
<Link to={shippingRoute}>Continue to shipping</Link>;

const billingState = routerMachine.transition(shippingState, { type: 'next' });

// Syncing to history
const routerActor = interpret(routerMachine).start();

syncToHistory(routerActor, window.history);

// Calls `history.push(...)`
routerActor.send({ type: 'next' });

// Syncs machine state to URL
window.history.back();
```

```tsx
import { navigateTo } from 'xstate/router';

// Navigate directly to a route
routerActor.send(navigateTo('/checkout'));

routerActor.send(navigateTo('/items/123?order=asc'));

// Same as...
routerActor.send(
  navigateTo({
    path: '/items',
    params: {
      id: '123',
    },
    query: {
      order: 'asc',
    },
  })
);
```

**Usage with React Router**

```tsx
import * as React from 'react';
import { useRoutes } from 'react-router-dom';
import { createMachine } from 'xstate';
import { getRoutes } from 'xstate/router';

const routerMachine = createMachine({
  initial: 'home',
  states: {
    home: {
      route: { path: '/', element: <Dashboard /> },
      initial: 'messages',
      states: {
        messages: {
          route: {
            // default:
            // path: 'messages'
            element: <DashboardMessages />,
          },
          on: {
            viewTasks: 'tasks',
          },
        },
        tasks: {
          route: {
            element: <DashboardTasks />,
          },
        },
      },
    },
    team: {
      route: { element: <AboutPage /> },
    },
  },
});

function App() {
  let element = useRoutes(getRoutes(routerMachine));
  const routerActor = useInterpret(routerMachine);

  routerActor.send('viewTasks');

  routerActor.push('/team');

  return element;
}
```

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

> Explain the design in enough detail for somebody familiar with the framework to
> understand, and for somebody familiar with the implementation to implement. Any
> new terminology should be defined here.

> Explain not just the final design, but also how you arrived at it. What
> constraints did you face? Are there corner cases you’ve come up with solutions for?

> Explain how your design fits into the larger picture. Are the other open problems
> in this area you’re familiar with? How does this design fit with potential
> solutions for those issues?

> Connect your design to the motivations you listed above. When describing a part of
> the design, it can be useful to share an example of what it would look like to
> utilize the implementation as solution to the problem.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Stately patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Stately guides must be
> re-organized or altered? Does it change how Stately is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Stately
> users?

## How do we type this

> Especially if the RFC proposes adding/changing a runtime feature of the XState itself
> we should think about strong typing. Can the feature be strictly typed? If yes, is it complicated?
> Can a proof of concept be prepared on the [TS playground](https://www.typescriptlang.org/play)?

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Stately,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the
> same domain have solved this problem differently.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still to be decided?
