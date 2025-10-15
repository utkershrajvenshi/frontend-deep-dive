# You have a FlatList with 10,000 items, each containing an image and some text. The list is scrolling slowly and users are complaining. Walk me through your debugging and optimization process

Excellent question! This is a **real-world performance problem** that tests your systematic debugging approach, understanding of React Native internals, and practical optimization skills. Let me walk you through this like I'm solving it in a live interview.

---

## **Interview Approach: Structured Problem-Solving**

*"Great question. I'd approach this systematically - first measure and identify the bottleneck, then apply targeted optimizations. Let me walk through my process."*

---

## **Phase 1: Gather Information & Reproduce**

### **Initial Questions to Ask (Interview Setting):**

Before diving in, I'd clarify:

```javascript
// Understanding the problem scope:
1. "What device/OS are we seeing this on? Low-end Android, iPhone, both?"
2. "Is the slowness during initial render, scrolling, or both?"
3. "Are the images local or remote URLs?"
4. "What's the approximate size of each image?"
5. "Is this debug or release build?"
6. "Are we using Hermes?"
```

These questions show you understand that performance issues can have different causes.

### **Reproduce and Measure:**

```javascript
// Enable performance monitoring
import { LogBox, InteractionManager } from 'react-native';

// 1. Check FPS during scroll
// React Native Performance Monitor (dev menu → "Show Perf Monitor")
// Should see: JS frame rate and UI frame rate

// 2. Enable debug logging
const ItemComponent = React.memo(({ item }) => {
  console.log('Rendering item:', item.id); // See how often items re-render
  return (
    <View>
      <Image source={{ uri: item.imageUrl }} />
      <Text>{item.text}</Text>
    </View>
  );
});

// 3. Use Profiler to measure render times
import { Profiler } from 'react';

<Profiler id="FlatList" onRender={(id, phase, actualDuration) => {
  if (actualDuration > 16) { // 60 FPS = 16.67ms per frame
    console.warn(`FlatList render took ${actualDuration}ms`);
  }
}}>
  <FlatList data={items} renderItem={renderItem} />
</Profiler>
```

**What to look for:**
- **JS FPS < 60**: JavaScript thread bottleneck (component rendering)
- **UI FPS < 60**: Native thread bottleneck (image loading, native rendering)
- **Both low**: Likely bridge saturation or memory pressure

---

## **Phase 2: Identify the Bottleneck**

Let me analyze common culprits:

### **Bottleneck #1: Excessive Re-renders**

```javascript
// ❌ PROBLEM: Every item re-renders on every scroll
const BadList = () => {
  const [items] = useState(generateItems(10000));
  
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        // Inline component - new function every render!
        <View style={{ padding: 10 }}> {/* Inline style - new object! */}
          <Image source={{ uri: item.imageUrl }} /> {/* New object! */}
          <Text>{item.text}</Text>
        </View>
      )}
      // Missing optimizations
    />
  );
};

// Measure: Use React DevTools Profiler
// Symptom: JS FPS drops, many "Render" events
```

**Root cause analysis:**
```javascript
// Problem 1: renderItem creates new component instance each render
// FlatList re-renders → New renderItem function → All items re-render

// Problem 2: Inline styles create new objects
// React sees different object reference → Thinks props changed → Re-renders

// Problem 3: Image source creates new object
// { uri: item.imageUrl } is new object each time
```

---

### **Bottleneck #2: Image Loading Performance**

```javascript
// ❌ PROBLEM: Large images loaded at full resolution
const BadImageItem = ({ item }) => {
  return (
    <View>
      <Image 
        source={{ uri: item.imageUrl }} // 4000x3000 image for 100x100 view!
        style={{ width: 100, height: 100 }}
      />
      <Text>{item.text}</Text>
    </View>
  );
};

// Measure: Check network tab, memory usage
// Symptom: UI FPS drops, memory grows, images load slowly
```

**Root cause:**
- Downloading full-resolution images (4MB each)
- React Native Image component decoding large images
- Memory pressure from many decoded bitmaps
- No caching strategy

---

### **Bottleneck #3: Layout Calculations**

```javascript
// ❌ PROBLEM: Unpredictable item heights
const BadList = () => (
  <FlatList
    data={items}
    renderItem={({ item }) => (
      <View>
        <Image 
          source={{ uri: item.imageUrl }}
          style={{ width: '100%', aspectRatio: item.aspectRatio }} // Variable height!
        />
        <Text>{item.text}</Text> {/* Variable text length */}
      </View>
    )}
    // No getItemLayout - FlatList must measure every item
  />
);

// Measure: Use Systrace/Profiler
// Symptom: Shadow thread busy, janky scroll, especially on Android
```

**Root cause:**
- Shadow thread calculating layout for each item
- FlatList can't predict scroll position
- Must mount items to measure them
- Re-layouts during scroll

---

### **Bottleneck #4: Too Many Items Rendered**

```javascript
// ❌ PROBLEM: Default window size too large
const BadList = () => (
  <FlatList
    data={items}
    renderItem={renderItem}
    // Using defaults:
    // initialNumToRender: 10
    // maxToRenderPerBatch: 10
    // windowSize: 21
    // With 10,000 items, this can render 200+ items off-screen!
  />
);

// Measure: Component tree size, memory
// Symptom: High memory, slow initial render, sluggish scroll
```

**Root cause:**
- Too many items mounted simultaneously
- Each item holds memory (components + images)
- React reconciliation slower with more nodes

---

## **Phase 3: Apply Optimizations (Systematic Approach)**

Now let's fix each issue systematically:

---

## **Optimization #1: Prevent Unnecessary Re-renders**

### **Step 1: Memoize the Item Component**

```javascript
// ✅ OPTIMIZATION: Memoize list items
const ListItem = React.memo(({ item }) => {
  // Only re-renders if item reference changes
  return (
    <View style={styles.item}>
      <Image 
        source={{ uri: item.imageUrl }}
        style={styles.image}
      />
      <Text style={styles.text}>{item.text}</Text>
    </View>
  );
}, (prevProps, nextProps) => {
  // Custom comparison - only re-render if these changed
  return (
    prevProps.item.id === nextProps.item.id &&
    prevProps.item.imageUrl === nextProps.item.imageUrl &&
    prevProps.item.text === nextProps.item.text
  );
});

// Use StyleSheet.create outside component
const styles = StyleSheet.create({
  item: {
    padding: 10,
    flexDirection: 'row',
    backgroundColor: '#fff',
  },
  image: {
    width: 100,
    height: 100,
    borderRadius: 8,
  },
  text: {
    flex: 1,
    fontSize: 16,
    marginLeft: 10,
  },
});
```

**Why this works:**
```javascript
// Without memo:
FlatList renders → renderItem called for ALL visible items
→ Each item component re-renders (even if data unchanged)

// With memo:
FlatList renders → React checks if item props changed
→ If unchanged: Skip render, reuse previous result
→ If changed: Re-render only that item

// Performance impact:
// 50 visible items × 16ms render time = 800ms (janky!)
// With memo: 2 changed items × 16ms = 32ms (smooth!)
```

### **Step 2: Optimize Callbacks**

```javascript
// ❌ BAD: New function on every render
const BadList = () => {
  return (
    <FlatList
      data={items}
      renderItem={({ item }) => (
        <ListItem 
          item={item}
          onPress={() => handlePress(item)} // New function every render!
        />
      )}
    />
  );
};

// ✅ GOOD: Stable callback
const ListItem = React.memo(({ item, onPress }) => (
  <TouchableOpacity onPress={onPress}>
    <Image source={{ uri: item.imageUrl }} />
    <Text>{item.text}</Text>
  </TouchableOpacity>
));

const GoodList = () => {
  const handlePress = useCallback((itemId) => {
    // Handle press with item ID, not full item object
    console.log('Pressed:', itemId);
    navigation.navigate('Detail', { itemId });
  }, [navigation]);
  
  const renderItem = useCallback(({ item }) => (
    <ListItem 
      item={item}
      onPress={() => handlePress(item.id)} // Still creates function, but...
    />
  ), [handlePress]);
  
  return (
    <FlatList
      data={items}
      renderItem={renderItem} // Stable function reference
      keyExtractor={keyExtractor}
    />
  );
};

// ✅ BETTER: Pass ID only, use context or ref for handler
const ListItem = React.memo(({ item }) => {
  const handlePress = useListItemPress(); // From context
  
  return (
    <TouchableOpacity onPress={() => handlePress(item.id)}>
      <Image source={{ uri: item.imageUrl }} />
      <Text>{item.text}</Text>
    </TouchableOpacity>
  );
});
```

### **Step 3: Stable Key Extractor**

```javascript
// ❌ BAD: Default key extractor uses index
<FlatList data={items} /> // Keys: 0, 1, 2, 3...
// Problem: If list reorders, React confuses items

// ✅ GOOD: Unique, stable keys
const keyExtractor = useCallback((item) => item.id, []);
// Or define outside component if it doesn't depend on state:
const keyExtractor = (item) => item.id;

<FlatList
  data={items}
  keyExtractor={keyExtractor}
/>

// Keys should be:
// - Unique: No duplicates
// - Stable: Same item = same key across renders
// - String/number: Not objects or arrays
```

---

## **Optimization #2: Optimize Image Loading**

### **Step 1: Use Appropriately Sized Images**

```javascript
// ❌ BAD: Loading full resolution
<Image 
  source={{ uri: 'https://cdn.com/image.jpg' }} // 4000x3000, 4MB
  style={{ width: 100, height: 100 }}
/>

// ✅ GOOD: Request thumbnail size from API/CDN
<Image 
  source={{ 
    uri: 'https://cdn.com/image.jpg?w=200&h=200&quality=80' // Optimized
  }}
  style={{ width: 100, height: 100 }}
/>

// Or use srcset-style multiple sizes
const getOptimizedImageUrl = (url, width) => {
  const pixelRatio = PixelRatio.get();
  const targetWidth = width * pixelRatio; // Account for device pixel density
  return `${url}?w=${targetWidth}&q=80`;
};

<Image 
  source={{ uri: getOptimizedImageUrl(item.imageUrl, 100) }}
  style={{ width: 100, height: 100 }}
/>
```

**Impact:**
```
Before: 4000×3000 image = 4MB download, 45MB decoded in memory
After: 200×200 image = 40KB download, 0.15MB in memory
Result: 100x less memory, 100x faster download
```

### **Step 2: Use Fast Image Library**

```javascript
// ❌ Standard Image component: Limited caching
import { Image } from 'react-native';

// ✅ Use react-native-fast-image: Better caching
import FastImage from 'react-native-fast-image';

const ListItem = React.memo(({ item }) => (
  <View style={styles.item}>
    <FastImage
      source={{
        uri: item.imageUrl,
        priority: FastImage.priority.normal,
        cache: FastImage.cacheControl.immutable, // Aggressive caching
      }}
      style={styles.image}
      resizeMode={FastImage.resizeMode.cover}
    />
    <Text>{item.text}</Text>
  </View>
));

// Preload images ahead of time
useEffect(() => {
  const urlsToPreload = items.slice(0, 20).map(item => ({
    uri: item.imageUrl,
    priority: FastImage.priority.high,
  }));
  
  FastImage.preload(urlsToPreload);
}, [items]);
```

**Why FastImage is better:**
- Uses SDWebImage (iOS) / Glide (Android) native libraries
- Better disk caching
- Placeholder support
- Priority queuing
- Fade-in animations
- Memory-efficient

### **Step 3: Implement Progressive Loading**

```javascript
// ✅ Show placeholder while loading
const ListItem = React.memo(({ item }) => {
  const [imageLoaded, setImageLoaded] = useState(false);
  
  return (
    <View style={styles.item}>
      {!imageLoaded && (
        <View style={[styles.image, styles.placeholder]}>
          <ActivityIndicator />
        </View>
      )}
      <FastImage
        source={{ uri: item.imageUrl }}
        style={styles.image}
        onLoadEnd={() => setImageLoaded(true)}
      />
      <Text>{item.text}</Text>
    </View>
  );
});

// ✅ Or use blurhash/low-quality placeholder
import { Blurhash } from 'react-native-blurhash';

const ListItem = React.memo(({ item }) => {
  const [imageLoaded, setImageLoaded] = useState(false);
  
  return (
    <View style={styles.item}>
      {!imageLoaded && (
        <Blurhash
          blurhash={item.blurhash} // Tiny string from backend
          style={styles.image}
        />
      )}
      <FastImage
        source={{ uri: item.imageUrl }}
        style={[styles.image, imageLoaded && styles.imageLoaded]}
        onLoadEnd={() => setImageLoaded(true)}
      />
      <Text>{item.text}</Text>
    </View>
  );
});
```

---

## **Optimization #3: Optimize FlatList Configuration**

### **Step 1: Provide getItemLayout**

```javascript
// ✅ If items have fixed height - HUGE performance boost
const ITEM_HEIGHT = 120;

<FlatList
  data={items}
  renderItem={renderItem}
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
  // Now FlatList can:
  // - Instantly calculate scroll position
  // - Know which items are visible without measuring
  // - Skip expensive layout calculations
/>

// For variable heights, pre-calculate and cache
const itemHeights = useMemo(() => {
  return items.map(item => calculateItemHeight(item));
}, [items]);

<FlatList
  getItemLayout={(data, index) => ({
    length: itemHeights[index],
    offset: itemHeights.slice(0, index).reduce((a, b) => a + b, 0),
    index,
  })}
/>
```

**Performance impact:**
```javascript
// Without getItemLayout:
// Scroll to item 5000:
// 1. Mount items 0-20 (measure)
// 2. Calculate scroll offset
// 3. Mount items 21-40 (measure)
// 4. Repeat... takes seconds

// With getItemLayout:
// Scroll to item 5000:
// 1. Calculate offset: 5000 * 120 = 600,000px
// 2. Mount items 4990-5010 immediately
// Result: Instant
```

### **Step 2: Tune Rendering Windows**

```javascript
// ✅ Optimize rendering parameters
<FlatList
  data={items}
  renderItem={renderItem}
  
  // How many items to render initially
  initialNumToRender={10} // Default: 10
  // Lower = faster initial render, more blank space
  // Higher = slower initial render, less blank space
  
  // How many items to render per batch while scrolling
  maxToRenderPerBatch={5} // Default: 10
  // Lower = less work per frame = smoother scroll
  
  // Number of screens to render above/below viewport
  windowSize={5} // Default: 21 (!)
  // Lower = less memory, more blank space on fast scroll
  // Higher = more memory, less blank space
  
  // Remove off-screen items from native view hierarchy
  removeClippedSubviews={true} // IMPORTANT on Android
  
  // Update frequency during scroll
  updateCellsBatchingPeriod={50} // Default: 50ms
  // Higher = less frequent updates = smoother scroll
  
  // When to start rendering next batch
  onEndReachedThreshold={0.5} // Default: 2
  // Trigger pagination earlier
/>
```

**Recommended configuration for 10,000 items:**

```javascript
<FlatList
  data={items}
  renderItem={renderItem}
  keyExtractor={keyExtractor}
  getItemLayout={getItemLayout} // If possible
  
  // Conservative rendering
  initialNumToRender={8}
  maxToRenderPerBatch={3}
  windowSize={5}
  
  // Memory optimization
  removeClippedSubviews={true}
  
  // Scroll performance
  updateCellsBatchingPeriod={100}
  
  // Progressive loading
  onEndReachedThreshold={0.5}
  onEndReached={loadMoreItems}
/>
```

### **Step 3: Implement Virtualization Properly**

```javascript
// Ensure items are properly recycled
const ListItem = React.memo(({ item }) => {
  // ❌ Don't store state in list items
  // const [expanded, setExpanded] = useState(false);
  // This breaks virtualization - state lost when recycled
  
  // ✅ Store state in parent or global state
  const isExpanded = expandedItems.has(item.id);
  
  return (
    <View style={styles.item}>
      <FastImage source={{ uri: item.imageUrl }} style={styles.image} />
      <Text>{item.text}</Text>
      {isExpanded && <Text>{item.details}</Text>}
    </View>
  );
});
```

---

## **Optimization #4: Reduce Bridge Traffic**

```javascript
// ✅ Use useNativeDriver where possible
const ListItem = React.memo(({ item }) => {
  const opacity = useRef(new Animated.Value(0)).current;
  
  useEffect(() => {
    Animated.timing(opacity, {
      toValue: 1,
      duration: 200,
      useNativeDriver: true, // Keeps animation off JS thread
    }).start();
  }, []);
  
  return (
    <Animated.View style={[styles.item, { opacity }]}>
      <FastImage source={{ uri: item.imageUrl }} style={styles.image} />
      <Text>{item.text}</Text>
    </Animated.View>
  );
});

// ✅ Batch state updates
const [visibleRange, setVisibleRange] = useState({ start: 0, end: 20 });

const onViewableItemsChanged = useRef(({ viewableItems }) => {
  // Batch multiple changes
  const start = viewableItems[0]?.index ?? 0;
  const end = viewableItems[viewableItems.length - 1]?.index ?? 20;
  
  setVisibleRange({ start, end }); // Single state update
}).current;

<FlatList
  onViewableItemsChanged={onViewableItemsChanged}
  viewabilityConfig={{ itemVisiblePercentThreshold: 50 }}
/>
```

---

## **Optimization #5: Memory Management**

### **Monitor Memory Usage**

```javascript
// Check memory during scroll
import { PerformanceObserver } from 'react-native-performance';

const observer = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`Memory: ${entry.memory.usedJSHeapSize / 1048576}MB`);
  });
});

observer.observe({ entryTypes: ['measure'] });
```

### **Implement Pagination**

```javascript
// ✅ Don't load all 10,000 items at once
const [items, setItems] = useState([]);
const [page, setPage] = useState(1);
const pageSize = 50;

const loadMoreItems = useCallback(() => {
  const start = page * pageSize;
  const end = start + pageSize;
  const newItems = allItems.slice(start, end);
  
  setItems(prev => [...prev, ...newItems]);
  setPage(p => p + 1);
}, [page]);

<FlatList
  data={items}
  renderItem={renderItem}
  onEndReached={loadMoreItems}
  onEndReachedThreshold={0.5}
  ListFooterComponent={<ActivityIndicator />}
/>
```

### **Clear Cache Periodically**

```javascript
// Clear image cache when memory pressure
useEffect(() => {
  const handleMemoryWarning = () => {
    console.warn('Memory warning - clearing cache');
    FastImage.clearMemoryCache();
  };
  
  // Listen for memory warnings (iOS)
  // On Android, system handles this automatically
  const subscription = DeviceEventEmitter.addListener(
    'memoryWarning',
    handleMemoryWarning
  );
  
  return () => subscription.remove();
}, []);
```

---

## **Optimization #6: Use RecyclerListView (Advanced)**

For truly massive lists, consider RecyclerListView:

```javascript
// ✅ RecyclerListView: Better than FlatList for huge lists
import { RecyclerListView, DataProvider, LayoutProvider } from 'recyclerlistview';

const dataProvider = new DataProvider((r1, r2) => r1.id !== r2.id);

const layoutProvider = new LayoutProvider(
  (index) => 'ITEM', // Type of item
  (type, dim) => {
    dim.width = Dimensions.get('window').width;
    dim.height = 120; // Fixed height
  }
);

const rowRenderer = (type, item) => {
  return (
    <ListItem item={item} />
  );
};

<RecyclerListView
  dataProvider={dataProvider.cloneWithRows(items)}
  layoutProvider={layoutProvider}
  rowRenderer={rowRenderer}
  // RecyclerListView recycles views more aggressively
  // Better performance for 10,000+ items
/>
```

**Why RecyclerListView is faster:**
- True recycling (reuses unmounted views)
- More efficient memory management
- Better scroll performance
- But: Steeper learning curve, less flexible

---

## **Complete Optimized Implementation**

Here's the final, production-ready code:

```javascript
import React, { useCallback, useMemo, useRef, useState } from 'react';
import { FlatList, View, Text, StyleSheet, Dimensions } from 'react-native';
import FastImage from 'react-native-fast-image';

// Constants
const ITEM_HEIGHT = 120;
const SCREEN_WIDTH = Dimensions.get('window').width;

// Styles outside component (created once)
const styles = StyleSheet.create({
  item: {
    height: ITEM_HEIGHT,
    flexDirection: 'row',
    padding: 12,
    backgroundColor: '#fff',
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  image: {
    width: 96,
    height: 96,
    borderRadius: 8,
    backgroundColor: '#f0f0f0',
  },
  content: {
    flex: 1,
    marginLeft: 12,
    justifyContent: 'center',
  },
  title: {
    fontSize: 16,
    fontWeight: '600',
    marginBottom: 4,
  },
  subtitle: {
    fontSize: 14,
    color: '#666',
  },
});

// Memoized item component
const ListItem = React.memo(({ item }) => {
  // Optimize image URL for size
  const imageUrl = useMemo(
    () => `${item.imageUrl}?w=200&h=200&q=80`,
    [item.imageUrl]
  );
  
  return (
    <View style={styles.item}>
      <FastImage
        source={{
          uri: imageUrl,
          priority: FastImage.priority.normal,
          cache: FastImage.cacheControl.immutable,
        }}
        style={styles.image}
        resizeMode={FastImage.resizeMode.cover}
      />
      <View style={styles.content}>
        <Text style={styles.title} numberOfLines={2}>
          {item.title}
        </Text>
        <Text style={styles.subtitle} numberOfLines={1}>
          {item.subtitle}
        </Text>
      </View>
    </View>
  );
}, (prevProps, nextProps) => {
  // Custom comparison - only re-render if data actually changed
  return (
    prevProps.item.id === nextProps.item.id &&
    prevProps.item.imageUrl === nextProps.item.imageUrl &&
    prevProps.item.title === nextProps.item.title &&
    prevProps.item.subtitle === nextProps.item.subtitle
  );
});

// Main list component
const OptimizedList = ({ data }) => {
  // Stable callbacks
  const keyExtractor = useCallback((item) => item.id, []);
  
  const getItemLayout = useCallback(
    (data, index) => ({
      length: ITEM_HEIGHT,
      offset: ITEM_HEIGHT * index,
      index,
    }),
    []
  );
  
  const renderItem = useCallback(({ item }) => {
    return <ListItem item={item} />;
  }, []);
  
  // Viewability tracking
  const viewabilityConfig = useRef({
    itemVisiblePercentThreshold: 50,
    minimumViewTime: 500,
  }).current;
  
  const onViewableItemsChanged = useRef(({ viewableItems }) => {
    // Track which items are visible (for analytics, etc.)
    console.log('Visible items:', viewableItems.length);
  }).current;
  
  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      getItemLayout={getItemLayout}
      
      // Performance optimizations
      initialNumToRender={8}
      maxToRenderPerBatch={3}
      windowSize={5}
      removeClippedSubviews={true}
      updateCellsBatchingPeriod={100}
      
      // Viewability
      onViewableItemsChanged={onViewableItemsChanged}
      viewabilityConfig={viewabilityConfig}
      
      // Scroll performance
      scrollEventThrottle={16}
      
      // Memory management
      onEndReachedThreshold={0.5}
    />
  );
};

export default OptimizedList;
```

---

## **Measurement & Validation**

### **Before Optimization:**
```
Initial render: 2000ms
Scroll FPS: 30-40
Memory: 250MB
JS thread: 80% utilization
UI thread: 70% utilization
```

### **After Optimization:**
```
Initial render: 300ms (6.6x faster)
Scroll FPS: 58-60 (smooth)
Memory: 80MB (3x less)
JS thread: 30% utilization
UI thread: 20% utilization
```

---

## **Debugging Tools Summary**

```javascript
// 1. Performance Monitor (Dev Menu)
// Shows real-time JS/UI FPS

// 2. React DevTools Profiler
import { Profiler } from 'react';
// Measure component render times

// 3. Flipper
// - React DevTools
// - Layout Inspector
// - Network Inspector
// - Performance Plugin

// 4. Systrace (Android) / Instruments (iOS)
// Low-level native performance profiling

// 5. Why Did You Render
import whyDidYouRender from '@welldone-software/why-did-you-render';
// Detect unnecessary re-renders

// 6. Custom logging
console.time('ListItem render');
// ... rendering code
console.timeEnd('ListItem render');
```

---

## **Interview Answer Template**

*"For a slow FlatList with 10,000 items, I'd follow this systematic approach:*

*First, I'd **measure and identify the bottleneck** using React Native's Performance Monitor to check JS and UI thread FPS, and React DevTools Profiler to see render times.*

*Most commonly, the issues are:*
*1. **Excessive re-renders** - I'd memoize list items with React.memo, use StyleSheet.create instead of inline styles, and implement stable keyExtractor*
*2. **Image loading** - I'd use appropriately sized images from a CDN, implement react-native-fast-image for better caching, and add progressive loading*
*3. **Layout calculations** - I'd provide getItemLayout if items have fixed heights to skip expensive measurements*
*4. **Rendering too many items** - I'd tune initialNumToRender, maxToRenderPerBatch, and windowSize to render fewer items*

*I'd also implement pagination to load items progressively rather than all 10,000 upfront, enable removeClippedSubviews on Android, and monitor memory usage.*

*After each optimization, I'd measure again to validate the improvement. For extremely large lists beyond 10,000 items, I might consider RecyclerListView for its superior recycling mechanism.*

*The key is systematic measurement, targeted fixes, and validation - not random optimization attempts."*

---

**Would you like to:**
- Dive deeper into any specific optimization?
- Discuss RecyclerListView vs FlatList in detail?
- Explore image optimization strategies further?
- Move on to another React Native topic?
