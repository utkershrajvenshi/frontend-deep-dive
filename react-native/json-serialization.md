## **3. JSON Serialization: Why It's Necessary**

This is a **critical architectural constraint** to understand.

### **The Core Problem: Process Isolation**

```
┌─────────────────┐          ┌──────────────────┐
│   JS Runtime    │          │  Native Runtime  │
│  (JSC/Hermes)   │          │   (iOS/Android)  │
│                 │          │                  │
│  Memory Space A │   ≠≠≠    │  Memory Space B  │
└─────────────────┘          └──────────────────┘
```

**Why they can't share memory directly:**
- Different runtimes: JavaScript engine vs Native platform code
- Different type systems: JS objects vs Objective-C/Java objects
- Security and stability: Isolation prevents crashes in one realm from affecting the other

### **Serialization Flow (Both Directions):**

```javascript
// NATIVE → JS (e.g., touch event)
// Native side:
NSDictionary *touchData = @{
  @"pageX": @150.5,
  @"pageY": @200.3,
  @"timestamp": @1234567890
};
NSString *jsonString = [TouchData toJSON]; // Serialize
[bridge sendToJS:jsonString]; // Send string across

// JS side receives:
const event = JSON.parse(jsonString); // Deserialize
console.log(event.pageX); // 150.5
```

```javascript
// JS → NATIVE (e.g., navigation command)
// JS side:
const navigationParams = {
  screen: 'ProfileScreen',
  userId: 12345,
  animated: true
};
const jsonString = JSON.stringify(navigationParams); // Serialize
bridge.sendToNative('navigate', jsonString); // Send

// Native side:
NSDictionary *params = [JSONParser parse:jsonString]; // Deserialize
[navigator pushViewController:params[@"screen"]]; // Use the data
```

### **Yes, Native Also Serializes!**

This is important to clarify - **serialization happens in both directions:**

- **Native → JS**: Touch events, scroll updates, native module responses
- **JS → Native**: Component updates, native method calls, event handlers

### **Performance Implications:**

```javascript
// ❌ Large object serialization is expensive
const BadScrollHandler = () => {
  const [items] = useState(Array(10000).fill({
    id: '...',
    data: '...',
    nested: { /* lots of data */ }
  }));
  
  const handleScroll = (event) => {
    // If this triggers re-render with new items array
    // All 10,000 objects serialize across bridge!
    setScrollPosition(event.nativeEvent.contentOffset.y);
  };
  
  return <ScrollView onScroll={handleScroll}>{/* ... */}</ScrollView>;
};

// ✅ Minimize bridge traffic
const GoodScrollHandler = () => {
  const handleScroll = (event) => {
    // Only serialize the specific data needed
    const yOffset = event.nativeEvent.contentOffset.y;
    setScrollPosition(yOffset); // Just a number
  };
  
  return (
    <ScrollView 
      onScroll={handleScroll}
      scrollEventThrottle={16} // Limit frequency
    >
      {/* ... */}
    </ScrollView>
  );
};
```

**Interview insight:**

*"JSON serialization is necessary because JavaScript and native code run in completely separate memory spaces with different runtime environments. Every piece of data crossing the bridge must be converted to a platform-neutral format - JSON strings - which both sides can parse. This creates overhead, especially for large payloads or high-frequency events, which is why techniques like `useNativeDriver` for animations are important: they keep data on one side of the bridge to avoid serialization costs."*

---
