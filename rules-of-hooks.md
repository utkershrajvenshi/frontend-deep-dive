# The Complete Rules of Hooks: A Deep Dive

Excellent question! The Rules of Hooks are fundamental to React's hook system. Let me coach you through all of them with the "why" behind each rule.

---

## **The Two Official Rules of Hooks**

### **Rule 1: Only Call Hooks at the Top Level**

**Don't call Hooks inside loops, conditions, or nested functions.**

```javascript
// ❌ BAD: Conditional hook
function Component({ shouldTrack }) {
  if (shouldTrack) {
    const [count, setCount] = useState(0); // Breaks hook order!
  }
}

// ❌ BAD: Hook in loop
function Component({ items }) {
  items.forEach(item => {
    const [value, setValue] = useState(item); // Different number of hooks each render
  });
}

// ❌ BAD: Hook in nested function
function Component() {
  const handleClick = () => {
    const [value, setValue] = useState(0); // Hooks can't be called in callbacks
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// ❌ BAD: Hook after early return
function Component({ condition }) {
  if (condition) {
    return <div>Early return</div>;
  }
  
  const [count, setCount] = useState(0); // Might not be called!
}
```

**Why this rule exists:**

```javascript
// Render 1: condition = true
function Component({ condition }) {
  // Hook index 0
  const [name, setName] = useState('Alice');
  
  if (condition) {
    // Hook index 1
    const [age, setAge] = useState(25);
  }
  
  // Hook index 2 (or 1 if condition is false!)
  const [city, setCity] = useState('NYC');
}

// React's internal state:
// Render 1 (condition = true):  ['Alice', 25, 'NYC']
// Render 2 (condition = false): ['Alice', 'NYC', ???]
//                                        ↑ React thinks this is 'age'!
```

**✅ CORRECT: Move condition inside hooks**

```javascript
function Component({ shouldTrack }) {
  // Always call the hook
  const [count, setCount] = useState(0);
  
  // Conditionally use the value
  const displayCount = shouldTrack ? count : null;
  
  return <div>{displayCount}</div>;
}

// ✅ CORRECT: Multiple components instead of conditions
function Component({ userType }) {
  if (userType === 'admin') {
    return <AdminComponent />;
  }
  return <UserComponent />;
}

function AdminComponent() {
  const [adminData, setAdminData] = useState(null); // Different component = different hooks
}

function UserComponent() {
  const [userData, setUserData] = useState(null);
}
```

---

### **Rule 2: Only Call Hooks from React Functions**

**Only call Hooks from React function components or custom Hooks.**

```javascript
// ❌ BAD: Hook in regular JavaScript function
function calculateTotal(items) {
  const [total, setTotal] = useState(0); // Not a React component!
  // ...
}

// ❌ BAD: Hook in class component
class MyComponent extends React.Component {
  componentDidMount() {
    const [data, setData] = useState(null); // Hooks don't work in classes!
  }
}

// ❌ BAD: Hook in event handler
function Component() {
  const handleClick = () => {
    const [clicked, setClicked] = useState(false); // Not top-level!
  };
  
  return <button onClick={handleClick}>Click</button>;
}

// ❌ BAD: Hook in regular function (not a component)
function utils() {
  const [value, setValue] = useState(0); // Not called during render!
}
```

**✅ CORRECT: Hooks in function components**

```javascript
function MyComponent() {
  const [count, setCount] = useState(0); // ✅ Function component
  
  return <div>{count}</div>;
}
```

**✅ CORRECT: Hooks in custom hooks**

```javascript
function useCustomHook() {
  const [state, setState] = useState(0); // ✅ Custom hook (starts with 'use')
  
  useEffect(() => {
    // ...
  }, [state]);
  
  return [state, setState];
}
```

**Why this rule exists:**

React needs to track which component instance is currently rendering to associate hooks with the correct component state:

```javascript
// React's internal logic (simplified)
let currentlyRenderingFiber = null;

function renderComponent(fiber) {
  currentlyRenderingFiber = fiber; // Set current context
  
  const Component = fiber.type;
  const result = Component(fiber.props); // Call component
  
  currentlyRenderingFiber = null; // Clear context
  return result;
}

function useState(initial) {
  if (!currentlyRenderingFiber) {
    throw new Error('Hooks can only be called inside function components!');
  }
  
  // Get hooks from current fiber
  const hooks = currentlyRenderingFiber.memoizedState;
  // ...
}

// If you call useState outside a component:
useState(0); // currentlyRenderingFiber is null → Error!

// If you call useState in an event handler:
function Component() {
  const handleClick = () => {
    useState(0); // currentlyRenderingFiber is null → Error!
  };
  // handleClick is called later, not during render
}
```

---

## **Additional Unofficial But Important Rules**

These aren't in the official docs as "rules" but are critical best practices:

### **Rule 3: Custom Hooks Must Start with "use"**

```javascript
// ❌ BAD: Doesn't start with 'use'
function fetchData() {
  const [data, setData] = useState(null);
  // ESLint can't verify hook rules are followed
}

// ✅ GOOD: Starts with 'use'
function useFetchData() {
  const [data, setData] = useState(null);
  // ESLint can verify this follows hook rules
  
  useEffect(() => {
    // Fetch data
  }, []);
  
  return data;
}
```

**Why this matters:**

```javascript
// ESLint's 'rules-of-hooks' plugin uses the 'use' prefix to:
// 1. Identify functions that can call hooks
// 2. Verify hook rules are followed
// 3. Check dependency arrays

function useFetchData(url) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData);
  }, [url]); // ESLint verifies 'url' is in deps
  
  return data;
}

// Without 'use' prefix:
function fetchData(url) {
  const [data, setData] = useState(null); // ESLint error!
  // "React Hook 'useState' is called in function 'fetchData' that is 
  // neither a React function component nor a custom React Hook function"
}
```

### **Rule 4: Include All Dependencies in Effect/Callback Arrays**

```javascript
// ❌ BAD: Missing dependencies
function Component({ userId }) {
  const [user, setUser] = useState(null);
  
  useEffect(() => {
    fetchUser(userId).then(setUser);
    // ESLint warning: React Hook useEffect has a missing dependency: 'userId'
  }, []); // userId is not in the array
}

// ❌ BAD: Disabling the lint rule without good reason
function Component({ userId, onSuccess }) {
  useEffect(() => {
    fetchUser(userId).then(onSuccess);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []); // Dangerous! userId and onSuccess can become stale
}
```

**Why this matters - Stale Closures:**

```javascript
function MessageDisplay({ messageId, onRead }) {
  useEffect(() => {
    fetchMessage(messageId).then(msg => {
      displayMessage(msg);
      onRead(messageId); // Uses stale onRead from first render!
    });
  }, [messageId]); // Missing: onRead
}

// Scenario:
// 1. Parent renders MessageDisplay with onRead1
// 2. Effect captures onRead1
// 3. Parent re-renders with onRead2 (but same messageId)
// 4. Effect doesn't re-run (deps haven't changed)
// 5. When message loads, calls old onRead1 instead of onRead2
```

**✅ CORRECT: Include all dependencies**

```javascript
function Component({ userId, onSuccess }) {
  useEffect(() => {
    fetchUser(userId).then(onSuccess);
  }, [userId, onSuccess]); // All dependencies included
}

// But now effect re-runs if onSuccess changes reference!
// Parent component should stabilize it:
function ParentComponent() {
  // ✅ useCallback prevents new function each render
  const handleSuccess = useCallback((user) => {
    console.log('Success:', user);
  }, []); // Stable reference
  
  return <Component userId={123} onSuccess={handleSuccess} />;
}
```

**When you CAN omit dependencies:**

```javascript
// ✅ setState is stable (guaranteed by React)
function Component() {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const timer = setInterval(() => {
      setCount(c => c + 1); // setState is stable, no need to add to deps
    }, 1000);
    
    return () => clearInterval(timer);
  }, []); // Empty array is safe here
}

// ✅ Refs are stable (guaranteed by React)
function Component() {
  const ref = useRef(null);
  
  useEffect(() => {
    console.log(ref.current); // ref is stable
  }, []); // Safe to omit ref
}

// ✅ Values from outside component scope (constants)
const API_URL = 'https://api.example.com';

function Component() {
  useEffect(() => {
    fetch(API_URL); // Constant from outside
  }, []); // Safe to omit API_URL
}
```

---

## **Edge Cases and Gotchas**

### **Edge Case 1: Dynamic Number of Hooks**

```javascript
// ❌ BAD: Number of hooks changes
function Form({ fields }) {
  // fields = ['name', 'email'] on first render
  // fields = ['name', 'email', 'phone'] on second render
  
  fields.forEach(field => {
    const [value, setValue] = useState(''); // Different count each render!
  });
}

// ✅ GOOD: Use single state object
function Form({ fields }) {
  const [values, setValues] = useState(() => 
    fields.reduce((acc, field) => ({ ...acc, [field]: '' }), {})
  );
  
  const updateField = (field, value) => {
    setValues(prev => ({ ...prev, [field]: value }));
  };
}

// ✅ GOOD: Use dynamic component
function Form({ fields }) {
  return fields.map(field => (
    <Field key={field} name={field} /> // Each Field has its own hooks
  ));
}

function Field({ name }) {
  const [value, setValue] = useState(''); // Different component instance
  return <input value={value} onChange={e => setValue(e.target.value)} />;
}
```

### **Edge Case 2: Hooks in Callbacks That Look Like Components**

```javascript
// ❌ This looks weird but might be called during render
function Component() {
  const renderHelper = () => {
    const [value, setValue] = useState(0); // Is this allowed?
    return <div>{value}</div>;
  };
  
  return renderHelper(); // Called during render, so... maybe?
}

// React's perspective:
// - renderHelper is called during Component's render
// - Hooks ARE called at top level of Component's execution
// - But ESLint will complain because renderHelper doesn't start with 'use'

// ✅ BETTER: Just inline it
function Component() {
  const [value, setValue] = useState(0);
  return <div>{value}</div>;
}

// ✅ OR: Make it a proper component
function Helper() {
  const [value, setValue] = useState(0);
  return <div>{value}</div>;
}

function Component() {
  return <Helper />;
}
```

### **Edge Case 3: Hooks in Higher-Order Components**

```javascript
// ❌ BAD: Hook in HOC (not a component or custom hook)
function withData(Component) {
  return function(props) {
    const [data, setData] = useState(null); // ✅ Actually this is fine!
    // This IS a component (the wrapper)
    
    return <Component data={data} {...props} />;
  };
}

// But this is confusing, better to use proper naming:

// ✅ GOOD: Clear component name
function withData(Component) {
  function WithData(props) { // Named component
    const [data, setData] = useState(null);
    return <Component data={data} {...props} />;
  }
  
  return WithData;
}

// ✅ BETTER: Use a custom hook instead
function useData() {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    // Fetch data
  }, []);
  
  return data;
}

function Component() {
  const data = useData(); // Compose with hooks
  return <div>{data}</div>;
}
```

### **Edge Case 4: Hooks and Error Boundaries**

```javascript
// ❌ You can't use hooks to catch errors
function Component() {
  const [hasError, setHasError] = useState(false);
  
  try {
    // This won't catch render errors!
    return <ChildThatMightError />;
  } catch (error) {
    // Never executed for render errors
    setHasError(true);
  }
}

// ✅ Must use class component for error boundaries
class ErrorBoundary extends React.Component {
  state = { hasError: false };
  
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  
  componentDidCatch(error, errorInfo) {
    logError(error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return <ErrorFallback />;
    }
    return this.props.children;
  }
}

// Usage:
function App() {
  return (
    <ErrorBoundary>
      <ComponentThatMightError />
    </ErrorBoundary>
  );
}
```

---

## **ESLint Configuration for Hook Rules**

**Essential ESLint setup:**

```json
{
  "extends": [
    "react-app"
  ],
  "plugins": [
    "react-hooks"
  ],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

**What each rule catches:**

```javascript
// react-hooks/rules-of-hooks catches:
function Component() {
  if (condition) {
    useState(0); // ❌ Error: Conditional hook
  }
  
  for (let i = 0; i < 10; i++) {
    useState(i); // ❌ Error: Hook in loop
  }
}

// react-hooks/exhaustive-deps catches:
function Component({ userId }) {
  useEffect(() => {
    fetchUser(userId);
  }, []); // ⚠️ Warning: Missing dependency 'userId'
  
  const memoized = useMemo(() => {
    return expensiveCalc(userId);
  }, []); // ⚠️ Warning: Missing dependency 'userId'
  
  const callback = useCallback(() => {
    doSomething(userId);
  }, []); // ⚠️ Warning: Missing dependency 'userId'
}
```

---

## **Common Interview Questions About Hook Rules**

### **Q: "Why does React use hook call order instead of hook names/IDs?"**

```javascript
// Hypothetical: What if React used names instead of order?
function Component() {
  const [count, setCount] = useState('myCount', 0);
  const [name, setName] = useState('myName', 'Alice');
}

// Problems with this approach:
// 1. Name collisions in custom hooks
function useCounter() {
  const [count, setCount] = useState('count', 0); // Same name as parent!
}

// 2. Refactoring becomes dangerous
// Can't rename without breaking state

// 3. More expensive - need string comparison instead of index lookup

// Order-based approach:
// ✅ No naming needed
// ✅ Fast (array index lookup)
// ✅ Works seamlessly with composition
// ❌ Requires call order stability (hence the rules)
```

### **Q: "Can I dynamically create hooks?"**

```javascript
// ❌ No, this violates the rules
function Component({ fieldNames }) {
  const states = fieldNames.map(name => useState('')); // Dynamic hooks!
  // This creates different number of hooks on each render
}

// ✅ Instead, use a single hook with an object
function Component({ fieldNames }) {
  const [fields, setFields] = useState(() =>
    fieldNames.reduce((acc, name) => ({ ...acc, [name]: '' }), {})
  );
  
  const updateField = (name, value) => {
    setFields(prev => ({ ...prev, [name]: value }));
  };
  
  return (
    <>
      {fieldNames.map(name => (
        <input
          key={name}
          value={fields[name]}
          onChange={e => updateField(name, e.target.value)}
        />
      ))}
    </>
  );
}
```

### **Q: "What if I need conditional state?"**

```javascript
// ❌ BAD: Conditional hook
function Component({ userType }) {
  if (userType === 'admin') {
    const [adminData, setAdminData] = useState(null);
  }
}

// ✅ SOLUTION 1: Always call hook, conditionally use value
function Component({ userType }) {
  const [adminData, setAdminData] = useState(null);
  
  const dataToDisplay = userType === 'admin' ? adminData : null;
}

// ✅ SOLUTION 2: Separate components
function Component({ userType }) {
  if (userType === 'admin') {
    return <AdminComponent />;
  }
  return <UserComponent />;
}

// ✅ SOLUTION 3: Custom hook with conditional logic
function useConditionalData(condition) {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    if (condition) {
      fetchData().then(setData);
    } else {
      setData(null);
    }
  }, [condition]);
  
  return data;
}

function Component({ userType }) {
  const adminData = useConditionalData(userType === 'admin');
}
```

---

## **Interview Pro Tips: Explaining Hook Rules**

### **The Elevator Pitch**

*"React hooks use call order to maintain state consistency. Each hook call maps to an index in an array stored on the component's Fiber node. This requires hooks to be called in the same order every render, which is why we can't use them conditionally or in loops. The trade-off is a simple, composable API without needing to manage hook IDs or names."*

### **Visual Explanation**

```javascript
function Component({ showExtra }) {
  // Render 1:
  const [name, setName] = useState('Alice');    // Index 0
  if (showExtra) {
    const [age, setAge] = useState(25);         // Index 1
  }
  const [city, setCity] = useState('NYC');      // Index 1 or 2?
  
  // React's internal array:
  // showExtra=true:  [State('Alice'), State(25), State('NYC')]
  // showExtra=false: [State('Alice'), State('NYC')]
  //                                    ↑ React thinks this is age!
}

// The fix - stable order:
function Component({ showExtra }) {
  const [name, setName] = useState('Alice');    // Always index 0
  const [age, setAge] = useState(25);           // Always index 1
  const [city, setCity] = useState('NYC');      // Always index 2
  
  // Just don't display age when !showExtra
  return (
    <div>
      <p>{name}</p>
      {showExtra && <p>{age}</p>}
      <p>{city}</p>
    </div>
  );
}
```

### **When They Ask: "Have you ever broken hook rules?"**

**Good answer:**
*"Early on, I tried to conditionally call useEffect based on a prop, which caused inconsistent behavior. ESLint caught it immediately with the 'rules-of-hooks' error. I learned to structure the logic so the hook is always called, but the effect's behavior is conditional based on dependencies or internal checks. This taught me to think in terms of 'always declare, conditionally execute' rather than 'conditionally declare.'"*

---

## **Summary Checklist for Interviews**

✅ **The Two Official Rules:**
1. Only call hooks at the top level (no conditions, loops, or nested functions)
2. Only call hooks from React function components or custom hooks

✅ **Critical Best Practices:**
3. Custom hooks must start with "use" (for ESLint)
4. Include all dependencies in effect/callback arrays

✅ **Key Insights to Communicate:**
- Hook order = array index mapping
- Stable order required for state consistency
- ESLint enforces these rules automatically
- Trade-off: Simplicity vs. flexibility

✅ **Common Gotchas:**
- Stale closures from missing dependencies
- New object/array in deps causes re-runs
- Can't catch render errors with hooks (need Error Boundary class)
- Dynamic hook count violates rules

This comprehensive understanding will show you don't just know the rules—you understand **why** they exist and how to work within them effectively!
