# Question 3: A screen in your app takes 5 seconds to become interactive after navigation. What could be causing this and how would you fix it?

## **Interview Approach:**

*"5 seconds is way too long. I'd profile the screen initialization to identify where time is spent, then optimize the critical path. Let me break this down."*

---

## **Phase 1: Measure Where Time Is Spent**

```javascript
// Add performance markers
import { Performance } from 'react-native-performance';

const SlowScreen = ({ navigation, route }) => {
  useEffect(() => {
    Performance.mark('screen-start');
  }, []);
  
  // ... component code
  
  useEffect(() => {
    Performance.mark('screen-interactive');
    Performance.measure('screen-load', 'screen-start', 'screen-interactive');
    
    const measures = Performance.getEntriesByType('measure');
    console.log('Screen load time:', measures[measures.length - 1].duration, 'ms');
  }, []);
  
  return <View>...</View>;
};

// Break down timing
const DetailedProfiling = () => {
  useEffect(() => {
    console.time('Total load time');
    
    console.time('1. Data fetch');
    fetchData().then(() => {
      console.timeEnd('1. Data fetch'); // e.g., 2000ms
      
      console.time('2. Process data');
      const processed = processData(data);
      console.timeEnd('2. Process data'); // e.g., 1500ms
      
      console.time('3. Render');
      setProcessedData(processed);
      requestAnimationFrame(() => {
        console.timeEnd('3. Render'); // e.g., 1500ms
        console.timeEnd('Total load time'); // 5000ms
      });
    });
  }, []);
};
```

---

## **Common Bottlenecks and Fixes**

### **Bottleneck #1: Synchronous Data Processing**

```javascript
// ❌ PROBLEM: Heavy computation blocks render
const SlowScreen = () => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    const rawData = fetchDataSync(); // 500ms
    
    // Heavy processing on main thread!
    const processed = rawData.map(item => ({
      ...item,
      computed: expensiveCalculation(item), // 100ms × 50 items = 5000ms!
      formatted: complexFormatting(item),
    }));
    
    setData(processed);
    // Screen frozen for 5 seconds!
  }, []);
  
  if (!data) return <LoadingSpinner />;
  return <FlatList data={data} />;
};

// ✅ FIX 1: Progressive rendering
const FixedScreen = () => {
  const [data, setData] = useState([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    let cancelled = false;
    
    const loadIncrementally = async () => {
      const rawData = await fetchData();
      
      // Show basic data immediately
      if (!cancelled) {
        setData(rawData);
        setLoading(false);
      }
      
      // Process in background
      const processed = await processInBackground(rawData);
      
      if (!cancelled) {
        setData(processed);
      }
    };
    
    loadIncrementally();
    
    return () => {
      cancelled = true;
    };
  }, []);
  
  // Screen interactive immediately, data enhances progressively
  return <FlatList data={data} />;
};

// ✅ FIX 2: Use InteractionManager
import { InteractionManager } from 'react-native';

const BetterFixedScreen = () => {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    // Load basic data first
    fetchBasicData().then(setData);
    
    // Defer heavy processing until animations complete
    InteractionManager.runAfterInteractions(() => {
      processHeavyData().then(processed => {
        setData(processed);
      });
    });
  }, []);
  
  return <FlatList data={data} />;
};

// ✅ FIX 3: Web Workers (if processing is REALLY heavy)
// Note: React Native doesn't have Web Workers, but can use:
// - Separate thread via Native Module
// - Or react-native-workers library

import { Worker } from 'react-native-workers';

const WorkerBasedScreen = () => {
  const [data, setData] = useState([]);
  
  useEffect(() => {
    const worker = new Worker('data-processor.js');
    
    worker.postMessage({ data: rawData });
    
    worker.onmessage = (e) => {
      setData(e.data.processed);
    };
    
    return () => worker.terminate();
  }, []);
  
  return <FlatList data={data} />;
};
```

### **Bottleneck #2: Multiple Sequential API Calls**

```javascript
// ❌ PROBLEM: Sequential fetches (waterfall)
const SlowScreen = () => {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  
  useEffect(() => {
    const loadData = async () => {
      const userData = await fetchUser();      // 1000ms
      setUser(userData);
      
      const userPosts = await fetchPosts(userData.id);  // 1500ms
      setPosts(userPosts);
      
      const postComments = await fetchComments(userPosts[0].id); // 1000ms
      setComments(postComments);
      
      // Total: 3500ms + render time = 4-5 seconds
    };
    
    loadData();
  }, []);
  
  return <View>{/* ... */}</View>;
};

// ✅ FIX 1: Parallel requests
const FixedScreen = () => {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  
  useEffect(() => {
    const loadData = async () => {
      // Fetch in parallel!
      const [userData, userPosts, latestComments] = await Promise.all([
        fetchUser(),
        fetchPosts(),
        fetchComments(),
      ]);
      
      setUser(userData);
      setPosts(userPosts);
      setComments(latestComments);
      
      // Total: max(1000ms, 1500ms, 1000ms) = 1500ms
    };
    
    loadData();
  }, []);
  
  return <View>{/* ... */}</View>;
};

// ✅ FIX 2: Show data as it arrives
const ProgressiveLoadScreen = () => {
  const [user, setUser] = useState(null);
  const [posts, setPosts] = useState([]);
  const [comments, setComments] = useState([]);
  
  useEffect(() => {
    // Start all requests immediately
    fetchUser().then(setUser);       // Renders when ready
    fetchPosts().then(setPosts);     // Renders when ready
    fetchComments().then(setComments); // Renders when ready
    
    // Screen shows data progressively as each request completes
  }, []);
  
  return (
    <View>
      {user && <UserHeader user={user} />}
      {posts.length > 0 && <PostsList posts={posts} />}
      {comments.length > 0 && <CommentsList comments={comments} />}
    </View>
  );
};

// ✅ FIX 3: GraphQL or batched endpoint
const GraphQLScreen = () => {
  const { loading, data } = useQuery(GET_PROFILE_DATA, {
    variables: { userId: route.params.userId },
  });
  
  // Single request fetches user + posts + comments
  // Backend optimizes queries
  // Faster than 3 separate REST calls
  
  if (loading) return <LoadingSpinner />;
  
  return (
    <View>
      <UserHeader user={data.user} />
      <PostsList posts={data.posts} />
      <CommentsList comments={data.comments} />
    </View>
  );
};
```

### **Bottleneck #3: Heavy Initial Render**

```javascript
// ❌ PROBLEM: Rendering too much on mount
const SlowScreen = () => {
  const [data] = useState(Array(1000).fill(null).map((_, i) => ({
    id: i,
    title: `Item ${i}`,
    description: 'Long description...',
    image: `https://example.com/image${i}.jpg`,
  })));
  
  return (
    <ScrollView>
      {data.map(item => (
        <ComplexItemComponent key={item.id} item={item} />
        // Renders ALL 1000 items immediately!
        // 1000 × 5ms per component = 5000ms
      ))}
    </ScrollView>
  );
};

// ✅ FIX: Use FlatList (virtualization)
const FixedScreen = () => {
  const [data] = useState(/* ... */);
  
  return (
    <FlatList
      data={data}
      renderItem={({ item }) => <ComplexItemComponent item={item} />}
      // Only renders visible items (maybe 10-15)
      // 15 × 5ms = 75ms
      initialNumToRender={10}
      maxToRenderPerBatch={5}
      windowSize={5}
    />
  );
};
```

### **Bottleneck #4: Navigation Animation + Heavy Load**

```javascript
// ❌ PROBLEM: Heavy work during navigation animation
const SlowScreen = () => {
  useEffect(() => {
    // Starts immediately when screen mounts
    // But navigation animation is still running!
    heavyDataProcessing(); // Blocks animation = janky
  }, []);
  
  return <View>...</View>;
};

// ✅ FIX: Wait for animation to complete
import { InteractionManager } from 'react-native';

const FixedScreen = () => {
  const [ready, setReady] = useState(false);
  
  useEffect(() => {
    // Wait for navigation animation to complete
    const task = InteractionManager.runAfterInteractions(() => {
      setReady(true);
      // Now start heavy work
      loadData();
    });
    
    return () => task.cancel();
  }, []);
  
  if (!ready) {
    return <SkeletonScreen />; // Fast to render
  }
  
  return <ActualScreen />;
};
```

### **Bottleneck #5: Large Bundle Loading**

```javascript
// ❌ PROBLEM: Heavy dependencies loaded eagerly
import moment from 'moment'; // 300KB
import 'moment/locale/es';
import 'moment/locale/fr';
import 'moment/locale/de';
// All locales loaded even if not needed

import _ from 'lodash'; // 70KB
import * as Charts from 'react-native-charts'; // 500KB
import { decode } from 'base-64'; // 10KB

// Heavy screen bundle = slow to load and parse

// ✅ FIX: Lazy load heavy dependencies
const FixedScreen = () => {
  const [ChartComponent, setChartComponent] = useState(null);
  
  useEffect(() => {
    // Load chart library only when needed
    import('react-native-charts').then((module) => {
      setChartComponent(() => module.LineChart);
    });
  }, []);
  
  return (
    <View>
      <BasicContent />
      {ChartComponent ? (
        <ChartComponent data={data} />
      ) : (
        <LoadingChart />
      )}
    </View>
  );
};

// ✅ Better: Use react-native-code-split (if available)
const HeavyChartScreen = loadable(() => import('./HeavyChartScreen'));
```

### **Bottleneck #6: Images Loading Blocking Render**

```javascript
// ❌ PROBLEM: Waiting for images before showing content
const SlowScreen = () => {
  const [imagesLoaded, setImagesLoaded] = useState(false);
  const [loadedCount, setLoadedCount] = useState(0);
  const totalImages = 20;
  
  const handleImageLoad = () => {
    setLoadedCount(prev => {
      const newCount = prev + 1;
      if (newCount === totalImages) {
        setImagesLoaded(true);
      }
      return newCount;
    });
  };
  
  if (!imagesLoaded) {
    return <LoadingSpinner />;
  }
  
  return <View>{/* Images */}</View>;
};

// ✅ FIX: Progressive image loading
const FixedScreen = () => {
  // Show content immediately, images load progressively
  return (
    <ScrollView>
      <Text>Content visible immediately</Text>
      {images.map(img => (
        <FastImage
          key={img.id}
          source={{ uri: img.url }}
          style={styles.image}
          // Each image loads independently
          // Use placeholder or blurhash while loading
        />
      ))}
    </ScrollView>
  );
};
```

---

## **Complete Optimized Implementation**

```javascript
import React, { useEffect, useState, useCallback } from 'react';
import { View, FlatList, InteractionManager } from 'react-native';
import FastImage from 'react-native-fast-image';

const OptimizedScreen = ({ route, navigation }) => {
  const [basicData, setBasicData] = useState(null);
  const [enrichedData, setEnrichedData] = useState(null);
  const [ready, setReady] = useState(false);
  
  // Phase 1: Immediate - Show skeleton
  useEffect(() => {
    // Screen mounts, show skeleton immediately
    console.time('Screen Interactive');
  }, []);
  
  // Phase 2: After animation - Load critical data
  useEffect(() => {
    const task = InteractionManager.runAfterInteractions(async () => {
      try {
        // Parallel fetch of critical data
        const [user, posts] = await Promise.all([
          fetchUser(route.params.userId),
          fetchPosts(route.params.userId),
        ]);
        
        setBasicData({ user, posts });
        setReady(true);
        
        console.timeEnd('Screen Interactive'); // ~500ms
        
        // Phase 3: Background - Enhance with additional data
        fetchCommentsInBackground(posts).then(comments => {
          setEnrichedData({ ...basicData, comments });
        });
        
      } catch (error) {
        console.error(error);
        setReady(true); // Show error state
      }
    });
    
    return () => task.cancel();
  }, [route.params.userId]);
  
  // Skeleton while loading
  if (!ready || !basicData) {
    return <SkeletonScreen />;
  }
  
  return (
    <FlatList
      data={basicData.posts}
      ListHeaderComponent={<UserHeader user={basicData.user} />}
      renderItem={({ item }) => (
        <PostItem 
          post={item}
          comments={enrichedData?.comments[item.id]}
        />
      )}
      initialNumToRender={5}
      maxToRenderPerBatch={3}
      windowSize={5}
      removeClippedSubviews={true}
    />
  );
};

// Lightweight skeleton (renders in <50ms)
const SkeletonScreen = () => (
  <View style={styles.skeleton}>
    <View style={styles.skeletonHeader} />
    <View style={styles.skeletonItem} />
    <View style={styles.skeletonItem} />
    <View style={styles.skeletonItem} />
  </View>
);

export default OptimizedScreen;
```

---

### **Results:**

```
Before:
- 0-5000ms: Loading spinner (blocking)
- 5000ms: Content appears
- TTI: 5000ms

After:
- 0ms: Skeleton appears (instant)
- 300ms: Animation completes
- 500ms: Basic content visible
- 1000ms: Enhanced content loads in background
- TTI: 500ms (10x improvement!)
```

---
