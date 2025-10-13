## **6. TurboModules vs Native Modules**

This distinction shows you understand the evolution of React Native's module system.

### **Native Modules (Legacy):**

```javascript
// JavaScript side:
import { NativeModules } from 'react-native';
const { CalendarModule } = NativeModules;

// All modules loaded at startup (eager loading)
// Even if you never use them!
CalendarModule.createEvent('Party', '2024-12-31');
```

```objc
// iOS side (Objective-C):
@interface CalendarModule : NSObject <RCTBridgeModule>
@end

@implementation CalendarModule
RCT_EXPORT_MODULE();

RCT_EXPORT_METHOD(createEvent:(NSString *)name date:(NSString *)date) {
  // Implementation
}
@end
```

**Problems with Native Modules:**

1. **Eager loading**: All modules loaded at app startup
   ```javascript
   // Even if you only use AsyncStorage,
   // CameraModule, GeolocationModule, etc. all load
   // Increases startup time and memory
   ```

2. **Async-only**: All calls go through the bridge
   ```javascript
   // Always asynchronous, even for simple getters
   CalendarModule.getAuthStatus((status) => {
     console.log(status); // Callback required
   });
   ```

3. **No type safety**: JavaScript has no way to know what methods exist
   ```javascript
   // Typo won't be caught until runtime
   CalendarModule.creatEvent(); // Oops, should be createEvent
   ```

### **TurboModules (New Architecture):**

```javascript
// JavaScript side with type safety:
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

interface Spec extends TurboModule {
  createEvent(name: string, date: string): Promise<number>;
  getAuthStatus(): string; // Can be sync!
}

export default TurboModuleRegistry.get<Spec>('CalendarModule');
```

```cpp
// Native side (C++ with JSI):
class CalendarModule : public NativeCalendarModuleSpec {
public:
  jsi::String getAuthStatus(jsi::Runtime& rt) {
    // Synchronous method! No bridge, immediate return
    return jsi::String::createFromUtf8(rt, "authorized");
  }
  
  jsi::Promise createEvent(
    jsi::Runtime& rt,
    jsi::String name,
    jsi::String date
  ) {
    // Can still be async when needed
    return createPromise(rt, [=]() {
      // Async work
    });
  }
};
```

### **Key Differences:**

| Feature | Native Modules | TurboModules |
|---------|---------------|--------------|
| **Loading** | Eager (all at startup) | Lazy (on-demand) |
| **Performance** | All async through bridge | Can be synchronous via JSI |
| **Type Safety** | None | TypeScript/Flow codegen |
| **Memory** | All loaded always | Loaded only when used |
| **Initialization** | Slower startup | Faster startup |

### **Lazy Loading Example:**

```javascript
// TurboModule - only loads when first accessed
const CalendarModule = require('./CalendarTurboModule');

// If user never opens calendar feature,
// CalendarModule never loads into memory!

function CalendarScreen() {
  // Module loads NOW, not at app startup
  const event = CalendarModule.createEvent(...);
}
```

### **Synchronous Methods:**

```javascript
// OLD (Native Modules) - always async:
NativeModules.DeviceInfo.getDeviceName((name) => {
  console.log(name); // Requires callback
});

// NEW (TurboModules) - can be sync:
const name = TurboModules.DeviceInfo.getDeviceName();
console.log(name); // Immediate return!
```

**Interview talking point:**

*"TurboModules are the JSI-powered evolution of Native Modules. The key improvements are lazy loading - modules only load when actually used, reducing startup time and memory; type-safe interfaces generated from specs; and the ability to provide synchronous methods via JSI instead of forcing everything through the async bridge. For large apps with many native modules, TurboModules can significantly improve performance and developer experience."*

---
