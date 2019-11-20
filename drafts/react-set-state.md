### When is setState asynchronous
Currently, setState is asynchronous inside event handlers.

This ensures, for example, that if both Parent and Child call setState during a click event, Child isn’t re-rendered twice. Instead, React “flushes” the state updates at the end of the browser event. This results in significant performance improvements in larger apps.

This is an implementation detail so avoid relying on it directly. In the future versions, React will batch updates by default in more cases.

### Why doesn’t React update this.state synchronously

As explained in the previous section, React intentionally “waits” until all components call setState() in their event handlers before starting to re-render. This boosts performance by avoiding unnecessary re-renders.

### why doesn’t React just update this.state immediately without re-rendering

There are two main reasons:

This would break the consistency between props and state, causing issues that are very hard to debug.
This would make some of the new features we’re working on impossible to implement.

https://github.com/facebook/react/issues/11527#issuecomment-360199710

### Does React keep the order for state updates?
