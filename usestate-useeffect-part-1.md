# Implementing useState and useEffect from Scratch

Excellent question! This tests whether you understand React's mental model beyond just using the APIs. Let me coach you through building simplified versions of these hooks that demonstrate the core concepts.

---

## **The Mental Model First**

Before we code, let's establish the key insights you need to communicate:

**Interview Talking Points:**
- *"Hooks rely on a stable call order, which is why we can't use them conditionally"*
- *"React maintains a hidden state array and cursor for each component"*
- *"Closures are fundamental—hooks capture values from specific renders"*
- *"useEffect dependencies determine when effects re-run via shallow comparison"*

---

## **Part 1: Building useState from Scratch**

### **The Core Challenge**

```javascript
// When we write:
function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('Alice');
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}

// Questions to answer:
// 1. Where is state stored between renders?
// 2. How does useState know WHICH state to return?
// 3. How does setCount trigger a re-render?
// 4. Why does each component instance have separate state?
```

### **Implementation: Version 1 - Single Component**

Let's start simple and build up:

```javascript
// Our mini React implementation
const MyReact = (function() {
  // Global state for the currently rendering component
  let currentComponent = null;
  let hooks = [];
  let hookIndex = 0;
  
  function useState(initialValue) {
    // Capture the current hook index
    const currentIndex = hookIndex;
    
    // Initialize state on first render
    if (hooks[currentIndex] === undefined) {
      hooks[currentIndex] = initialValue;
    }
    
    // The setter function
    const setState = (newValue) => {
      // Handle functional updates: setState(prev => prev + 1)
      const value = typeof newValue === 'function' 
        ? newValue(hooks[currentIndex])
        : newValue;
      
      // Only update if value changed
      if (hooks[currentIndex] === value) {
        return; // Bail out optimization
      }
      
      hooks[currentIndex] = value;
      
      // Trigger re-render
      render();
    };
    
    // Move to next hook for next useState call
    hookIndex++;
    
    // Return current state and setter
    return [hooks[currentIndex], setState];
  }
  
  function render() {
    // Reset hook index for new render
    hookIndex = 0;
    
    // Call the component function
    const output = currentComponent();
    
    // In real React, this would update the DOM
    console.log('Rendered:', output);
    
    return output;
  }
  
  return { useState, render };
})();

// Test it!
function Counter() {
  const [count, setCount] = MyReact.useState(0);
  const [name, setName] = MyReact.useState('Alice');
  
  console.log('Counter rendered with:', { count, name });
  
  return { count, name, setCount };
}

// Initial render
MyReact.currentComponent = Counter;
const result1 = MyReact.render();
// Logs: "Counter rendered with: { count: 0, name: 'Alice' }"

// Update state
result1.setCount(1);
// Logs: "Counter rendered with: { count: 1, name: 'Alice' }"

result1.setCount(prev => prev + 1);
// Logs: "Counter rendered with: { count: 2, name: 'Alice' }"
```

**Interview Explanation:**
*"This demonstrates the core principle: React maintains an array of hook values and uses a cursor (hookIndex) to track which hook is being called. The order must be stable because we're relying on array indices. This is why Rules of Hooks exist."*

### **Why Hook Order Matters**

```javascript
// ❌ This breaks our implementation:
function BadComponent() {
  if (someCondition) {
    const [a, setA] = useState(1); // Sometimes index 0
  }
  const [b, setB] = useState(2);   // Sometimes index 0, sometimes index 1!
  
  // React loses track of which state is which
}

// Render 1: someCondition = true
// hooks = [1, 2]
//          ^a ^b

// Render 2: someCondition = false  
// hooks = [1, 2]
//          ^b (but React thinks this is 'a'!) 🐛
```

### **Implementation: Version 2 - Multiple Components**

Now let's handle multiple component instances:

```javascript
const MyReact = (function() {
  // Map of component instances to their hooks
  const componentStates = new WeakMap();
  let currentComponent = null;
  
  function useState(initialValue) {
    // Get or create hooks array for this component
    if (!componentStates.has(currentComponent)) {
      componentStates.set(currentComponent, {
        hooks: [],
        hookIndex: 0
      });
    }
    
    const componentState = componentStates.get(currentComponent);
    const currentIndex = componentState.hookIndex;
    
    // Initialize on first render
    if (componentState.hooks[currentIndex] === undefined) {
      componentState.hooks[currentIndex] = initialValue;
    }
    
    const setState = (newValue) => {
      const value = typeof newValue === 'function'
        ? newValue(componentState.hooks[currentIndex])
        : newValue;
      
      if (Object.is(componentState.hooks[currentIndex], value)) {
        return; // Bail out
      }
      
      componentState.hooks[currentIndex] = value;
      render(currentComponent);
    };
    
    // Increment for next hook
    componentState.hookIndex++;
    
    return [componentState.hooks[currentIndex], setState];
  }
  
  function render(component) {
    currentComponent = component;
    const componentState = componentStates.get(component);
    
    // Reset hook index
    if (componentState) {
      componentState.hookIndex = 0;
    }
    
    // Call component
    const output = component();
    
    console.log('Rendered:', component.name, output);
    return output;
  }
  
  return { useState, render };
})();

// Test with multiple components
function Counter() {
  const [count, setCount] = MyReact.useState(0);
  return { component: 'Counter', count, setCount };
}

function Greeting() {
  const [name, setName] = MyReact.useState('Alice');
  return { component: 'Greeting', name, setName };
}

const counter = MyReact.render(Counter);
const greeting = MyReact.render(Greeting);

counter.setCount(5);   // Only Counter re-renders
greeting.setName('Bob'); // Only Greeting re-renders
```

**Interview Insight:**
*"In real React, component instances are represented by Fiber nodes, and each Fiber has a `memoizedState` property that holds a linked list of hooks. We're using WeakMap for simplicity, but the principle is the same—each component instance maintains its own hook state."*

### **Real React's Hook Structure**

```javascript
// Simplified version of React's actual implementation
function useState(initialState) {
  // Get current fiber (component instance)
  const fiber = getCurrentFiber();
  
  // Hook is a linked list node
  const hook = {
    memoizedState: initialState,  // Current state value
    baseState: initialState,      // State before updates
    queue: {                       // Pending updates
      pending: null,
      dispatch: null
    },
    next: null                     // Next hook in list
  };
  
  // Add to fiber's hook list
  if (fiber.memoizedState === null) {
    fiber.memoizedState = hook;
  } else {
    // Append to linked list
    let lastHook = fiber.memoizedState;
    while (lastHook.next !== null) {
      lastHook = lastHook.next;
    }
    lastHook.next = hook;
  }
  
  const dispatch = (action) => {
    // Add update to queue
    const update = { action, next: null };
    
    // Schedule work on fiber
    scheduleUpdateOnFiber(fiber);
  };
  
  return [hook.memoizedState, dispatch];
}

// Hook linked list example:
// fiber.memoizedState → hook1 → hook2 → hook3 → null
//                       (count)  (name)  (active)
```

---
