# Section 6.3: Mixing Concurrent and Synchronous Logic

The most "ugly" bugs in modern React often occur when "Concurrent" features (like transitions) are mixed with "Synchronous" logic (like standard event handlers or third-party libraries that expect immediate updates). React's concurrent mode assumes that renders are "interruptible" and "discardable," but many JavaScript patterns and legacy libraries assume that once an update starts, it will finish synchronously and uniquely. When these two worlds collide, you end up with "Impossible States" and "Tearing."

"Tearing" is a phenomenon where different parts of the UI show inconsistent values for the same piece of external data. This is especially common when using global state libraries or native browser APIs (like `window.innerWidth`) that change independently of React's render cycle. If a concurrent render is interrupted, and the external data changes before the render resumes, the "second half" of the render will use a different value than the "first half." The result is a single UI frame that is internally inconsistent—a "tear" in the user experience.

The "ugly" side of mixing logic is that it often breaks "Controlled Components." If you wrap a state update for a text input in a transition to keep the UI smooth, but the input itself is a "controlled" component (using `value` and `onChange`), you will experience a "laggy" or "stuttering" typing experience. This happens because the "urgent" update to the input's value is fighting with the "non-urgent" transition that is trying to re-render the same input. The input's cursor might jump to the end, or characters might be dropped entirely.

Furthermore, many third-party libraries—especially those that perform their own DOM mutations or use their own internal state (like D3, Google Maps, or jQuery-based plugins)—are completely "Concurrent-incompatible." They expect the React component tree to be a stable, synchronous representation of the UI. If React starts a concurrent render, stops it, and then discards it, these libraries might be left in a "half-initialized" or "corrupted" state, leading to hard-to-debug crashes or visual glitches.

To combat tearing, React introduced the `useSyncExternalStore` hook. This hook is an "ugly" but necessary admission that the concurrent model is incompatible with simple external state. It forces React to treat updates from a specific external store as synchronous, effectively "opting out" of concurrent mode for that specific data source. While it fixes the tearing, it also means you lose the benefits of concurrent rendering for any UI that depends on that store.

Another "bad" practice is mixing synchronous side effects with concurrent transitions. If you have a `useEffect` that performs an action based on a state variable, and that variable is updated inside a transition, the effect will only run after the transition *finally* commits. If the transition takes several seconds, your side effect is delayed. If the user performs another action in the meantime, your side effect might fire with data that is already obsolete. The "ordering" of effects becomes a complex, non-deterministic puzzle.

The "ugly" truth of concurrent development is that it requires "poisoning" your codebase with special hooks. You cannot simply use `useState` anymore; you have to decide for every single update whether it should be urgent or a transition. You have to decide for every external data source whether it needs `useSyncExternalStore`. This "priority-based" programming is significantly more complex than the simple "state-triggers-render" model of early React.

Testing mixed logic is notoriously difficult. A test might pass if it's executed synchronously, but fail if it's executed in a way that triggers concurrent scheduling. Because the behavior depends on the internal timing of the React scheduler, it can be "flaky"—passing on one CI run and failing on another without any changes to the code. This non-determinism is the enemy of a reliable deployment pipeline.

Furthermore, mixing logic can lead to "Double Renders" that are hard to avoid. If a synchronous update triggers a concurrent transition (or vice versa), React might end up performing two full render cycles where one would have sufficed. On a complex page, these "unnecessary" renders can negate any performance gains you hoped to achieve with concurrent mode, leaving you with an app that is both more complex and slower.

Ultimately, mastering "Mixed Logic" requires a defensive approach to framework features. You should only use concurrent features when you have a specific performance problem to solve, and you must be extremely mindful of how those features interact with the "synchronous" parts of your application. Whether through the careful use of `useSyncExternalStore` or by isolating concurrent-heavy logic in specific branches of the tree, the goal is a stable, consistent UI in a world that is increasingly non-linear.

```javascript
// The "Tearing" Example (BAD)
let externalState = { count: 0 };
// Imagine this changes outside React (e.g., a WebSocket)
setInterval(() => { externalState.count++ }, 50);

function TearingComponent() {
  // DANGER: Reading external state during a concurrent render
  // Part of the tree might read '5', then an interrupt happens,
  // then the rest of the tree reads '6'. UI TEARING!
  const currentCount = externalState.count;
  
  return (
    <div>
      <SlowHeader count={currentCount} />
      <SlowFooter count={currentCount} />
    </div>
  );
}

// The "Fixed" version with useSyncExternalStore
function StableComponent() {
  const count = useSyncExternalStore(
    (callback) => {
      // Subscribe to external changes
      const id = setInterval(callback, 50);
      return () => clearInterval(id);
    },
    () => externalState.count // Get snapshot
  );

  return <div>{count}</div>;
}

// The "Controlled Input" Trap
function LaggyInput() {
  const [val, setVal] = useState("");
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    const next = e.target.value;
    // DANGER: Trying to make the input update "non-urgent"
    // This will lead to stuttering and cursor jumping.
    startTransition(() => {
      setVal(next);
    });
  };

  return <input value={val} onChange={handleChange} />;
}
```
