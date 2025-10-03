## **Part 4: Advanced Patterns**

### **Custom Hook Using Our Implementation**

```javascript
// Build a custom hook on top of our useState/useEffect
function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = MyReact.useState(value);
  
  MyReact.useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);
    
    return () => {
      clearTimeout(timer);
    };
  }, [value, delay]);
  
  return debouncedValue;
}

// Usage
function SearchComponent() {
  const [query, setQuery] = MyReact.useState('');
  const debouncedQuery = useDebounce(query, 500);
  
  MyReact.useEffect(() => {
    if (debouncedQuery) {
      console.log('Searching for:', debouncedQuery);
      // performSearch(debouncedQuery);
    }
  }, [debouncedQuery]);
  
  return { query, setQuery };
}
```

### **Implementing useReducer**

```javascript
function useReducer(reducer, initialState) {
  const [state, setState] = MyReact.useState(initialState);
  
  const dispatch = (action) => {
    const newState = reducer(state, action);
    setState(newState);
  };
  
  return [state, dispatch];
}

// Usage
function counterReducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { count: state.count + 1 };
    case 'decrement':
      return { count: state.count - 1 };
    case 'reset':
      return { count: 0 };
    default:
      return state;
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });
  
  return {
    count: state.count,
    increment: () => dispatch({ type: 'increment' }),
    decrement: () => dispatch({ type: 'decrement' }),
    reset: () => dispatch({ type: 'reset' })
  };
}
```

---

## **Interview Talking Points Summary**

### **For useState:**

✅ **Key Points to Make:**
- *"useState relies on stable hook ordering, tracked by index in an array or linked list"*
- *"Each component instance has its own hooks array stored on the Fiber node"*
- *"setState uses Object.is for equality checking to avoid unnecessary re-renders"*
- *"Functional updates (setState(prev => ...)) ensure you always work with latest state"*
- *"Batching: Multiple setState calls in the same event handler are batched into one render"*

### **For useEffect:**

✅ **Key Points to Make:**
- *"useEffect runs asynchronously after paint, unlike componentDidMount"*
- *"Dependencies are compared using Object.is (shallow equality)"*
- *"Cleanup functions run before the next effect and on unmount"*
- *"Empty dependency array [] means 'run once', no array means 'run every render'"*
- *"useLayoutEffect is synchronous (blocks paint) for DOM measurements"*

### **Common Follow-Up Questions:**

**Q: "Why can't hooks be conditional?"**
**A:** *"Hooks rely on call order to maintain state consistency. If we conditionally call hooks, the index mapping between renders would break, causing state to be assigned to wrong hooks."*

**Q: "What's the difference between useEffect and useLayoutEffect?"**
**A:** *"useLayoutEffect fires synchronously after DOM mutations but before paint, blocking visual updates. Use it for DOM measurements or mutations that need to happen before the user sees the screen. useEffect fires asynchronously after paint, not blocking the browser."*

**Q: "How does React know which component is currently rendering?"**
**A:** *"React maintains a global 'current fiber' pointer. When a component function is called, React sets this pointer to that component's Fiber node. Hooks read from and write to this Fiber's memoizedState."*

**Q: "What happens if I compare objects in useEffect dependencies?"**
**A:** *"Object.is compares by reference, not value. A new object literal {} created each render will always fail equality check, causing the effect to run every render. Extract primitives or use useMemo to stabilize references."*

---

## **Code You Can Run in an Interview**

Here's a complete, runnable implementation you could code live:

```javascript
// Mini React implementation
const MiniReact = (() => {
  let state = [];
  let effects = [];
  let cursor = 0;
  
  const useState = (initialValue) => {
    const index = cursor;
    state[index] = state[index] ?? initialValue;
    
    const setState = (newValue) => {
      state[index] = newValue;
      render();
    };
    
    cursor++;
    return [state[index], setState];
  };
  
  const useEffect = (cb, deps) => {
    const index = cursor;
    const oldDeps = effects[index]?.deps;
    
    const hasChanged = !oldDeps || 
      !deps ||
      deps.some((d, i) => !Object.is(d, oldDeps[i]));
    
    if (hasChanged) {
      effects[index]?.cleanup?.();
      const cleanup = cb();
      effects[index] = { deps, cleanup };
    }
    
    cursor++;
  };
  
  const render = () => {
    cursor = 0;
    App();
  };
  
  return { useState, useEffect, render };
})();

// Test app
function App() {
  const [count, setCount] = MiniReact.useState(0);
  const [name, setName] = MiniReact.useState('Alice');
  
  MiniReact.useEffect(() => {
    console.log(`Effect: count is ${count}`);
    return () => console.log(`Cleanup: count was ${count}`);
  }, [count]);
  
  console.log(`Render: count=${count}, name=${name}`);
  
  // Simulate interactions
  if (count === 0) setTimeout(() => setCount(1), 100);
  if (count === 1) setTimeout(() => setName('Bob'), 200);
}

MiniReact.render();
```

This demonstrates the core concepts in ~40 lines of code that you could write on a whiteboard!

---
