### **Implementation: useLayoutEffect (Synchronous Effects)**

```javascript
const MyReact = (function() {
  // ... previous code ...
  
  function useLayoutEffect(callback, dependencies) {
    // Same as useEffect but different execution timing
    const componentState = getOrCreateComponentState(currentComponent);
    const currentIndex = componentState.hookIndex;
    
    const previousEffect = componentState.hooks[currentIndex];
    const hasChanged = !previousEffect || 
      !dependencies ||
      !areDependenciesEqual(previousEffect?.dependencies, dependencies);
    
    componentState.hooks[currentIndex] = {
      callback,
      dependencies,
      cleanup: previousEffect?.cleanup,
      hasChanged,
      isLayoutEffect: true // Flag to run synchronously
    };
    
    componentState.hookIndex++;
  }
  
  function render(component) {
    currentComponent = component;
    const componentState = getOrCreateComponentState(component);
    componentState.hookIndex = 0;
    
    const output = component();
    
    // Execute layout effects SYNCHRONOUSLY (before paint)
    executeLayoutEffects(componentState);
    
    // Execute regular effects ASYNCHRONOUSLY (after paint)
    executeEffects(componentState);
    
    return output;
  }
  
  function executeLayoutEffects(componentState) {
    componentState.hooks.forEach((hook, index) => {
      if (hook?.isLayoutEffect && hook.hasChanged) {
        if (hook.cleanup) {
          hook.cleanup();
        }
        
        console.log(`Running layout effect ${index} (SYNC)`);
        hook.cleanup = hook.callback();
        hook.hasChanged = false;
      }
    });
  }
  
  return { useState, useEffect, useLayoutEffect, render };
})();

// When to use which:
function MeasureComponent() {
  const [height, setHeight] = useState(0);
  const ref = useRef();
  
  // ❌ useEffect: Causes flicker (measures after paint)
  useEffect(() => {
    setHeight(ref.current.offsetHeight);
    // Paint → User sees wrong height → Update → Repaint
  });
  
  // ✅ useLayoutEffect: No flicker (measures before paint)
  useLayoutEffect(() => {
    setHeight(ref.current.offsetHeight);
    // Measure → Update state → Paint once with correct height
  });
  
  return <div ref={ref} style={{ height }}>Content</div>;
}
```

### **Complete Implementation with Cleanup Handling**

```javascript
const MyReact = (function() {
  const componentStates = new WeakMap();
  let currentComponent = null;
  
  function useState(initialValue) {
    const componentState = getOrCreateComponentState(currentComponent);
    const currentIndex = componentState.hookIndex;
    
    if (componentState.hooks[currentIndex] === undefined) {
      componentState.hooks[currentIndex] = {
        type: 'state',
        value: initialValue
      };
    }
    
    const hook = componentState.hooks[currentIndex];
    
    const setState = (newValue) => {
      const value = typeof newValue === 'function'
        ? newValue(hook.value)
        : newValue;
      
      if (Object.is(hook.value, value)) {
        return;
      }
      
      hook.value = value;
      render(currentComponent);
    };
    
    componentState.hookIndex++;
    return [hook.value, setState];
  }
  
  function useEffect(callback, dependencies) {
    const componentState = getOrCreateComponentState(currentComponent);
    const currentIndex = componentState.hookIndex;
    const previousHook = componentState.hooks[currentIndex];
    
    const hasChanged = !previousHook ||
      !dependencies ||
      dependencies.length === 0 ||
      !areDependenciesEqual(previousHook.dependencies, dependencies);
    
    componentState.hooks[currentIndex] = {
      type: 'effect',
      callback,
      dependencies,
      cleanup: previousHook?.cleanup,
      hasChanged,
      isLayoutEffect: false
    };
    
    componentState.hookIndex++;
  }
  
  function useLayoutEffect(callback, dependencies) {
    const componentState = getOrCreateComponentState(currentComponent);
    const currentIndex = componentState.hookIndex;
    const previousHook = componentState.hooks[currentIndex];
    
    const hasChanged = !previousHook ||
      !dependencies ||
      !areDependenciesEqual(previousHook?.dependencies, dependencies);
    
    componentState.hooks[currentIndex] = {
      type: 'effect',
      callback,
      dependencies,
      cleanup: previousHook?.cleanup,
      hasChanged,
      isLayoutEffect: true
    };
    
    componentState.hookIndex++;
  }
  
  function areDependenciesEqual(prevDeps, nextDeps) {
    if (!prevDeps || !nextDeps || prevDeps.length !== nextDeps.length) {
      return false;
    }
    
    return prevDeps.every((dep, i) => Object.is(dep, nextDeps[i]));
  }
  
  function render(component) {
    currentComponent = component;
    const componentState = getOrCreateComponentState(component);
    componentState.hookIndex = 0;
    
    // Render phase
    const output = component();
    
    // Commit phase: Layout effects (sync)
    componentState.hooks.forEach((hook, index) => {
      if (hook?.type === 'effect' && hook.isLayoutEffect && hook.hasChanged) {
        if (hook.cleanup) {
          console.log(`Cleanup layout effect ${index}`);
          hook.cleanup();
        }
        console.log(`Run layout effect ${index}`);
        hook.cleanup = hook.callback();
        hook.hasChanged = false;
      }
    });
    
    // Passive effects (async)
    setTimeout(() => {
      componentState.hooks.forEach((hook, index) => {
        if (hook?.type === 'effect' && !hook.isLayoutEffect && hook.hasChanged) {
          if (hook.cleanup) {
            console.log(`Cleanup effect ${index}`);
            hook.cleanup();
          }
          console.log(`Run effect ${index}`);
          hook.cleanup = hook.callback();
          hook.hasChanged = false;
        }
      });
    }, 0);
    
    return output;
  }
  
  function unmount(component) {
    const componentState = componentStates.get(component);
    if (!componentState) return;
    
    // Run all cleanup functions
    componentState.hooks.forEach((hook, index) => {
      if (hook?.type === 'effect' && hook.cleanup) {
        console.log(`Final cleanup effect ${index}`);
        hook.cleanup();
      }
    });
    
    // Remove component state
    componentStates.delete(component);
  }
  
  function getOrCreateComponentState(component) {
    if (!componentStates.has(component)) {
      componentStates.set(component, {
        hooks: [],
        hookIndex: 0
      });
    }
    return componentStates.get(component);
  }
  
  return { useState, useEffect, useLayoutEffect, render, unmount };
})();
```

---

## **Part 3: Common Pitfalls and Edge Cases**

### **Pitfall 1: Stale Closures**

```javascript
function Counter() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      // ❌ BUG: 'count' is captured from first render
      // Always logs 0, even as count increases!
      console.log(count);
      setCount(count + 1); // Always sets to 1
    }, 1000);
    
    return () => clearInterval(interval);
  }, []); // Empty deps = effect runs once, captures count=0
  
  return <div>{count}</div>;
}

// Why this happens:
// Render 1: count=0, effect captures count=0
// Render 2: count=1, but effect still has old closure with count=0
// Render 3: count=1 (setCount(0+1)), effect still has count=0

// ✅ Fix 1: Functional updates
useEffect(() => {
  const interval = setInterval(() => {
    setCount(prev => prev + 1); // Always uses latest value
  }, 1000);
  
  return () => clearInterval(interval);
}, []);

// ✅ Fix 2: Include dependency
useEffect(() => {
  const interval = setInterval(() => {
    console.log(count); // Fresh closure each time count changes
    setCount(count + 1);
  }, 1000);
  
  return () => clearInterval(interval);
}, [count]); // Effect re-runs when count changes

// ✅ Fix 3: useRef for mutable value
function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  
  useEffect(() => {
    countRef.current = count; // Keep ref in sync
  });
  
  useEffect(() => {
    const interval = setInterval(() => {
      console.log(countRef.current); // Always latest
      setCount(countRef.current + 1);
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);
}
```

### **Pitfall 2: Dependency Array Mistakes**

```javascript
// ❌ Missing dependencies
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, []); // ESLint warning: Missing 'userId'
  
  // Bug: When userId prop changes, effect doesn't re-run
}

// ❌ Object/Array in dependencies
function DataFetcher() {
  const options = { userId: 1, include: 'posts' }; // New object every render!
  
  useEffect(() => {
    fetchData(options);
  }, [options]); // Effect runs EVERY render
}

// ✅ Extract primitives
function DataFetcher({ userId, includePosts }) {
  useEffect(() => {
    const options = { userId, include: includePosts ? 'posts' : null };
    fetchData(options);
  }, [userId, includePosts]); // Primitive dependencies
}

// ✅ Or use useMemo
function DataFetcher({ userId, includePosts }) {
  const options = useMemo(
    () => ({ userId, include: includePosts ? 'posts' : null }),
    [userId, includePosts]
  );
  
  useEffect(() => {
    fetchData(options);
  }, [options]); // Stable reference
}
```

### **Pitfall 3: Effect Execution Order**

```javascript
function Component() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    console.log('Effect 1');
    return () => console.log('Cleanup 1');
  });
  
  useEffect(() => {
    console.log('Effect 2');
    return () => console.log('Cleanup 2');
  });
  
  return <button onClick={() => setCount(count + 1)}>Click</button>;
}

// Initial render:
// "Effect 1"
// "Effect 2"

// After click (re-render):
// "Cleanup 1"  ← Previous render's cleanup
// "Cleanup 2"  ← Previous render's cleanup
// "Effect 1"   ← New render's effect
// "Effect 2"   ← New render's effect

// On unmount:
// "Cleanup 1"
// "Cleanup 2"
```

---
