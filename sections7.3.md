# Section 7.3: Tearing and Inconsistency in useSyncExternalStore

`useSyncExternalStore` is an "emergency" hook introduced to address the "Tearing" problem that occurs when external data sources are used in React's Concurrent Mode. Tearing happens when a component tree renders with a value from an external store, but that store changes mid-render, causing other components in the same tree to render with the new value. The "ugly" reality of this hook is that it is a "synchronous" fix for an "asynchronous" framework, representing a major compromise in React's concurrent rendering vision.

The primary "malice" of `useSyncExternalStore` is that it effectively disables concurrent rendering for any part of the UI that depends on the external store. When the store updates, React is forced to re-render the entire dependent tree synchronously and all at once to ensure consistency. This prevents React from "breaking up" the work or "interrupting" the render to handle higher-priority events. You've solved the "tearing" bug, but you've re-introduced the "blocking main thread" problem that Concurrent Mode was supposed to eliminate.

The "bad" practice associated with this hook is using it for everything "just to be safe." Developers who are afraid of tearing or who don't fully understand Concurrent Mode might use `useSyncExternalStore` to wrap all their state, even if that state is internal to React. This turns their "Concurrent React" application into a "Legacy Synchronous" application, negating all the performance benefits of the modern framework. It's a form of "fear-based" architectural regression.

The implementation of `useSyncExternalStore` is also notoriously "ugly." It requires you to provide a `subscribe` function and a `getSnapshot` function. The `getSnapshot` function must return a stable, immutable value for as long as the store hasn't changed. If you return a new object reference on every call to `getSnapshot`, you will trigger an infinite re-render loop. This puts a massive "referential stability" burden on the developer, making it one of the most dangerous hooks in the library.

Another "ugly" side of this hook is its impact on "Time-Travel Debugging." Because the store is external to React, the framework doesn't naturally "capture" the state history of the store. If you are using React's internal tools to debug your application, the external store's state will always be the "current" value, making it impossible to see what the UI looked like at a previous point in the session. You've broken the "unidirectional" data flow that made React so easy to reason about.

Furthermore, `useSyncExternalStore` creates a "partitioned" architecture. Some parts of your app (those using standard hooks) are concurrent and smooth, while other parts (those using external stores) are synchronous and potentially laggy. This "mixed-mode" execution makes performance profiling a nightmare, as the "slowness" of the app is no longer localized to specific components but is spread across different scheduling paradigms.

The "malice" of the hook also extends to library authors. If a state management library (like Redux or MobX) wants to be "Concurrent-safe," it *must* use `useSyncExternalStore`. But by doing so, it forces all its users into the synchronous rendering path. This has led to a significant "schism" in the React community between "React-native state" advocates and "External state" advocates, with each group having a different vision for the future of the framework's performance.

Hydration with `useSyncExternalStore` is another source of "bad" complexity. You must provide a third argument, `getServerSnapshot`, to ensure that the server-rendered HTML matches the client's initial state. If the external store's value on the client is different from what was used on the server (e.g., a value from `window` or `localStorage`), you will still get a hydration mismatch. You've replaced one "tearing" problem with another "hydration" problem.

The "shared burden" of this hook is the technical debt it represents. It is an admission that the "purely functional" model of React cannot handle the realities of mutable, external state. It is a "hack" that was added to the framework to prevent it from being incompatible with the existing ecosystem of state management libraries. Using it feels like "fighting" the framework's own philosophy just to get basic consistency.

Ultimately, mastering `useSyncExternalStore` means using it only as a last resort. It is for "External Systems"—like WebSockets, browser APIs, or legacy state stores—not for standard application state. The goal should be to move as much state as possible *into* React's managed systems so that you can leverage the full power of Concurrent Mode without the "ugly" compromises of synchronous synchronization.

```javascript
// The "Referential Stability" Trap (BAD)
const externalStore = { value: 0 };

function BadExternalComponent() {
  const value = useSyncExternalStore(
    subscribe,
    // DANGER: Returning a new object on every call!
    // This will trigger an infinite re-render loop.
    () => ({ val: externalStore.value }) 
  );
}

// The "Correct" but Boilerplate-heavy implementation
const store = {
  state: { count: 0 },
  listeners: new Set(),
  subscribe(callback) {
    this.listeners.add(callback);
    return () => this.listeners.delete(callback);
  },
  getSnapshot() {
    return this.state; // Stable reference to immutable state
  }
};

function GoodExternalComponent() {
  const state = useSyncExternalStore(
    (cb) => store.subscribe(cb),
    () => store.getSnapshot(),
    () => ({ count: 0 }) // Server snapshot
  );
  return <div>{state.count}</div>;
}
```
