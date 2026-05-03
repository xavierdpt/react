# Section 2.4: Race Conditions in Data Fetching

Race conditions are a classic problem in distributed systems and asynchronous programming, but they manifest with a particular "ugly" intensity within the React `useEffect` model. Because effects are triggered by state changes and run asynchronously, multiple effects can be "in flight" at the same time. In the context of data fetching, this means that a user navigating through several pages or changing search filters rapidly can trigger multiple network requests, the responses of which may arrive out of order, leading to a UI that displays the wrong data for the current state.

The fundamental issue is that `useEffect` provides no built-in mechanism for "ignoring" or "cancelling" a previous execution's asynchronous results. When a second effect starts because a dependency changed, the first effect's promise is still running. When that first promise finally resolves, it will call its `then()` block and update the component's state. If the second promise has already resolved or is still pending, the "stale" data from the first request will overwrite the "fresh" data, leaving the user with a confusing or incorrect display.

This problem is often hidden by fast network connections during local development. A developer might click through filters and see everything working perfectly. But in a real-world scenario with variable latency, a request for "Category A" might take 5 seconds while a subsequent request for "Category B" takes only 1 second. The UI will briefly show B, then "snap back" to show A when the slower request finishes. This jittery behavior is a hallmark of unhandled race conditions.

To fix this "ugly" side of fetching, developers must implement a manual "ignore" pattern using the `useEffect` cleanup function. By defining a boolean variable (often named `ignore` or `isMounted`) and setting it to `true` in the cleanup function, you can ensure that the async callback only updates state if it's still the "current" execution of the effect. While effective, this pattern adds significant boilerplate to every single data-fetching effect in an application, increasing the likelihood of human error.

Another approach is to use `AbortController`, a web standard that allows you to actually signal the browser to cancel an ongoing network request. Integrating `AbortController` into `useEffect` is the "correct" way to handle cancellation, but it is syntactically verbose and requires careful coordination between the effect's body and its cleanup function. Many developers avoid it because of the extra complexity, opting for the simpler but less efficient "ignore" pattern.

The problem of race conditions also extends beyond simple `fetch` calls. Any side effect that involves a sequence of asynchronous steps—such as fetching a list, then fetching details for each item, then merging them—is susceptible to being interrupted at any point. If the component re-renders and triggers a new effect while these steps are mid-way, the partial results from the previous "run" can leak into the state of the current one, causing data corruption that is notoriously hard to debug.

This fragility is one of the main reasons why the React community has largely shifted away from "useEffect-based fetching" toward specialized libraries like `TanStack Query` (formerly React Query) or `SWR`. These libraries handle the complexities of caching, deduplication, and race condition management internally. However, relying on these libraries means introducing a major third-party dependency just to perform what should be a fundamental task: fetching data and showing it on the screen.

The "ugly" truth is that `useEffect` was never actually designed for data fetching. It was designed for "synchronizing" the component's state with an external system. Using it for fetching is a form of "misuse" that has become the standard pattern because there was no better alternative for several years. React's upcoming "Suspense for Data Fetching" and the `use` hook are attempts to solve this, but they introduce their own set of architectural constraints and "bad/ugly" trade-offs.

Race conditions also reveal the limitations of the "loading state" pattern. If you only have a single `isLoading` boolean, it's impossible to tell which request is currently loading. A race condition might cause the `isLoading` flag to be set to `false` when a stale request finishes, even though a fresh request is still pending. This leads to a UI that shows "No data found" while actually waiting for the correct data to arrive.

Ultimately, handling race conditions requires a shift from "imperative fetching" to "managed synchronization." You must assume that any asynchronous operation will be interrupted and ensure that your code is resilient to out-of-order responses. This level of defensive coding is a significant "tax" on productivity and is a direct consequence of the mismatch between the asynchronous nature of the web and the synchronous nature of React's rendering cycle.

```javascript
// The Race Condition Bug
function SearchResults({ query }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    // DANGER: If query changes fast, multiple fetches will be in flight.
    // The last one to FINISH wins, not the last one to START.
    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(setData); 
  }, [query]);

  return <div>{JSON.stringify(data)}</div>;
}

// The "Ignore" Pattern Fix
function FixedResults({ query }) {
  const [data, setData] = useState(null);

  useEffect(() => {
    let ignore = false; // Local variable to this execution

    fetch(`/api/search?q=${query}`)
      .then(res => res.json())
      .then(result => {
        if (!ignore) {
          setData(result);
        }
      });

    return () => {
      ignore = true; // Set to true when the effect is cleaned up
    };
  }, [query]);

  return <div>{JSON.stringify(data)}</div>;
}

// The "AbortController" Fix (Best Practice)
useEffect(() => {
  const controller = new AbortController();
  
  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then(res => res.json())
    .then(setData)
    .catch(err => {
      if (err.name === 'AbortError') return; // Expected
      console.error(err);
    });

  return () => controller.abort();
}, [query]);
```
