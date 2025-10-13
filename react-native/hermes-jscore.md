## **8. Hermes vs JavaScriptCore: The JS Runtime Evolution**

Understanding why React Native needs different JavaScript engines shows architectural depth.

### **What Are They?**

**JavaScript Engines** execute your JavaScript code. Different engines optimize for different things.

### **JavaScriptCore (JSC):**

- **Created by**: Apple (for Safari)
- **Optimized for**: Desktop/laptop performance
- **Used in**: Safari, WebKit-based apps, older React Native

```javascript
// JSC Architecture
JavaScript Code → Parser → Bytecode → JIT Compiler → Native Code
                                    ↓
                            (Optimizing compiler)
```

**JSC Characteristics:**
- Sophisticated JIT (Just-In-Time) compilation
- Large binary size (~1-2 MB)
- High memory usage
- Great for long-running applications
- Optimizes hot code paths over time

### **Hermes:**

- **Created by**: Facebook (specifically for React Native)
- **Optimized for**: Mobile constraints (battery, memory, storage)
- **Used in**: Modern React Native (default since RN 0.70)

```javascript
// Hermes Architecture
JavaScript Code → Bytecode (at build time) → Hermes VM → Execution
                                           ↓
                                    (Minimal JIT)
```

### **Why React Native Needed Hermes:**

**Problem 1: Slow Startup with JSC**
```javascript
// With JSC:
App launches → Load JS bundle (1-2 MB) 
→ Parse JavaScript (slow!) 
→ Compile to bytecode 
→ Start executing
// Could take 1-2 seconds on low-end Android
```

**Solution with Hermes:**
```javascript
// With Hermes:
Build time: JavaScript → Hermes bytecode (.hbc file)

App launches → Load pre-compiled bytecode
→ Start executing immediately!
// Startup 2-3x faster
```

### **Key Differences:**

| Feature | JavaScriptCore | Hermes |
|---------|----------------|---------|
| **Compilation** | Runtime (JIT) | Build time (AOT) |
| **Startup Time** | Slower (parse + compile) | Faster (pre-compiled) |
| **Memory Usage** | Higher (~10-15 MB) | Lower (~5-7 MB) |
| **Binary Size** | Larger | Smaller |
| **Optimization** | Runtime optimizations | Ahead-of-time |
| **Debugging** | Chrome DevTools | Chrome DevTools + Flipper |

### **Concrete Performance Example:**

```javascript
// Complex app initialization
const App = () => {
  useEffect(() => {
    // Lots of initial JavaScript execution
    initializeAnalytics();
    loadUserData();
    setupPushNotifications();
    configureApp();
  }, []);
  
  return <NavigationContainer>...</NavigationContainer>;
};

// JSC: Parse and compile all this code at startup = slow
// Hermes: Pre-compiled bytecode, executes immediately = fast
```

### **Hermes Optimizations for Mobile:**

**1. Bytecode Format:**
```
JavaScript (Text) → Hermes Bytecode (Binary)

Bundle size:
- JS: 2.5 MB
- Hermes .hbc: 1.8 MB (compressed better)
```

**2. Garbage Collection:**
```javascript
// Hermes uses incremental, concurrent GC
// Spreads GC work across frames
// Reduces pause times that cause jank

// JSC uses generational GC
// Can have longer pause times on mobile
```

**3. Memory Efficiency:**
```javascript
// Hermes allocates memory more efficiently
const largeArray = new Array(100000);

// JSC: More memory overhead per object
// Hermes: Optimized object layout for mobile
```

### **Why Keep JSC?**

**iOS Consideration:**
- JSC is **built into iOS** (part of WebKit)
- Using JSC on iOS = no extra binary size
- Hermes adds ~1-2 MB to iOS app

**Current Best Practice:**
```javascript
// react-native.config.js
module.exports = {
  project: {
    ios: {},
    android: {},
  },
  // Hermes is default now
  // But can be disabled if needed
};
```

**When to use JSC:**
- If app doesn't have startup time issues
- If you need features not in Hermes yet
- If minimizing iOS binary size is critical

**When to use Hermes (most cases):**
- Faster startup needed
- Low-end Android devices targeted
- Lower memory footprint desired
- Modern React Native (it's the default)

### **Hermes and the New Architecture:**

Hermes was **designed for JSI compatibility**:

```cpp
// Hermes exposes JSI interface natively
class HermesRuntimeImpl : public jsi::Runtime {
  // Direct C++ API for JSI
  jsi::Value evaluateJavaScript(...);
  jsi::Object global();
  // etc.
};

// Perfect integration with TurboModules and Fabric
```

### **Debugging Differences:**

**JSC:**
```javascript
// Remote debugging via Chrome DevTools
// Runs in Chrome's V8 (not actual device engine!)
// Can cause timing differences
```

**Hermes:**
```javascript
// Direct debugging of Hermes engine
// Flipper provides accurate device debugging
// Chrome DevTools still works
// Can inspect actual bytecode with hermes-profile
```

**Interview insight:**

*"React Native supports multiple JavaScript engines because different engines optimize for different scenarios. JavaScriptCore, built for desktop browsers, emphasizes runtime optimizations and is built into iOS. Hermes was created specifically for React Native to address mobile constraints - it uses ahead-of-time compilation to bytecode, dramatically improving startup time and reducing memory usage. On Android, Hermes is the clear choice for better performance. On iOS, it's a trade-off between startup speed and binary size, though Hermes is now the recommended default for both platforms."*

---
