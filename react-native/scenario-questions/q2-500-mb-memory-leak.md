# Question 2: You notice your app is using 500MB of memory and eventually crashes after scrolling through several screens with images. How do you diagnose and fix this?

## **Interview Approach:**

*"500MB is definitely a memory leak. I'd use a systematic debugging approach: identify what's consuming memory, find the leak source, then fix it. Let me walk through this."*

---

## **Phase 1: Diagnose Memory Usage**

### **Step 1: Profile Memory with Tools**

```javascript
// Enable memory profiling
import { PerformanceObserver } from 'react-native-performance';

// Monitor memory continuously
const memoryMonitor = setInterval(() => {
  if (global.performance && global.performance.memory) {
    const memoryUsage = global.performance.memory.usedJSHeapSize / 1048576;
    console.log(`JS Memory: ${memoryUsage.toFixed(2)}MB`);
    
    if (memoryUsage > 300) {
      console.error('Memory threshold exceeded!');
      // Trigger memory cleanup or warning
    }
  }
}, 5000);

// Use React DevTools Profiler
import { Profiler } from 'react';

<Profiler id="MemoryLeakyScreen" onRender={(id, phase, actualDuration) => {
  // Track render counts - if increasing without unmounting, likely leak
  console.log(`${id} rendered in ${phase}: ${actualDuration}ms`);
}}>
  <LeakyScreen />
</Profiler>
```

**Using Flipper:**

```javascript
// Flipper → Plugins → Hermes Debugger → Memory
// Take heap snapshots before and after scrolling
// Compare snapshots to find retained objects

// Look for:
// 1. Detached DOM nodes (native views not released)
// 2. Closure retaining large objects
// 3. Event listeners not removed
// 4. Images not released from memory
```

### **Step 2: Identify Leak Pattern**

```javascript
// Navigate through app and monitor memory
const testMemoryLeak = async () => {
  const measurements = [];
  
  // Baseline
  measurements.push({ screen: 'Home', memory: getMemory() });
  
  // Navigate to image-heavy screen
  navigation.navigate('Gallery');
  await wait(2000);
  measurements.push({ screen: 'Gallery', memory: getMemory() });
  
  // Scroll through images
  for (let i = 0; i < 10; i++) {
    scrollToIndex(i * 10);
    await wait(1000);
    measurements.push({ screen: `Gallery-${i}`, memory: getMemory() });
  }
  
  // Go back
  navigation.goBack();
  await wait(2000);
  measurements.push({ screen: 'Home-return', memory: getMemory() });
  
  // Force garbage collection
  if (global.gc) global.gc();
  await wait(2000);
  measurements.push({ screen: 'Home-after-GC', memory: getMemory() });
  
  console.table(measurements);
  
  // Analysis:
  // If memory doesn't drop after goBack() and GC, there's a leak
};

/*
Expected output without leak:
| Screen       | Memory |
|--------------|--------|
| Home         | 50MB   |
| Gallery      | 150MB  | ← Spike is normal (images loaded)
| Gallery-5    | 180MB  |
| Gallery-10   | 200MB  |
| Home-return  | 60MB   | ← Should drop!
| Home-after-GC| 55MB   | ← Back to baseline

Actual output with leak:
| Screen       | Memory |
|--------------|--------|
| Home         | 50MB   |
| Gallery      | 150MB  |
| Gallery-5    | 250MB  | ← Growing!
| Gallery-10   | 380MB  | ← Still growing!
| Home-return  | 370MB  | ← Didn't drop! LEAK!
| Home-after-GC| 365MB  | ← Objects retained! LEAK!
*/
```

---

## **Phase 2: Common Memory Leak Causes**

### **Leak #1: Images Not Released**

```javascript
// ❌ PROBLEM: Images decoded and kept in memory
const LeakyGallery = () => {
  const [images] = useState(Array(1000).fill(null).map((_, i) => ({
    id: i,
    uri: `https://example.com/image${i}.jpg` // Each 5MB original
  })));
  
  return (
    <FlatList
      data={images}
      renderItem={({ item }) => (
        <Image
          source={{ uri: item.uri }}
          style={{ width: 400, height: 400 }}
          // Default Image component:
          // - Downloads full 5MB image
          // - Decodes to bitmap: 400 × 400 × 4 bytes = 640KB in memory
          // - Keeps in memory even after scrolling away
          // - 1000 images × 640KB = 640MB!
        />
      )}
    />
  );
};

// ✅ FIX 1: Use FastImage with proper memory management
import FastImage from 'react-native-fast-image';

const FixedGallery = () => {
  return (
    <FlatList
      data={images}
      renderItem={({ item }) => (
        <FastImage
          source={{
            uri: item.uri,
            priority: FastImage.priority.normal,
            cache: FastImage.cacheControl.web, // Smart caching
          }}
          style={{ width: 400, height: 400 }}
          resizeMode={FastImage.resizeMode.cover}
        />
      )}
      // Crucial: Unmount off-screen items
      removeClippedSubviews={true}
      maxToRenderPerBatch={5}
      windowSize={3}
      initialNumToRender={5}
    />
  );
};

// ✅ FIX 2: Clear image cache periodically
useEffect(() => {
  return () => {
    // Clear cache when leaving screen
    FastImage.clearMemoryCache();
  };
}, []);

// ✅ FIX 3: Use appropriately sized images
const optimizedUri = `${item.uri}?w=800&h=800&q=80`;
// Server/CDN resizes: 5MB → 200KB
// Memory usage: 640KB → 50KB per image
```

### **Leak #2: Event Listeners Not Removed**

```javascript
// ❌ PROBLEM: Listeners keep component alive
const LeakyScreen = () => {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    // Subscribe to updates
    const subscription = DataService.subscribe((newData) => {
      setData(newData); // Closure captures component state
    });
    
    // Listen to app state
    AppState.addEventListener('change', (state) => {
      console.log('App state:', state);
      // Another closure
    });
    
    // WebSocket listener
    WebSocketService.on('message', (msg) => {
      setData(prev => [...prev, msg]);
      // Captures setData, which captures component
    });
    
    // FORGOT TO CLEANUP!
    // Component unmounts but listeners remain
    // Closures keep component in memory
  }, []);
  
  return <FlatList data={data} />;
};

// After navigating away:
// Component "unmounted" but:
// - DataService subscription holds reference to setData
// - AppState listener holds reference to console.log
// - WebSocket listener holds reference to setData
// → Component can't be garbage collected
// → Memory leak!

// ✅ FIX: Always cleanup
const FixedScreen = () => {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    const dataSubscription = DataService.subscribe(setData);
    
    const appStateListener = AppState.addEventListener('change', (state) => {
      console.log('App state:', state);
    });
    
    const messageHandler = (msg) => {
      setData(prev => [...prev, msg]);
    };
    WebSocketService.on('message', messageHandler);
    
    return () => {
      // Cleanup ALL listeners
      dataSubscription.unsubscribe();
      appStateListener.remove();
      WebSocketService.off('message', messageHandler);
    };
  }, []);
  
  return <FlatList data={data} />;
};
```

### **Leak #3: Timers Not Cleared**

```javascript
// ❌ PROBLEM: Intervals run forever
const LeakyComponent = () => {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    setInterval(() => {
      setCount(c => c + 1);
      // Runs forever, even after unmount!
    }, 1000);
    
    // User scrolls through 50 screens with this component
    // 50 intervals running = 50 × memory
  }, []);
  
  return <Text>{count}</Text>;
};

// ✅ FIX: Clear timers
const FixedComponent = () => {
  const [count, setCount] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      setCount(c => c + 1);
    }, 1000);
    
    return () => clearInterval(interval);
  }, []);
  
  return <Text>{count}</Text>;
};
```

### **Leak #4: Closures Capturing Large Objects**

```javascript
// ❌ PROBLEM: Closure holds reference to huge dataset
const LeakyScreen = ({ route }) => {
  // Large data passed via navigation params
  const hugeDataset = route.params.data; // 50MB array of objects
  
  const handleItemPress = (itemId) => {
    // This closure captures entire hugeDataset
    const item = hugeDataset.find(i => i.id === itemId);
    
    setTimeout(() => {
      // Even after navigating away, this timeout holds hugeDataset
      console.log(item);
    }, 5000);
  };
  
  return (
    <FlatList
      data={hugeDataset}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => handleItemPress(item.id)}>
          <Text>{item.title}</Text>
        </TouchableOpacity>
      )}
    />
  );
};

// Navigate through 10 screens → 10 × 50MB = 500MB!

// ✅ FIX 1: Don't capture large objects
const FixedScreen = ({ route }) => {
  const hugeDataset = route.params.data;
  
  // Store in ref instead of closure
  const dataRef = useRef(hugeDataset);
  
  useEffect(() => {
    dataRef.current = hugeDataset;
  }, [hugeDataset]);
  
  const handleItemPress = useCallback((itemId) => {
    // Only captures itemId, not entire dataset
    const item = dataRef.current.find(i => i.id === itemId);
    
    const timeoutId = setTimeout(() => {
      console.log(item);
    }, 5000);
    
    // Clear timeout on unmount
    return () => clearTimeout(timeoutId);
  }, []);
  
  return (
    <FlatList
      data={hugeDataset}
      renderItem={({ item }) => (
        <TouchableOpacity onPress={() => handleItemPress(item.id)}>
          <Text>{item.title}</Text>
        </TouchableOpacity>
      )}
    />
  );
};

// ✅ FIX 2: Don't pass huge data through navigation
// Instead, pass ID and fetch from global state/cache
const BetterApproach = ({ route }) => {
  const dataId = route.params.dataId; // Just ID, not data
  const data = useSelector(state => state.data[dataId]); // Get from Redux/Context
  
  return <FlatList data={data} />;
};
```

### **Leak #5: React Navigation Screen Stack**

```javascript
// ❌ PROBLEM: Navigation keeps all screens mounted
<Stack.Navigator>
  <Stack.Screen name="Home" component={Home} />
  <Stack.Screen name="ImageGallery" component={ImageGallery} />
  <Stack.Screen name="Profile" component={Profile} />
</Stack.Navigator>

// User flow:
// Home → ImageGallery (loads 100 images, 200MB)
// → Profile (loads data, 50MB)
// → ImageGallery again (loads more images, 200MB)
// → Profile again (50MB)

// All screens stay mounted!
// Total: 500MB

// ✅ FIX 1: Enable screen unmounting
<Stack.Navigator
  screenOptions={{
    detachPreviousScreen: true, // Unmount when not visible
  }}
>
  <Stack.Screen name="Home" component={Home} />
  <Stack.Screen name="ImageGallery" component={ImageGallery} />
</Stack.Navigator>

// ✅ FIX 2: Use react-native-screens for better memory
import { enableScreens } from 'react-native-screens';
enableScreens(true);

// Enables native screen optimization
// Unmounts off-screen content more aggressively

// ✅ FIX 3: Cleanup on blur
const ImageGallery = ({ navigation }) => {
  const [images, setImages] = useState([]);
  
  useEffect(() => {
    const unsubscribe = navigation.addListener('blur', () => {
      // Screen is no longer focused
      console.log('Screen blurred - cleanup!');
      
      // Clear images from memory
      setImages([]);
      
      // Clear image cache
      FastImage.clearMemoryCache();
    });
    
    return unsubscribe;
  }, [navigation]);
  
  useEffect(() => {
    const unsubscribe = navigation.addListener('focus', () => {
      // Screen focused - load images
      loadImages();
    });
    
    return unsubscribe;
  }, [navigation]);
  
  return <FlatList data={images} />;
};
```

### **Leak #6: Redux/Context Subscribers**

```javascript
// ❌ PROBLEM: Context updates cause entire tree to re-render
const DataContext = createContext();

const DataProvider = ({ children }) => {
  const [hugeData, setHugeData] = useState([]); // 50MB
  
  return (
    <DataContext.Provider value={{ hugeData, setHugeData }}>
      {children}
    </DataContext.Provider>
  );
};

// Every component using context re-renders when hugeData changes
// Each component keeps reference to hugeData
// Memory multiplies!

// ✅ FIX 1: Split contexts
const DataProvider = ({ children }) => {
  const [data, setData] = useState([]);
  const [metadata, setMetadata] = useState({});
  
  return (
    <DataContext.Provider value={{ data, setData }}>
      <MetadataContext.Provider value={{ metadata, setMetadata }}>
        {children}
      </MetadataContext.Provider>
    </DataContext.Provider>
  );
};

// Components only subscribe to what they need

// ✅ FIX 2: Use selector pattern
const useDataSelector = (selector) => {
  const data = useContext(DataContext);
  return useMemo(() => selector(data), [data, selector]);
};

// Component only re-renders if selected data changes
const MyComponent = () => {
  const itemCount = useDataSelector(data => data.items.length);
  // Only re-renders if itemCount changes, not entire data
};

// ✅ FIX 3: Use libraries designed for this (Redux, Zustand)
import create from 'zustand';

const useStore = create((set) => ({
  data: [],
  setData: (data) => set({ data }),
}));

// Better performance, automatic cleanup
```

---

## **Phase 3: Systematic Fix Implementation**

### **Complete Memory-Safe Implementation**

```javascript
import React, { useEffect, useCallback, useRef, useState } from 'react';
import { FlatList, AppState } from 'react-native';
import FastImage from 'react-native-fast-image';
import { enableScreens } from 'react-native-screens';

// Enable native screen optimization
enableScreens(true);

// Memoized item component
const ImageItem = React.memo(({ item, onPress }) => {
  // Use optimized image URL
  const imageUrl = useMemo(
    () => `${item.uri}?w=800&h=800&q=80`,
    [item.uri]
  );
  
  return (
    <TouchableOpacity onPress={() => onPress(item.id)}>
      <FastImage
        source={{
          uri: imageUrl,
          priority: FastImage.priority.normal,
          cache: FastImage.cacheControl.web,
        }}
        style={styles.image}
        resizeMode={FastImage.resizeMode.cover}
      />
      <Text>{item.title}</Text>
    </TouchableOpacity>
  );
});

const MemorySafeGallery = ({ navigation, route }) => {
  const [images, setImages] = useState([]);
  const [loading, setLoading] = useState(true);
  
  // Use ref to avoid closure capturing stale data
  const imagesRef = useRef(images);
  useEffect(() => {
    imagesRef.current = images;
  }, [images]);
  
  // Load images
  useEffect(() => {
    let cancelled = false;
    
    const loadImages = async () => {
      try {
        const data = await fetchImages(route.params.galleryId);
        if (!cancelled) {
          setImages(data);
          setLoading(false);
        }
      } catch (error) {
        console.error(error);
      }
    };
    
    loadImages();
    
    return () => {
      cancelled = true;
    };
  }, [route.params.galleryId]);
  
  // Cleanup on blur
  useEffect(() => {
    const unsubscribeBlur = navigation.addListener('blur', () => {
      console.log('Gallery blurred - cleanup');
      
      // Clear images
      setImages([]);
      
      // Clear image cache
      FastImage.clearMemoryCache();
    });
    
    return unsubscribeBlur;
  }, [navigation]);
  
  // Handle app state changes
  useEffect(() => {
    const subscription = AppState.addEventListener('change', (nextAppState) => {
      if (nextAppState === 'background') {
        // App going to background - aggressive cleanup
        FastImage.clearMemoryCache();
      }
    });
    
    return () => {
      subscription.remove();
    };
  }, []);
  
  // Stable callbacks
  const handleItemPress = useCallback((itemId) => {
    navigation.navigate('ImageDetail', { imageId: itemId });
  }, [navigation]);
  
  const keyExtractor = useCallback((item) => item.id, []);
  
  const renderItem = useCallback(({ item }) => {
    return <ImageItem item={item} onPress={handleItemPress} />;
  }, [handleItemPress]);
  
  if (loading) {
    return <LoadingSpinner />;
  }
  
  return (
    <FlatList
      data={images}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      
      // Memory optimization
      removeClippedSubviews={true}
      maxToRenderPerBatch={5}
      windowSize={3}
      initialNumToRender={5}
      
      // Prevent unnecessary re-renders
      getItemLayout={(data, index) => ({
        length: 300,
        offset: 300 * index,
        index,
      })}
    />
  );
};

// Navigator configuration
const App = () => (
  <NavigationContainer>
    <Stack.Navigator
      screenOptions={{
        // Unmount previous screens
        detachPreviousScreen: true,
        
        // Clear transient state on blur
        freezeOnBlur: true,
      }}
    >
      <Stack.Screen name="Home" component={Home} />
      <Stack.Screen name="Gallery" component={MemorySafeGallery} />
      <Stack.Screen name="ImageDetail" component={ImageDetail} />
    </Stack.Navigator>
  </NavigationContainer>
);

export default App;
```

---

## **Phase 4: Monitor and Validate**

```javascript
// Create memory monitor component
const MemoryMonitor = () => {
  const [memory, setMemory] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      if (global.performance?.memory) {
        const mb = global.performance.memory.usedJSHeapSize / 1048576;
        setMemory(mb.toFixed(2));
        
        if (mb > 300) {
          console.warn('High memory usage:', mb, 'MB');
          // Could trigger cleanup or show warning to user
        }
      }
    }, 2000);
    
    return () => clearInterval(interval);
  }, []);
  
  // Show in dev mode
  if (__DEV__) {
    return (
      <View style={styles.memoryMonitor}>
        <Text>Memory: {memory} MB</Text>
      </View>
    );
  }
  
  return null;
};

// Add to app root
const App = () => (
  <>
    <NavigationContainer>
      {/* ... */}
    </NavigationContainer>
    <MemoryMonitor />
  </>
);
```

---

### **Results After Fixes:**

```
Before:
- Home: 50MB
- Gallery (100 images): 400MB
- Profile: 450MB
- Back to Home: 420MB (leaked!)
- Crash after visiting 3-4 screens

After:
- Home: 50MB
- Gallery (100 images): 150MB
- Profile: 80MB
- Back to Home: 60MB (cleaned up!)
- Stable memory across 20+ screen visits
```

---
