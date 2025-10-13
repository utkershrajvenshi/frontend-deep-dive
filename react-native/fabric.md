## **5. Fabric: The New Rendering System**

Fabric is the **new rendering architecture** that replaces the legacy bridge-based renderer.

### **What Fabric Replaces:**

**Old Rendering Pipeline:**
```
JS: React reconciliation → Update instructions
→ Serialize to JSON → Bridge → Native UI Manager
→ Deserialize → Update native views (async, batched)
```

**Fabric Pipeline:**
```
JS: React reconciliation → C++ Shadow Tree (via JSI)
→ Direct synchronous layout → Immediate native updates
```

### **Key Improvements:**

**1. Synchronous Layout and Rendering:**

```javascript
// OLD: This had timing issues
const MyComponent = () => {
  const ref = useRef();
  
  const handlePress = () => {
    // This was async, measurement might be stale
    ref.current.measure((x, y, width, height) => {
      console.log(height); // Might not reflect just-rendered state
    });
  };
  
  return <View ref={ref} onPress={handlePress}>...</View>;
};

// FABRIC: Synchronous, always accurate
const MyComponent = () => {
  const ref = useRef();
  
  const handlePress = () => {
    const layout = ref.current.measureSync(); // Immediate, accurate
    console.log(layout.height);
  };
  
  return <View ref={ref} onPress={handlePress}>...</View>;
};
```

**2. Improved Interop with Native:**

```jsx
// Fabric enables better integration
<NativeNavigationView>
  <ReactNativeScreen>
    {/* React Native content */}
  </ReactNativeScreen>
</NativeNavigationView>
// Smoother integration, native navigation can host RN screens
```

**3. Priority-based Rendering:**

Fabric implements React's **Concurrent Rendering**, allowing:
- High-priority updates (user interactions) render immediately
- Low-priority updates (off-screen content) can be deferred
- Smoother UX during heavy operations

```javascript
// With Fabric + Concurrent React
import { startTransition } from 'react';

const handleSearch = (text) => {
  setInputValue(text); // High priority - immediate
  
  startTransition(() => {
    setSearchResults(expensiveSearch(text)); // Low priority - deferrable
  });
};
```

### **Architecture Changes:**

**Shadow Tree in C++:**

```
OLD: JS Shadow Tree → Bridge → Native UI
NEW: C++ Shadow Tree → Direct Native UI updates

Benefits:
- Faster layout calculations
- Shared between JS and Native
- Type-safe interface
```

**Interview insight:**

*"Fabric is React Native's new rendering system that leverages JSI to eliminate the async bridge for UI updates. By moving the Shadow Tree to C++ and enabling direct synchronous communication between JavaScript and native rendering, Fabric provides more accurate layout measurements, better integration with native UI components, and support for React's Concurrent features. It's a fundamental piece of the New Architecture that makes React Native more performant and capable."*

---
