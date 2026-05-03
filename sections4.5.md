# Section 4.5: React’s Resource Management and Value Discarding

One of the most important, yet least understood, aspects of `useMemo` is that it is a "hint," not a "guarantee." The official React documentation explicitly states that React may choose to discard a memoized value and re-calculate it during a future render to free up memory or to handle other internal resource constraints. This "value discarding" is an "ugly" reality that many developers ignore, assuming that once a value is memoized, it will remain in memory as long as its dependencies don't change.

The implications of value discarding are profound. If you rely on `useMemo` for "referential stability"—for example, to ensure an object passed to a child component is always the same reference to prevent re-renders—you are building on a shaky foundation. If React decides to discard your value, the reference will change, and the child will re-render, even if the underlying data is identical. In this way, memoization can be "non-deterministic" in its performance impact.

This design choice allows React to be resilient in low-memory environments. If a user has fifty tabs open and your application is using a massive amount of RAM for memoized data, React can "purge" those caches to prevent the browser from crashing. While this is a good "system-level" behavior, it is a "bad" behavior for a developer who is trying to fine-tune the performance of a critical UI component. You are essentially fighting a framework that has the power to undo your optimizations at any time.

Because values can be discarded, you must never use `useMemo` for any logic that *must* preserve its reference for correctness. For example, if you are initializing an external library (like a charting engine or a physical engine) that depends on a stable instance, `useMemo` is the wrong tool. You should use `useState` or `useRef` (with lazy initialization) to ensure that the instance is created exactly once and persists for the entire lifecycle of the component.

Value discarding also highlights the difference between "Caching" and "Memoization" in the React context. A traditional cache (like a Map) would hold onto its values until explicitly cleared. React's memoization is more like a "temporary storage" that the framework manages. This makes it unsuitable for complex "memoization-as-storage" patterns that are common in other functional programming environments.

The "ugly" side of this behavior is its unpredictability. There is no simple way to know *when* React will discard a value. It depends on the available memory, the complexity of the component tree, and the internal scheduling of the React version you are using. This makes "performance testing" a game of averages rather than a series of precise measurements, as the same code might perform differently under different environmental pressures.

Furthermore, value discarding interacts poorly with "Reference-based API" patterns. Some third-party libraries require you to pass a stable object reference to identify a specific "session" or "transaction." If React discards the memoized object and creates a new one, the library might think a new session has started, leading to broken state or duplicated work in the external system. This is a subtle, high-level bug that is notoriously difficult to track down.

The "discarding" behavior also applies to `useCallback`. While a function is smaller than a large data object, React still manages it within the same resource-balancing system. If your `useCallback` captures a large closure (as we discussed in Section 4.3), React is even more likely to want to discard it to reclaim that memory. This creates a "double-edged sword" where the more you need memoization for performance, the more likely the framework is to discard it to save memory.

Future versions of React and the React Compiler are expected to handle this resource management even more aggressively. The goal is to move toward a world where the framework "understands" the cost of every calculation and can make intelligent decisions about what to keep and what to re-calculate. Until then, developers are stuck in a middle ground where they must use a tool that doesn't fully guarantee the one thing it is supposed to provide: stable caching of work.

Ultimately, understanding value discarding is about managing expectations. You should use `useMemo` as an optimization to *improve* performance, not as a requirement for *correctness*. If your code breaks when a memoized value is re-calculated, your code is fundamentally flawed. Mastering React's resource management means writing code that is "resilient to re-calculation," accepting that the framework is the final arbiter of how and when memory is used.

```javascript
// DANGER: Relying on useMemo for correctness (BAD)
function BadStability({ externalLib }) {
  // If React discards this, 'instance' will be re-created, 
  // potentially breaking the external library connection.
  const instance = useMemo(() => new externalLib.Client(), []);

  return <div onClick={() => instance.doSomething()} />;
}

// BETTER: Using a Ref for guaranteed stability
function BetterStability({ externalLib }) {
  const instanceRef = useRef(null);
  
  // Lazy initialization
  if (instanceRef.current === null) {
    instanceRef.current = new externalLib.Client();
  }

  return <div onClick={() => instanceRef.current.doSomething()} />;
}

// The "Referential Stability" gamble
const data = useMemo(() => ({ id: 1 }), []);
// On render 1: data === { id: 1 } (Ref A)
// On render 100 (if memory is low): data === { id: 1 } (Ref B)
// Any child using React.memo(Child) will re-render when the ref changes!
```
