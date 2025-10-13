## **1. React Native vs React Web: Architectural Differences**

This is an excellent comparison that interviewers love to hear articulated clearly.

### **React Web (Browser Environment)**

```
JavaScript Code → React reconciliation → VDOM diffing 
→ DOM mutations → Browser rendering engine paints pixels
```

**Key characteristics:**
- **Single environment**: Everything runs in the browser's JavaScript engine
- **Direct access**: JavaScript can directly manipulate the DOM via APIs like `document.createElement()`
- **Synchronous operations**: `element.style.color = 'red'` happens immediately
- **No serialization**: The JS engine and DOM are in the same memory space

### **React Native (Mobile Environment)**

```
JavaScript Code → React reconciliation → Shadow Tree diffing
→ Bridge serialization → Native UI updates → Platform rendering
```

**Key differences:**

```javascript
// REACT WEB - Direct manipulation
const button = document.createElement('button');
button.textContent = 'Click me';
button.onclick = handleClick;
document.body.appendChild(button);
// All synchronous, same memory space

// REACT NATIVE - Bridge-mediated
<TouchableOpacity onPress={handlePress}>
  <Text>Click me</Text>
</TouchableOpacity>
// JSX → Bridge → Native UIButton (iOS) / android.widget.Button
// Asynchronous, separate memory spaces
```

### **The Critical Differences:**

| Aspect | React Web | React Native |
|--------|-----------|--------------|
| **Rendering Target** | Browser DOM (HTML elements) | Native platform components (UIView, TextView) |
| **Communication** | Direct function calls | Asynchronous message passing through bridge |
| **Memory Space** | Shared between JS and DOM | Separate JS and Native realms |
| **Threading** | Single main thread (with Web Workers optional) | Multiple threads (JS, Native, Shadow) |
| **Layout Engine** | Browser's layout engine (Blink, WebKit) | Yoga (Flexbox implementation) |

**Interview insight to share:**

*"While both use React's reconciliation algorithm and VDOM concept, React Native faces an additional architectural challenge: JavaScript cannot directly create or manipulate native UI components. The VDOM in React Native represents a tree of native components, and changes must be communicated across a process boundary, which is why the bridge exists. In contrast, React Web operates entirely within the browser's JavaScript context and can directly manipulate the DOM."*

---
