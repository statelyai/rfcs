- **Start Date:** 2022-06-05
- **RFC PR:** 

# DevTools Protocol

## Summary

This RFC outlines the protocol for XState DevTools; specifically, it outlines the communication between the **inspector** (inspecting actors at their source) and the **client**(s) (listening to actor updates from the inspector). The clients are typically separate from the source of the actors, such as in another browser window, in browser devtools, on a desktop app via WebSockets, or a completely remote server.

## Motivation

The motivation of this RFC is to create a unified protocol for an **inspector** to communicate with a devtools **client**, whether the client lives as a browser devtools extension or an electron app or even a remote server.

## Detailed design

### Technical Background

See the current protocol here: https://xstate.js.org/docs/packages/xstate-inspect/#implementing 

### Implementation

**Global XState object**

`globalThis.__xstate__` is a global `XStateDevInterface` object:

```ts
interface InspectedActorObject {
  actorRef: AnyActorRef; // local-only
  sessionId: string;
  parentId?: string; // Session ID
  systemId?: string; // Session ID
  events: ActorCommunicationEvent[]; // events where targetId === sessionId
  definition?: string; // JSON-stringified machine definition or URL
  createdAt: number; // Timestamp
  updatedAt: number; // Timestamp, derived from latest update createdAt
  status: 0 | 1 | 2; // 0 = not started, 1 = started, 2 = stopped, derived from latest update status
}

interface ActorTransitionEvent {
  type: "@xstate.transition";
  id: string; // unique string for this actor update
  snapshot: any;
  event: AnyEventObject; // { type: string, ... }
  status: 0 | 1 | 2; // 0 = not started, 1 = started, 2 = stopped
  sessionId: string; 
  actorRef: AnyActorRef; // Only available locally
  sourceId?: string; // Session ID
  createdAt: string; // Timestamp
}

interface ActorCommunicationEvent {
  type: "@xstate.communication";
  id: string; // unique string for this event
  event: AnyEventObject; // { type: string, ... }
  sourceId?: string; // Session ID
  targetId: string; // Session ID, required
  createdAt: string; // Timestamp
}

interface ActorRegistrationEvent {
  type: "@xstate.registration";
  actorRef: AnyActorRef;
  sessionId: string;
  parentId?: string;
  systemId?: string;
  definition?: string; // JSON-stringified definition or URL
  createdAt: string; // Timestamp
}

export type InspectorActorRef = ActorRef<ActorTransitionEvent | ActorCommunicationEvent | ActorRegistrationEvent>;
```

Differences from XState v4:

Instead of `XStateDevInterface`, there is the `InspectorActorRef` which can receive `ActorTransitionEvent` or `ActorRegistrationEvent` events.

- The `ActorTransitionEvent` event is sent to the inspector when an actor's state changes.
- The `ActorRegistrationEvent` event is sent to the inspector when an actor is registered.

**Messages**

These messages are sent to and from inspector clients, which can be listening through different mechanisms, such as `window.addEventListener('message', ...)` or WebSockets:

- **`@xstate/inspect.connect`** (Client -> Inspector)

```ts
interface XStateInspectConnectEvent {
  type: '@xstate/inspect.connect';
  actorRef: AnyActorRef; 
}
```

The client sends an `'@xstate/inspect.connect'` message to the inspector (e.g. via `window.postMessage(...)`), and some adapter abstracts the actor ref for sending messages back to the client via the `actorRef` property. It is only expected that the client sends the event with only the type:

```json
{
  "type": "@xstate/inspect.connect"
}
```

as the inspector will adapt that message to include the `actorRef` for responding.

-----

- **`@xstate/inspect.connected`** (Inspector -> Client)

```ts
interface XStateInspectConnectedEvent {
  type: '@xstate/inspect.connected';
}
```

The inspector sends back an `'@xstate/inspect.connected'` message to the client to signal that a connection has been established. The client can use the lack of this message to retry listening, or perform some custom behavior after a timeout.

-----

- **`@xstate/inspect.actors`** (Inspector -> Client)
```ts
interface XStateInspectActorsEvent {
  type: '@xstate/inspect.actors';
  actors: {
    [sessionId: string]: {
      sessionId: string;
      parent?: string; // Session ID
      machine?: string; // JSON-stringified
      events: ActorTransitionEvent[];
      createdAt: number; // Timestamp
    }
  };
}
```

Upon connection, the inspector also sends an `'@xstate/inspect.actors'` message, which includes all the current information on inspected actors. This is useful for the client being able to visualize the actor hierarchy and/or list of actors even when the client has connected late.

-----

- **`@xstate/inspect.read`** (Client -> Inspector)

The client can also request an `'@xstate/inspect.actors'` event to be sent to it by sending an `'@xstate/inspect.read'` event. This is useful if the client goes out-of-sync.

```ts
interface XStateInspectReadEvent {
  type: '@xstate/inspect.read';
}
```

-----

- **`@xstate/inspect.actor`** (Inspector -> Client)

```ts
interface XStateInspectActorEvent {
  type: '@xstate/inspect.actor';
  sessionId: string;
  definition?: string; // JSON-stringified machine definition or URL
  createdAt: string;
}
```

When an actor is registered, connected clients are notified of its existence. An actor may or may not be a state machine actor; if they are, the state machine definition (JSON-stringified) is provided.

-----

- **`@xstate/inspect.update`** (Inspector -> Client)

```ts
interface XStateInspectUpdateEvent {
  type: '@xstate/inspect.update';
  sessionId: string;
  snapshot: string; // JSON-stringified snapshot
  event: AnyEventObject; // JSON-stringified
  sourceId?: string; // Session ID
  status: 0 | 1 | 2; // Actor status
}
```

When an actor's snapshot updates due to a state transition, the client is notified via an `'@xstate/inspect.update'` message. The status signifies whether the actor is not yet started, started, or stopped. The client may choose to perform some cleanup behavior when the actor stops.

-----

- **`@xstate/inspect.send`** (Client -> Inspector)

```ts
interface XStateInspectSendEvent {
  type: '@xstate/inspect.send';
  sessionId: string;
  event: AnyEventObject;
  createdAt: number;
}
```

The client may send events directly to inspected actors.


## How we teach this

This should primarly be specific to XState tooling; however, we may want to eventually write guides for developers wanting to create their own XState dev tools.

## Drawbacks

TODO

## Alternatives

TODO

## Unresolved questions

- Bikeshedding the event names
