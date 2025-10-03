
## **Part 2: Building useEffect from Scratch**

### **The Core Challenge**

```javascript
// When we write:
function DataFetcher({ userId }) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetchUser(userId).then(setData);
    
    return () => {
      // Cleanup
      cancelRequest();
    };
  }, [userId]); // Only re-run if userId changes
  
  return <div>{data?.name}</div>;
}

// Questions to answer:
// 1. When does the effect run? (After paint)
// 2. How does React compare dependencies?
// 3. When does cleanup run?
// 4. What happens with empty dependency array []?
```

### **Implementation: Basic useEffect**

```javascript
const MyReact = (function() {
  const componentStates = new WeakMap();
  let currentComponent = null;
  
  function useState(initialValue) {
    // ... (same as before)
  }
  
  function useEffect(callback, dependencies) {
    const componentState = getOrCreateComponentState(currentComponent);
    const currentIndex = componentState.hookIndex;
    
    // Get previous effect hook (if exists)
    const previousEffect = componentState.hooks[currentIndex];
    
    // Check if dependencies changed
    const hasChanged = !previousEffect || 
      !dependencies ||
      dependencies.length === 0 ||
      !areDependenciesEqual(previousEffect.dependencies, dependencies);
    
    // Store effect
    componentState.hooks[currentIndex] = {
      callback,
      dependencies,
      cleanup: previousEffect?.cleanup, // Previous cleanup function
      hasChanged
    };
    
    componentState.hookIndex++;
  }
  
  // Helper: Shallow comparison of dependency arrays
  function areDependenciesEqual(prevDeps, nextDeps) {
    if (prevDeps === null || nextDeps === null) {
      return false;
    }
    
    if (prevDeps.length !== nextDeps.length) {
      return false;
    }
    
    // Shallow comparison using Object.is
    for (let i = 0; i < prevDeps.length; i++) {
      if (!Object.is(prevDeps[i], nextDeps[i])) {
        return false;
      }
    }
    
    return true;
  }
  
  function render(component) {
    currentComponent = component;
    const componentState = getOrCreateComponentState(component);
    
    // Reset hook index
    componentState.hookIndex = 0;
    
    // Call component (this calls useEffect during render)
    const output = component();
    
    // After render: Execute effects (simulating commit phase)
    executeEffects(componentState);
    
    return output;
  }
  
  function executeEffects(componentState) {
    // This should be async (after browser paint)
    // We'll use setTimeout to simulate
    setTimeout(() => {
      componentState.hooks.forEach((hook, index) => {
        if (hook && hook.callback && hook.hasChanged) {
          // Run cleanup from previous effect
          if (hook.cleanup && typeof hook.cleanup === 'function') {
            console.log(`Running cleanup for effect ${index}`);
            hook.cleanup();
          }
          
          // Run new effect
          console.log(`Running effect ${index}`);
          const cleanup = hook.callback();
          
          // Store cleanup for next time
          hook.cleanup = cleanup;
          hook.hasChanged = false; // Mark as executed
        }
      });
    }, 0);
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
  
  return { useState, useEffect, render };
})();

// Test it!
function UserProfile({ userId }) {
  const [user, setUser] = MyReact.useState(null);
  const [loading, setLoading] = MyReact.useState(false);
  
  MyReact.useEffect(() => {
    console.log('Effect: Fetching user', userId);
    setLoading(true);
    
    // Simulate API call
    const timer = setTimeout(() => {
      setUser({ id: userId, name: `User ${userId}` });
      setLoading(false);
    }, 100);
    
    // Cleanup function
    return () => {
      console.log('Cleanup: Cancelling fetch for', userId);
      clearTimeout(timer);
    };
  }, [userId]); // Only re-run when userId changes
  
  return { user, loading };
}

// Initial render
UserProfile.userId = 1;
MyReact.render(UserProfile);
// Logs: "Effect: Fetching user 1"

// Re-render with same userId
setTimeout(() => {
  MyReact.render(UserProfile);
  // Effect doesn't run (dependencies haven't changed)
}, 200);

// Re-render with different userId
setTimeout(() => {
  UserProfile.userId = 2;
  MyReact.render(UserProfile);
  // Logs: "Cleanup: Cancelling fetch for 1"
  // Logs: "Effect: Fetching user 2"
}, 400);
```

### **Key Implementation Details**

**1. Dependency Comparison (Shallow Equality)**

```javascript
// This is why these scenarios behave differently:

// ✅ Primitive values work correctly
useEffect(() => {
  console.log('Runs when count changes');
}, [count]); // number is compared by value

// ❌ Object references always "change"
useEffect(() => {
  console.log('Runs EVERY render!');
}, [{ userId: 1 }]); // New object reference each render

// ✅ Fix: Stable reference or primitive
const config = useMemo(() => ({ userId: 1 }), []);
useEffect(() => {
  console.log('Runs once');
}, [config]);

// Better:
useEffect(() => {
  console.log('Runs when userId changes');
}, [userId]); // Just use the primitive
```

**2. Effect Timing (Critical Difference from Lifecycle)**

```javascript
// useEffect runs AFTER paint (async)
function Component() {
  useEffect(() => {
    console.log('3. useEffect runs');
  });
  
  console.log('1. Render');
  
  return <div ref={() => console.log('2. DOM updated')}>Hello</div>;
}

// Output:
// 1. Render
// 2. DOM updated
// 3. useEffect runs (after browser paints)

// This is different from componentDidMount (which runs before paint)
```

---
