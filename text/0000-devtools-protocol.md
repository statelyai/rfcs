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
interface InspectedEventObject {
  name: string; // Event type
  data: AnyEventObject; // The actual event object
  origin?: string; // Session ID
  destination: string; // Session ID
  createdAt: number; // Timestamp
}

interface InspectedActorObject {
  actorRef: AnyActorRef;
  sessionId: string;
  parent?: string; // Session ID
  snapshot: any; 
  machine?: StateMachineDefinition; // This is originally StateNodeDefinition (renaming)
  events: InspectedEventObject[];
  createdAt: number; // Timestamp
  updatedAt: number; // Timestamp
  status: 0 | 1 | 2; // 0 = not started, 1 = started, 2 = stopped
}

interface ActorUpdate {
  sessionId: string;
  actorRef: AnyActorRef;
  snapshot: any;
  event: InspectedEventObject;
  status: 0 | 1 | 2; // 0 = not started, 1 = started, 2 = stopped
}

interface ActorRegistration {
  actorRef: AnyActorRef;
  sessionId: string;
  machine?: StateMachineDefinition;
  createdAt: number;
}

export interface XStateDevInterface {
  register: (actorRef: AnyActorRef) => void;
  unregister: (actorRef: AnyActorRef) => void;
  onRegister: (
    listener: (actorRegistration: ActorRegistration) => void
  ) => Subscription;
  actors: {
    [sessionId: string]: InspectedActorObject;
  };
  onUpdate: (listener: (update: ActorUpdate) => void) => Subscription;
}
```

Differences from XState v4:

```diff
 export interface XStateDevInterface {
   register: ...
   unregister: ...
   onRegister: () => ...
-  services: ...
+  actors: { ... }
+  onUpdate: { ... }
 }
```

- `onRegister()` provides listeners wtih `ActorRegistration` objects, which include:
  - The `actorRef` reference to the actor
  - The `sessionId` of the actor ref
  - The `machine` JSON-serializable state machine definition if the actor was created from a machine
  - The `createdAt` timestamp
- `services` is removed and replaced by `actors`, which is a mapping of session IDs to `InspectedActorObject` objects and can be services from machines or other actors.
- `InspectedEventObject` is introduced to include metadata about the event, such as its `origin` and timestamp (`createdAt`). This is similar to `SCXML.EventObject`.
- `onUpdate()` provides listeners with `ActorUpdate` objects, which include:
  - The `sessionId` of the updated actor
  - The `actorRef` reference to the actor
  - The `snapshot`, which is the latest observable state of the actor
  - The `event` as an `InspectedEventObject` which caused the update
  - The `status` of the actor: 0 = not started, 1 = started, 2 = stopped

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
      parent?: string;
      machine?: string; // JSON-stringified
      snapshot: string; // JSON-stringified
      createdAt: number;
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
  machine?: string; // JSON-stringified machine definition
  createdAt: number;
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
  event: InspectedEventObject;
  status: 0 | 1 | 2; // Actor status
}
```

When an actor's snapshot updates due to a state transition, the client is notified via an `'@xstate/inspect.update'` message. The status signifies whether the actor is not yet started, started, or stopped. The client may choose to perform some cleanup behavior when the actor stops.

-----

- **`@xstate/inspect.event`** (Inspector -> Client)

```ts
interface XStateInspectEventEvent {
  type: '@xstate/inspect.event';
  event: InspectedEventObject; // includes origin and destination
}
```

When an actor sends a message to another actor, the client is notified via an `'@xstate/inspect.message'` event. The `event` should contain both the `origin` (if known) and the `destination` (required) as session IDs.

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
