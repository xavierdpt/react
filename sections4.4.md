# Section 4.4: Misusing useMemo for Side Effects

One of the most dangerous "bad" practices in React is the misuse of `useMemo` to perform side effects. `useMemo` is designed for "pure" calculations—given a set of inputs, it should return a value without changing anything else in the world. However, because `useMemo` runs during the "render phase" and before `useEffect`, developers are often tempted to use it for things like triggering an API call, logging to a server, or modifying a ref, believing it will be faster or more "inline" than using an effect.

The "ugly" reality is that React provides no guarantee that the `useMemo` factory function will only run once per dependency change. In fact, React reserves the right to discard the memoized value and re-run the factory function at any time to free up resources or to handle concurrent rendering. If your factory function has side effects—like pushing an event to an analytics tracker—those effects might fire multiple times for a single logical update, leading to duplicate data and broken business logic.

Furthermore, the render phase is supposed to be "pure." By introducing side effects into `useMemo`, you are violating a core architectural principle of React. During concurrent rendering, React might "start and stop" a render several times before finally committing it to the screen. If your `useMemo` has side effects, those effects will fire during the "incomplete" renders, even if the final UI never actually updates or is thrown away entirely.

Misusing `useMemo` for side effects also creates a "hidden" flow of logic that is difficult to trace. When you look at a component, you expect the side effects to be inside `useEffect`. If a side effect is tucked away inside a `useMemo` factory function, it's easy to overlook it during debugging or code review. It's a "surprising" behavior that breaks the mental model of most React developers, leading to a codebase that is unpredictable and fragile.

The temptation to use `useMemo` for side effects often comes from a desire to "avoid a flash of content." Because `useMemo` runs during render, it feels like a way to have your side effect happen "immediately." But side effects almost always involve asynchronous operations or DOM mutations, neither of which are safe to perform during the render phase. Attempting to "beat" the lifecycle by using `useMemo` is a short-sighted optimization that leads to long-term stability issues.

Logging and analytics are the most common victims of this pattern. A developer might write `useMemo(() => { trackView(id); return id; }, [id])`. On the surface, it seems clever: only track the view when the ID changes. But if React re-renders that component due to a parent update, or if it runs a "pre-render" in concurrent mode, your analytics will be artificially inflated. You are trading data integrity for a slightly cleaner-looking hook call.

Another "ugly" side effect of this misuse is its impact on the React Compiler and future optimizations. React's internal optimizations—like "Automatic Memoization"—assume that the code inside your components is pure. If you hide side effects in `useMemo`, the compiler might move, remove, or re-order that code in ways that break your side effects entirely. You are writing code that is "incompatible" with the future of the framework.

The interaction with "Strict Mode" also reveals the flaws in this pattern. In development, React's Strict Mode intentionally double-invokes certain lifecycle functions, including the `useMemo` factory, to help you find impure code. If your `useMemo` has side effects, you will see them happen twice in development. Many developers "ignore" this as a framework quirk, but it is actually a warning that your code is not resilient to the realities of modern React rendering.

Finally, misusing `useMemo` for side effects makes unit testing a nightmare. Standard testing utilities expect side effects to happen in the commit phase (captured by `act()`). If your side effects happen during render, they can be difficult to trigger reliably in a test environment, leading to "flaky" tests that pass or fail based on the internal scheduling of the test runner's virtual DOM.

Ultimately, the rule is simple: if it has a side effect, it belongs in `useEffect` (or `useLayoutEffect` if it must be synchronous with DOM changes). `useMemo` is for data transformation, not for interacting with the outside world. Respecting this boundary is essential for writing code that is predictable, testable, and compatible with the ongoing evolution of the React ecosystem.

```javascript
// The "Impure" useMemo (BAD)
function ImpureComponent({ id }) {
  // DANGER: Side effect (logging) inside useMemo!
  // This might run multiple times, or during a discarded render.
  const processedId = useMemo(() => {
    console.log("Processing ID for analytics:", id);
    return `ID-${id.toUpperCase()}`;
  }, [id]);

  return <div>{processedId}</div>;
}

// The "Pure" and Correct Way (BETTER)
function PureComponent({ id }) {
  const processedId = useMemo(() => {
    return `ID-${id.toUpperCase()}`;
  }, [id]);

  // Side effects belong in useEffect
  useEffect(() => {
    console.log("ID changed, tracking view:", id);
  }, [id]);

  return <div>{processedId}</div>;
}

// The "Pre-fetching" Trap (DANGEROUS)
const data = useMemo(() => {
  // DANGER: Triggering a fetch during render!
  fetchData(id); 
  return null;
}, [id]);
```
