## **7. React Native vs Flutter Architecture**

This comparison is increasingly common in interviews as companies choose between frameworks.

### **Fundamental Architectural Difference:**

```
REACT NATIVE:
JavaScript → Bridge/JSI → Native Components (UIKit/Android)
Real native widgets controlled from JavaScript

FLUTTER:
Dart → Skia Rendering Engine → Canvas (pixels)
Custom widget system, not native components
```

### **React Native: Native Widget Approach**

```jsx
<View>         → UIView (iOS) / android.view.View
<Text>         → UILabel / TextView  
<TextInput>    → UITextField / EditText
<ScrollView>   → UIScrollView / ScrollView
```

**Pros:**
- Truly native look and feel (platform-specific by default)
- Automatic platform updates (new iOS switch style = automatic)
- Native accessibility built-in
- Smaller app size (reuses platform components)

**Cons:**
- Bridge/JSI overhead for communication
- Consistency across platforms requires work
- Dependent on native component capabilities

### **Flutter: Custom Rendering Approach**

```dart
Everything → Skia → Draws pixels directly
```

Flutter uses **Skia** (a 2D graphics library) to render everything from scratch.

```dart
// Flutter doesn't use native button
// It draws what looks like a button
ElevatedButton(
  child: Text('Click me'),
  onPressed: () {},
)
// Skia draws rounded rectangle, shadow, text, etc.
```

**Pros:**
- Perfect pixel consistency across platforms
- No bridge overhead (Dart compiles to native ARM)
- Can create custom designs without platform limitations
- 60/120 FPS more achievable (no bridge)

**Cons:**
- Larger app size (includes rendering engine)
- Doesn't auto-update with platform (iOS 18 new button style = manual update)
- Accessibility requires manual implementation
- Doesn't "feel" perfectly native without effort

### **Performance Architecture:**

**React Native:**
```
┌─────────────┐    Bridge/JSI    ┌──────────────┐
│ JavaScript  │ ←──────────────→ │ Native Layer │
│   Engine    │   (Serialization  │  (Platform)  │
│ (Hermes)    │    or Direct)     │  Rendering   │
└─────────────┘                   └──────────────┘
```

**Flutter:**
```
┌──────────────┐
│   Dart VM    │
│      ↓       │
│ Skia Engine  │  (All in one runtime)
│      ↓       │
│   Platform   │
└──────────────┘
```

Flutter's monolithic approach eliminates bridge overhead but loses native widget benefits.

### **Concrete Example - Button Touch:**

**React Native:**
```
1. Native touches UIButton
2. Touch event → Bridge → JavaScript
3. JS onPress handler executes
4. State updates → Bridge → Native
5. Native updates UIButton appearance
```

**Flutter:**
```
1. Gesture detector intercepts touch on canvas
2. Dart handler executes (no bridge)
3. Widget rebuilds
4. Skia redraws pixels
```

Flutter is faster here, but React Native can use native animation driver:

```javascript
// React Native optimization
<Animated.View
  style={{
    transform: [{
      scale: buttonScale.interpolate({
        inputRange: [0, 1],
        outputRange: [1, 0.95]
      })
    }]
  }}
>
  <TouchableOpacity
    onPressIn={() => {
      Animated.spring(buttonScale, {
        toValue: 1,
        useNativeDriver: true // Animation stays native!
      }).start();
    }}
  >
```

### **Development Experience:**

| Aspect | React Native | Flutter |
|--------|-------------|---------|
| **Language** | JavaScript/TypeScript | Dart |
| **Ecosystem** | Massive (npm) | Growing |
| **Learning Curve** | Lower (if know React) | Moderate |
| **Hot Reload** | Yes (Fast Refresh) | Yes (Hot Reload) |
| **Debugging** | Chrome DevTools, Flipper | Dart DevTools |
| **Web Support** | react-native-web | Flutter Web (first-class) |

### **When to Choose Each:**

**React Native if:**
- Want truly native UI components
- Have web React developers
- Need maximum npm ecosystem access
- Platform-specific design is okay/desired
- Building standard mobile apps

**Flutter if:**
- Need pixel-perfect cross-platform design
- Building games or highly custom UI
- Performance is critical (animations, transitions)
- Want web + mobile from same codebase
- Team can learn Dart

**Interview insight:**

*"React Native and Flutter take fundamentally different approaches. React Native acts as a bridge between JavaScript and platform-native components, giving you truly native widgets but with communication overhead. Flutter compiles Dart to native code and renders everything via Skia, eliminating the bridge but losing automatic platform UI updates. React Native is better for standard apps where native look-and-feel matters; Flutter excels at highly customized, performance-critical UIs where consistent cross-platform design is paramount. The New Architecture with JSI and Fabric narrows the performance gap considerably."*

---
