## **9. Optimizing Bridge Performance**

This is where you show practical expertise - how to actually build performant React Native apps.

### **Strategy 1: Use Native Driver for Animations**

**The Problem:**
```javascript
// ❌ BAD: Animation via setState
const BadAnimation = () => {
  const [position, setPosition] = useState(0);
  
  useEffect(() => {
    const interval = setInterval(() => {
      setPosition(p => p + 1); // Every frame!
      // 1. JS executes setState
      // 2. React reconciles
      // 3. Sends update across bridge
      // 4. Native updates view
      // Result: Janky, drops frames
    }, 16);
    return () => clearInterval(interval);
  }, []);
  
  return <View style={{ transform: [{ translateX: position }] }} />;
};
```

**The Solution:**
```javascript
// ✅ GOOD: Native driver animation
const GoodAnimation = () => {
  const translateX = useRef(new Animated.Value(0)).current;
  
  useEffect(() => {
    Animated.timing(translateX, {
      toValue: 300,
      duration: 1000,
      useNativeDriver: true, // Runs entirely on native side!
    }).start();
  }, []);
  
  return (
    <Animated.View 
      style={{ transform: [{ translateX }] }}
    />
  );
  // Bridge crossed once to start animation
  // Then native side handles all frames
  // 60 FPS guaranteed
};
```

**What useNativeDriver does:**
```
Without: JS updates value → Bridge → Native → Render (every frame)
With: JS sends animation config → Native runs animation loop → Render
```

### **Strategy 2: Optimize FlatList Rendering**

```javascript
// ❌ BAD: Causes excessive bridge traffic
const BadList = ({ items }) => {
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <View style={{ padding: 10 }}> {/* Inline style = new object every render */}
          <Text>{item.title}</Text>
          <Image source={{ uri: item.image }} /> {/* No caching */}
        </View>
      )}
      // Missing optimizations
    />
  );
};

// ✅ GOOD: Minimizes bridge crossings
const styles = StyleSheet.create({
  item: { padding: 10 },
});

const ListItem = React.memo(({ item }) => (
  <View style={styles.item}>
    <Text>{item.title}</Text>
    <Image 
      source={{ uri: item.image }}
      resizeMode="cover"
    />
  </View>
));

const GoodList = ({ items }) => {
  const keyExtractor = useCallback((item) => item.id, []);
  
  const renderItem = useCallback(({ item }) => (
    <ListItem item={item} />
  ), []);
  
  return (
    <FlatList
      data={items}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      removeClippedSubviews={true} // Unmounts off-screen views
      maxToRenderPerBatch={10} // Controls initial render
      windowSize={5} // How many screens to render
      initialNumToRender={10}
      getItemLayout={(data, index) => ({ // Skips measurement
        length: ITEM_HEIGHT,
        offset: ITEM_HEIGHT * index,
        index,
      })}
    />
  );
};
```

### **Strategy 3: Batch Bridge Updates**

```javascript
// ❌ BAD: Multiple bridge crossings
const BadComponent = () => {
  const [count, setCount] = useState(0);
  const [text, setText] = useState('');
  const [visible, setVisible] = useState(false);
  
  const handlePress = () => {
    setCount(c => c + 1);     // Bridge crossing
    setText('Updated');        // Bridge crossing
    setVisible(true);          // Bridge crossing
    // Three separate renders, three bridge crossings
  };
  
  return (
    <TouchableOpacity onPress={handlePress}>
      {/* ... */}
    </TouchableOpacity>
  );
};

// ✅ GOOD: Single batched update
const GoodComponent = () => {
  const [state, setState] = useState({
    count: 0,
    text: '',
    visible: false
  });
  
  const handlePress = () => {
    setState({
      count: state.count + 1,
      text: 'Updated',
      visible: true
    });
    // Single state update = one bridge crossing
  };
  
  return (
    <TouchableOpacity onPress={handlePress}>
      {/* ... */}
    </TouchableOpacity>
  );
};
```

### **Strategy 4: InteractionManager for Non-Critical Updates**

```javascript
// ✅ Defer heavy work until animations complete
import { InteractionManager } from 'react-native';

const SmartComponent = () => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    // Critical: Show screen immediately
    navigation.navigate('DetailScreen');
    
    // Defer heavy operation until animation done
    InteractionManager.runAfterInteractions(() => {
      const expensiveData = processLargeDataset();
      setData(expensiveData);
      // Bridge not overwhelmed during navigation animation
    });
  }, []);
  
  return data ? <DataView data={data} /> : <Skeleton />;
};
```

### **Strategy 5: Minimize Touch Event Serialization**

```javascript
// ❌ BAD: Every scroll event crosses bridge
<ScrollView
  onScroll={(event) => {
    console.log(event.nativeEvent.contentOffset.y);
    // Fires 100+ times per second = 100+ serializations
  }}
/>

// ✅ GOOD: Throttle events
<ScrollView
  onScroll={(event) => {
    console.log(event.nativeEvent.contentOffset.y);
  }}
  scrollEventThrottle={16} // Max once per frame (60 FPS)
/>

// ✅ BETTER: Use Animated.event (no JS bridge crossing)
const scrollY = useRef(new Animated.Value(0)).current;

<Animated.ScrollView
  onScroll={Animated.event(
    [{ nativeEvent: { contentOffset: { y: scrollY } } }],
    { useNativeDriver: true } // Native side only!
  )}
/>
```

### **Strategy 6: Optimize Images**

```javascript
// ❌ BAD: Large images serialized across bridge
<Image 
  source={{ uri: 'https://example.com/huge-image.jpg' }} // 5MB image
  style={{ width: 100, height: 100 }} // Displayed tiny!
/>

// ✅ GOOD: Proper image optimization
<Image 
  source={{ 
    uri: 'https://example.com/huge-image.jpg',
    cache: 'force-cache' // Native caching
  }}
  style={{ width: 100, height: 100 }}
  resizeMode="cover"
  // Better: Use CDN with size parameters
  // source={{ uri: 'https://cdn.com/image.jpg?w=100&h=100' }}
/>

// ✅ BEST: Use FastImage for better caching
import FastImage from 'react-native-fast-image';

<FastImage
  source={{
    uri: 'https://example.com/image.jpg',
    priority: FastImage.priority.high,
  }}
  style={{ width: 100, height: 100 }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

### **Strategy 7: Use Native Modules Wisely**

```javascript
// ❌ BAD: Frequent native calls
const BadLocationTracking = () => {
  useEffect(() => {
    const interval = setInterval(() => {
      Geolocation.getCurrentPosition((position) => {
        // Bridge crossing every second
        updateLocation(position);
      });
    }, 1000);
    return () => clearInterval(interval);
  }, []);
};

// ✅ GOOD: Let native handle high-frequency updates
const GoodLocationTracking = () => {
  useEffect(() => {
    const watchId = Geolocation.watchPosition(
      (position) => updateLocation(position),
      (error) => console.error(error),
      { 
        enableHighAccuracy: true,
        distanceFilter: 10, // Only update when moved 10m
        // Native filters updates, only sends relevant ones
      }
    );
    return () => Geolocation.clearWatch(watchId);
  }, []);
};
```

### **Strategy 8: Leverage React.memo and useMemo**

```javascript
// ❌ BAD: Unnecessary bridge traffic from re-renders
const BadParent = () => {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Button onPress={() => setCount(c => c + 1)} />
      <ExpensiveChild data={someData} />
      {/* Re-renders on every count change even though data unchanged */}
    </View>
  );
};

// ✅ GOOD: Prevent unnecessary bridge crossings
const ExpensiveChild = React.memo(({ data }) => {
  return <ComplexView data={data} />;
});

const GoodParent = () => {
  const [count, setCount] = useState(0);
  const memoizedData = useMemo(() => processData(), []);
  
  return (
    <View>
      <Button onPress={() => setCount(c => c + 1)} />
      <ExpensiveChild data={memoizedData} />
      {/* Only re-renders when data actually changes */}
    </View>
  );
};
```

### **Strategy 9: Use Hermes and Enable Optimization**

```javascript
// android/gradle.properties
hermesEnabled=true

// iOS Podfile
use_react_native!(
  :hermes_enabled => true
)

// Enables:
// - Faster startup (less bridge initialization time)
// - Lower memory (less GC bridge traffic)
// - Better performance overall
```

### **Strategy 10: Profile and Measure**

```javascript
// Use React DevTools Profiler
import { Profiler } from 'react';

const onRenderCallback = (
  id, phase, actualDuration, baseDuration, startTime, commitTime
) => {
  console.log(`${id} took ${actualDuration}ms`);
};

<Profiler id="MyComponent" onRender={onRenderCallback}>
  <MyComponent />
</Profiler>

// Use Flipper to inspect:
// - Bridge traffic
// - React DevTools
// - Performance metrics
// - Network requests

// Use Systrace/Instruments:
// See actual thread utilization
// Identify bridge bottlenecks
```

### **Quick Reference - Bridge Optimization Checklist:**

✅ **Animations**: Use `useNativeDriver: true`  
✅ **Lists**: Implement `getItemLayout`, `windowSize`, `memo`  
✅ **Images**: Use appropriate sizes, caching, FastImage  
✅ **Events**: Throttle with `scrollEventThrottle`  
✅ **State**: Batch updates, use reducers  
✅ **Styles**: Use `StyleSheet.create()` not inline objects  
✅ **Native Modules**: Minimize call frequency  
✅ **Defer**: Use `InteractionManager` for non-critical work  
✅ **Memory**: Memoize expensive computations  
✅ **Engine**: Use Hermes for better performance  

**Interview closing insight:**

*"Bridge performance optimization in React Native centers on minimizing serialization and communication between JavaScript and native threads. The most impactful techniques are using the native animation driver to keep animations off the bridge entirely, optimizing FlatList rendering to reduce unnecessary re-renders, batching state updates, and throttling high-frequency events. With the New Architecture's JSI and Fabric, many of these concerns are lessened, but understanding bridge optimization remains crucial for maintaining performant React Native applications, especially on lower-end devices."*

---
