## **4. Memory Leaks in React/React Native and Garbage Collection**

This is a critical topic that separates experienced developers from beginners.

### **What is a Memory Leak?**

A memory leak occurs when your application **allocates memory but never releases it**, causing memory usage to grow continuously until the app crashes or becomes unresponsive.

```javascript
// Conceptual example
let leakedMemory = [];

function causingLeak() {
  // This array grows forever
  leakedMemory.push(new Array(1000000).fill('data'));
  // Nothing ever removes items from leakedMemory
  // Memory usage keeps growing...
}

setInterval(causingLeak, 1000); // Leak every second!
```

### **How Garbage Collection Works in JavaScript**

JavaScript engines (V8, Hermes, JSC) use **automatic garbage collection** to reclaim unused memory.

**Mark-and-Sweep Algorithm:**

```javascript
// GC Process:

1. MARK PHASE:
   - Start from "roots" (global variables, call stack)
   - Traverse all reachable objects
   - Mark each object as "reachable"

2. SWEEP PHASE:
   - Find all unmarked (unreachable) objects
   - Free their memory
   - Return memory to available pool

// Example:
let user = { name: 'John', age: 30 }; // Reachable (in scope)
let temp = { data: 'temp' };          // Reachable

temp = null; // Now { data: 'temp' } is unreachable
// GC will eventually collect it
```

**Reachability:**

```javascript
// Object is REACHABLE if:
// 1. It's a root (global, in scope)
// 2. Referenced from a reachable object

let obj1 = { name: 'Object 1' };        // Reachable (root)
let obj2 = { ref: obj1 };               // Reachable (referenced)
obj1.circular = obj2;                   // Circular reference

obj1 = null;
obj2 = null;
// Both obj1 and obj2 are now unreachable
// GC will collect both (handles circular references)
```

### **Generational Garbage Collection (in Hermes/V8):**

```javascript
// Modern GCs use generational hypothesis:
// "Most objects die young"

YOUNG GENERATION (nursery):
- New objects allocated here
- Frequent, fast GC
- Most objects collected quickly

OLD GENERATION (tenured):
- Objects that survive multiple GC cycles promoted here
- Infrequent, slower GC
- Typically longer-lived objects

// Example:
function processData() {
  const temp = { data: 'processing' }; // Young gen
  // temp dies at end of function
  // Quick collection
}

const globalConfig = { setting: 'value' }; // Stays alive
// Eventually promoted to old gen
// Rare collection
```

### **Common Memory Leak Patterns in React/React Native:**

---

### **Leak #1: Uncleared Timers and Intervals**

```javascript
// ❌ MEMORY LEAK
const LeakyComponent = () => {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    setInterval(() => {
      setCount(c => c + 1); // Creates closure over setCount
    }, 1000);
    
    // Component unmounts but interval keeps running!
    // setCount reference prevents GC
    // Component instance can't be collected
  }, []);
  
  return <Text>{count}</Text>;
};

// ✅ FIXED
const FixedComponent = () => {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
    
    return () => clearInterval(interval); // Cleanup!
  }, []);
  
  return <Text>{count}</Text>;
};
```

**What happens without cleanup:**
```javascript
// Component lifecycle:
Mount → setInterval created → Interval runs forever
Unmount → Component tries to be garbage collected
         → But interval still holds reference to setCount
         → setCount references component state
         → Component can't be collected
         → MEMORY LEAK
```

---

### **Leak #2: Event Listeners Not Removed**

```javascript
// ❌ MEMORY LEAK
const LeakyComponent = () => {
  useEffect(() => {
    const handleResize = () => {
      console.log('Window resized');
    };
    
    window.addEventListener('resize', handleResize);
    // Listener persists after unmount!
  }, []);
  
  return <View>...</View>;
};

// ✅ FIXED
const FixedComponent = () => {
  useEffect(() => {
    const handleResize = () => {
      console.log('Window resized');
    };
    
    window.addEventListener('resize', handleResize);
    
    return () => {
      window.removeEventListener('resize', handleResize);
    };
  }, []);
  
  return <View>...</View>;
};
```

**React Native specific:**
```javascript
// ❌ LEAK: Native event listener
import { AppState, DeviceEventEmitter } from 'react-native';

const LeakyComponent = () => {
  useEffect(() => {
    const subscription = AppState.addEventListener('change', nextAppState => {
      console.log('App state:', nextAppState);
    });
    
    // Must remove listener!
  }, []);
};

// ✅ FIXED
const FixedComponent = () => {
  useEffect(() => {
    const subscription = AppState.addEventListener('change', nextAppState => {
      console.log('App state:', nextAppState);
    });
    
    return () => subscription.remove();
  }, []);
};
```

---

### **Leak #3: Async Operations After Unmount**

```javascript
// ❌ MEMORY LEAK
const LeakyComponent = () => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    fetch('https://api.example.com/data')
      .then(res => res.json())
      .then(json => {
        setData(json); // Component might be unmounted!
        // Tries to update unmounted component
        // Keeps component in memory
      });
  }, []);
  
  return <Text>{data?.value}</Text>;
};

// ✅ FIXED - Method 1: Cancellation flag
const FixedComponent = () => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    let cancelled = false;
    
    fetch('https://api.example.com/data')
      .then(res => res.json())
      .then(json => {
        if (!cancelled) {
          setData(json);
        }
      });
    
    return () => {
      cancelled = true; // Prevent state update
    };
  }, []);
  
  return <Text>{data?.value}</Text>;
};

// ✅ FIXED - Method 2: AbortController
const BetterFixedComponent = () => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const abortController = new AbortController();
    
    fetch('https://api.example.com/data', {
      signal: abortController.signal
    })
      .then(res => res.json())
      .then(json => setData(json))
      .catch(error => {
        if (error.name !== 'AbortError') {
          console.error(error);
        }
      });
    
    return () => {
      abortController.abort(); // Cancel request
    };
  }, []);
  
  return <Text>{data?.value}</Text>;
};
```

---

### **Leak #4: Closure Capturing State**

```javascript
// ❌ SUBTLE MEMORY LEAK
const LeakyComponent = () => {
  const [items, setItems] = useState(Array(1000).fill(null).map((_, i) => ({
    id: i,
    data: new Array(10000).fill('large data'), // Large object
  })));
  
  const [selectedId, setSelectedId] = useState(null);
  
  const handleSelect = (id) => {
    setSelectedId(id);
    
    // This function captures 'items' in closure
    setTimeout(() => {
      console.log('Selected from items:', items.find(i => i.id === id));
      // Even if items changes, this closure holds old items array
      // Old items array can't be garbage collected for 5 seconds!
    }, 5000);
  };
  
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => handleSelect(item.id)}>
          <Text>{item.id}</Text>
        </TouchableOpacity>
      )}
    />
  );
};

// ✅ FIXED
const FixedComponent = () => {
  const [items, setItems] = useState(/* ... */);
  const [selectedId, setSelectedId] = useState(null);
  const itemsRef = useRef(items);
  
  useEffect(() => {
    itemsRef.current = items; // Keep ref updated
  }, [items]);
  
  const handleSelect = useCallback((id) => {
    setSelectedId(id);
    
    setTimeout(() => {
      // Use ref, not closure-captured value
      console.log('Selected:', itemsRef.current.find(i => i.id === id));
    }, 5000);
  }, []); // Empty deps - doesn't capture items
  
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => handleSelect(item.id)}>
          <Text>{item.id}</Text>
        </TouchableOpacity>
      )}
    />
  );
};
```

---

### **Leak #5: Global State References**

```javascript
// ❌ MEMORY LEAK
// GlobalStore.js
class GlobalStore {
  listeners = []; // Grows forever!
  
  subscribe(callback) {
    this.listeners.push(callback);
    // No way to unsubscribe!
  }
  
  notify() {
    this.listeners.forEach(cb => cb());
  }
}

export default new GlobalStore();

// Component.js
const LeakyComponent = () => {
  useEffect(() => {
    GlobalStore.subscribe(() => {
      // This callback references component
      // Component can never be GC'd
      console.log('Store updated');
    });
  }, []);
};

// ✅ FIXED
class GlobalStore {
  listeners = [];
  
  subscribe(callback) {
    this.listeners.push(callback);
    
    // Return unsubscribe function
    return () => {
      this.listeners = this.listeners.filter(cb => cb !== callback);
    };
  }
  
  notify() {
    this.listeners.forEach(cb => cb());
  }
}

const FixedComponent = () => {
  useEffect(() => {
    const unsubscribe = GlobalStore.subscribe(() => {
      console.log('Store updated');
    });
    
    return unsubscribe; // Cleanup
  }, []);
};
```

---

### **Leak #6: Retained DOM/Native References**

```javascript
// ❌ MEMORY LEAK
const LeakyComponent = () => {
  const viewRefs = useRef([]);
  
  const storeRef = (ref, index) => {
    viewRefs.current[index] = ref;
    // Array grows but never shrinks
    // Old refs prevent GC even after unmount
  };
  
  return (
    <FlatList
      data={items}
      renderItem={({ item, index }) => (
        <View ref={ref => storeRef(ref, index)}>
          <Text>{item.text}</Text>
        </View>
      )}
    />
  );
};

// ✅ FIXED
const FixedComponent = () => {
  const viewRefs = useRef(new Map());
  
  const storeRef = useCallback((ref, id) => {
    if (ref) {
      viewRefs.current.set(id, ref);
    } else {
      viewRefs.current.delete(id); // Clean up!
    }
  }, []);
  
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <View ref={ref => storeRef(ref, item.id)}>
          <Text>{item.text}</Text>
        </View>
      )}
    />
  );
};
```

---

### **Leak #7: React Native Specific - Animated Values**

```javascript
// ❌ POTENTIAL LEAK
const LeakyComponent = () => {
  const animatedValue = new Animated.Value(0);
  
  useEffect(() => {
    Animated.loop(
      Animated.timing(animatedValue, {
        toValue: 1,
        duration: 1000,
        useNativeDriver: true,
      })
    ).start();
    
    // Animation runs forever, even after unmount!
  }, []);
  
  return <Animated.View style={{ opacity: animatedValue }} />;
};

// ✅ FIXED
const FixedComponent = () => {
  const animatedValue = useRef(new Animated.Value(0)).current;
  
  useEffect(() => {
    const animation = Animated.loop(
      Animated.timing(animatedValue, {
        toValue: 1,
        duration: 1000,
        useNativeDriver: true,
      })
    );
    
    animation.start();
    
    return () => {
      animation.stop(); // Stop animation on unmount
    };
  }, [animatedValue]);
  
  return <Animated.View style={{ opacity: animatedValue }} />;
};
```

---

### **Detecting Memory Leaks:**

**1. Chrome DevTools Memory Profiler:**
```javascript
// In development:
// 1. Open Chrome DevTools
// 2. Go to Memory tab
// 3. Take heap snapshot
// 4. Navigate around app
// 5. Force GC (trash icon)
// 6. Take another snapshot
// 7. Compare - look for detached nodes or growing objects
```

**2. React DevTools Profiler:**
```javascript
import { Profiler } from 'react';

<Profiler id="MyComponent" onRender={(id, phase, actualDuration) => {
  // Monitor component lifecycle
  console.log(`${id} ${phase} took ${actualDuration}ms`);
}}>
  <MyComponent />
</Profiler>
```

**3. Flipper Memory Inspector:**
```javascript
// Flipper shows:
// - Native memory usage
// - JavaScript heap size
// - Component tree
// - Can track memory over time
```

**4. Why Did You Render (Library):**
```javascript
import whyDidYouRender from '@welldone-software/why-did-you-render';

whyDidYouRender(React, {
  trackAllPureComponents: true,
  logOnDifferentValues: true,
});

// Logs unnecessary re-renders that might indicate leaks
```

---

### **Best Practices to Prevent Leaks:**

```javascript
// ✅ 1. Always clean up effects
useEffect(() => {
  // Setup
  const subscription = subscribe();
  const timer = setInterval(() => {}, 1000);
  
  return () => {
    // Cleanup
    subscription.unsubscribe();
    clearInterval(timer);
  };
}, []);

// ✅ 2. Use useCallback for event handlers
const handlePress = useCallback(() => {
  // Don't capture unnecessary dependencies
}, []); // Or minimal dependencies

// ✅ 3. Prevent updates after unmount
useEffect(() => {
  let mounted = true;
  
  fetchData().then(data => {
    if (mounted) {
      setData(data);
    }
  });
  
  return () => {
    mounted = false;
  };
}, []);

// ✅ 4. Use WeakMap for caching
const cache = new WeakMap(); // Allows GC of keys
// Instead of: const cache = new Map();

// ✅ 5. Clear refs when done
useEffect(() => {
  return () => {
    myRef.current = null; // Allow GC
  };
}, []);
```

**Interview talking point:**

*"Memory leaks in React and React Native occur when components maintain references that prevent garbage collection, most commonly through uncleaned timers, event listeners, or async operations that continue after unmount. JavaScript's garbage collector uses mark-and-sweep to reclaim memory, but it can only collect objects that are unreachable. The key to preventing leaks is ensuring all effect cleanup functions properly remove subscriptions, clear timers, cancel requests, and stop animations. In React Native specifically, we must also be careful with native event listeners, Animated values, and bridge-crossing references that can keep components alive. Tools like Chrome DevTools heap snapshots and Flipper's memory profiler help identify leaks by showing detached DOM nodes or unexpected memory growth over time."*

---
