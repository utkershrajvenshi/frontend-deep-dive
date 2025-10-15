# Question 4: You're building a real-time chat application. Messages arrive frequently, and the FlatList re-renders are causing frame drops. How do you optimize this?

## **Interview Approach:**

*"Chat apps are uniquely challenging because they combine frequent data updates, complex UI, scrolling performance, and real-time synchronization. I'd approach this systematically by understanding the bottlenecks, then applying targeted optimizations. Let me walk through my complete strategy."*

---

## **Phase 1: Understand the Problem**

### **Profile the Bottleneck**

```javascript
// First, measure where the performance issue is
import { Profiler } from 'react';

const ChatScreen = () => {
  const onRenderCallback = (id, phase, actualDuration, baseDuration) => {
    if (actualDuration > 16.67) { // 60 FPS threshold
      console.warn(`[PERF] ${id} took ${actualDuration}ms (dropped frames!)`);
      console.log('Phase:', phase); // mount or update
    }
  };
  
  return (
    <Profiler id="ChatList" onRender={onRenderCallback}>
      <FlatList data={messages} renderItem={renderItem} />
    </Profiler>
  );
};

// Enable FPS monitor in dev menu
// Look for:
// 1. JS thread FPS dropping below 60
// 2. UI thread FPS dropping below 60
// 3. Correlation with message arrivals
```

**Common bottleneck patterns:**

```javascript
// Problem Pattern 1: Entire list re-renders on each new message
// Symptom: JS FPS drops to 20-30, all items re-mount

// Problem Pattern 2: Scroll position jumps
// Symptom: User scrolls up, new message arrives, jumps back to bottom

// Problem Pattern 3: Input lag
// Symptom: Typing feels sluggish when messages are arriving

// Problem Pattern 4: Memory growth
// Symptom: App slows down over time, eventually crashes
```

---

## **Phase 2: Core Optimizations**

### **Optimization #1: Aggressive Memoization**

```javascript
// ❌ PROBLEM: Every message re-renders on every update
const BadMessageItem = ({ message, onPress }) => {
  console.log('Rendering message:', message.id); // Logs 1000 times per new message!
  
  return (
    <View style={{ padding: 10 }}>
      <Text>{message.user}: {message.text}</Text>
      <Text>{new Date(message.timestamp).toLocaleString()}</Text>
    </View>
  );
};

// ✅ SOLUTION: Deep memoization
const ChatMessage = React.memo(({ 
  message, 
  currentUserId, 
  onPress,
  onLongPress,
  showTimestamp 
}) => {
  // Memoize computed values
  const isOwnMessage = useMemo(
    () => message.userId === currentUserId,
    [message.userId, currentUserId]
  );
  
  const formattedTime = useMemo(
    () => formatTimestamp(message.timestamp),
    [message.timestamp]
  );
  
  const messageStyle = useMemo(
    () => [
      styles.messageBubble,
      isOwnMessage && styles.ownMessage,
    ],
    [isOwnMessage]
  );
  
  return (
    <TouchableOpacity 
      onPress={onPress}
      onLongPress={onLongPress}
      activeOpacity={0.7}
    >
      <View style={styles.messageContainer}>
        {!isOwnMessage && (
          <Text style={styles.username}>{message.username}</Text>
        )}
        <View style={messageStyle}>
          <Text style={styles.messageText}>{message.text}</Text>
          {showTimestamp && (
            <Text style={styles.timestamp}>{formattedTime}</Text>
          )}
        </View>
      </View>
    </TouchableOpacity>
  );
}, (prevProps, nextProps) => {
  // Custom comparison - critical for performance
  // Only re-render if actual content changed
  return (
    prevProps.message.id === nextProps.message.id &&
    prevProps.message.text === nextProps.message.text &&
    prevProps.message.status === nextProps.message.status && // e.g., sent, delivered, read
    prevProps.currentUserId === nextProps.currentUserId &&
    prevProps.showTimestamp === nextProps.showTimestamp
    // Note: We don't compare onPress/onLongPress if they're stable
  );
});

// Define styles outside component (never recreated)
const styles = StyleSheet.create({
  messageContainer: {
    paddingHorizontal: 16,
    paddingVertical: 4,
  },
  messageBubble: {
    backgroundColor: '#E5E5EA',
    borderRadius: 18,
    paddingHorizontal: 14,
    paddingVertical: 8,
    maxWidth: '75%',
  },
  ownMessage: {
    backgroundColor: '#007AFF',
    alignSelf: 'flex-end',
  },
  messageText: {
    fontSize: 16,
    lineHeight: 20,
  },
  username: {
    fontSize: 12,
    color: '#8E8E93',
    marginBottom: 2,
  },
  timestamp: {
    fontSize: 11,
    color: '#8E8E93',
    marginTop: 4,
  },
});

// Helper function (pure, can be memoized by engine)
const formatTimestamp = (timestamp) => {
  const date = new Date(timestamp);
  const now = new Date();
  const diffMs = now - date;
  
  if (diffMs < 60000) return 'Just now';
  if (diffMs < 3600000) return `${Math.floor(diffMs / 60000)}m ago`;
  if (diffMs < 86400000) return `${Math.floor(diffMs / 3600000)}h ago`;
  
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
};
```

---

### **Optimization #2: Batch Updates**

```javascript
// ❌ PROBLEM: Each message triggers a separate render
const BadChatScreen = () => {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    socket.on('message', (msg) => {
      setMessages(prev => [...prev, msg]);
      // 10 messages/sec = 10 state updates/sec = 10 renders/sec
    });
  }, []);
};

// ✅ SOLUTION: Batch multiple messages into single update
const OptimizedChatScreen = () => {
  const [messages, setMessages] = useState([]);
  
  // Buffer for incoming messages
  const messageBuffer = useRef([]);
  const flushTimeoutRef = useRef(null);
  const isFlushingRef = useRef(false);
  
  // Flush buffered messages
  const flushMessages = useCallback(() => {
    if (isFlushingRef.current || messageBuffer.current.length === 0) {
      return;
    }
    
    isFlushingRef.current = true;
    
    // Use functional update to avoid stale closures
    setMessages(prevMessages => {
      const newMessages = [...prevMessages, ...messageBuffer.current];
      messageBuffer.current = [];
      return newMessages;
    });
    
    // Schedule next flush check after paint
    requestAnimationFrame(() => {
      isFlushingRef.current = false;
    });
  }, []);
  
  useEffect(() => {
    const socket = connectToChat();
    
    socket.on('message', (newMessage) => {
      // Add to buffer
      messageBuffer.current.push(newMessage);
      
      // Clear existing timeout
      if (flushTimeoutRef.current) {
        clearTimeout(flushTimeoutRef.current);
      }
      
      // Strategy 1: Flush after short delay (coalesce rapid messages)
      flushTimeoutRef.current = setTimeout(flushMessages, 50);
      
      // Strategy 2: Flush immediately if buffer is large
      if (messageBuffer.current.length >= 10) {
        clearTimeout(flushTimeoutRef.current);
        flushMessages();
      }
    });
    
    return () => {
      socket.disconnect();
      if (flushTimeoutRef.current) {
        clearTimeout(flushTimeoutRef.current);
      }
      flushMessages(); // Flush remaining messages
    };
  }, [flushMessages]);
  
  return <FlatList data={messages} renderItem={renderItem} />;
};

// Performance impact:
// Before: 10 messages in 1 second = 10 renders
// After: 10 messages in 1 second = 1-2 renders (batched)
// Result: 5-10x fewer renders
```

---

### **Optimization #3: Intelligent Scrolling**

```javascript
// ✅ Smart auto-scroll behavior
const SmartScrollingChat = () => {
  const flatListRef = useRef(null);
  const [messages, setMessages] = useState([]);
  const [isNearBottom, setIsNearBottom] = useState(true);
  const [unreadCount, setUnreadCount] = useState(0);
  
  // Track if user is near bottom
  const handleScroll = useCallback((event) => {
    const { contentOffset, contentSize, layoutMeasurement } = event.nativeEvent;
    
    // Consider "near bottom" if within 100px
    const distanceFromBottom = 
      contentSize.height - contentOffset.y - layoutMeasurement.height;
    
    const nearBottom = distanceFromBottom < 100;
    
    setIsNearBottom(nearBottom);
    
    // Reset unread count if user scrolls to bottom
    if (nearBottom) {
      setUnreadCount(0);
    }
  }, []);
  
  // Add new message
  const addMessage = useCallback((newMessage) => {
    setMessages(prev => [...prev, newMessage]);
    
    if (isNearBottom) {
      // User is at bottom - auto-scroll
      setTimeout(() => {
        flatListRef.current?.scrollToEnd({ animated: true });
      }, 100);
    } else {
      // User scrolled up - don't interrupt, show badge
      setUnreadCount(count => count + 1);
    }
  }, [isNearBottom]);
  
  // Scroll to bottom on button press
  const scrollToBottom = useCallback(() => {
    flatListRef.current?.scrollToEnd({ animated: true });
    setUnreadCount(0);
  }, []);
  
  return (
    <View style={styles.container}>
      <FlatList
        ref={flatListRef}
        data={messages}
        renderItem={renderItem}
        onScroll={handleScroll}
        scrollEventThrottle={400} // Don't track too frequently
        keyExtractor={keyExtractor}
        
        // Maintain scroll position when new items added above
        maintainVisibleContentPosition={{
          minIndexForVisible: 0,
          autoscrollToTopThreshold: 100,
        }}
      />
      
      {/* Unread messages badge */}
      {!isNearBottom && unreadCount > 0 && (
        <TouchableOpacity 
          style={styles.unreadBadge}
          onPress={scrollToBottom}
        >
          <Text style={styles.unreadText}>
            ↓ {unreadCount} new message{unreadCount > 1 ? 's' : ''}
          </Text>
        </TouchableOpacity>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  unreadBadge: {
    position: 'absolute',
    bottom: 80,
    alignSelf: 'center',
    backgroundColor: '#007AFF',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
    elevation: 5,
  },
  unreadText: {
    color: '#fff',
    fontSize: 14,
    fontWeight: '600',
  },
});
```

---

### **Optimization #4: Message Limit / Pagination**

```javascript
// ✅ Keep only recent messages in memory
const PaginatedChat = () => {
  const MESSAGE_LIMIT = 100; // Keep last 100 messages
  const [messages, setMessages] = useState([]);
  const [hasOlderMessages, setHasOlderMessages] = useState(true);
  const [isLoadingOlder, setIsLoadingOlder] = useState(false);
  
  const addNewMessage = useCallback((newMessage) => {
    setMessages(prev => {
      const updated = [...prev, newMessage];
      
      // Keep only last MESSAGE_LIMIT messages
      if (updated.length > MESSAGE_LIMIT) {
        return updated.slice(-MESSAGE_LIMIT);
      }
      
      return updated;
    });
  }, []);
  
  // Load older messages when user scrolls to top
  const loadOlderMessages = useCallback(async () => {
    if (isLoadingOlder || !hasOlderMessages) return;
    
    setIsLoadingOlder(true);
    
    try {
      const oldestMessageId = messages[0]?.id;
      const olderMessages = await fetchOlderMessages(oldestMessageId, 50);
      
      if (olderMessages.length === 0) {
        setHasOlderMessages(false);
      } else {
        setMessages(prev => [...olderMessages, ...prev]);
      }
    } catch (error) {
      console.error('Failed to load older messages:', error);
    } finally {
      setIsLoadingOlder(false);
    }
  }, [messages, isLoadingOlder, hasOlderMessages]);
  
  return (
    <FlatList
      data={messages}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
      
      // Load older messages when reaching top
      onEndReached={loadOlderMessages}
      onEndReachedThreshold={0.1}
      inverted={true} // Inverted list (newer at bottom)
      
      // Show loading indicator at top
      ListHeaderComponent={
        isLoadingOlder ? <ActivityIndicator /> : null
      }
    />
  );
};

// Alternative: Virtual windowing (keep all messages, render subset)
const VirtualWindowedChat = () => {
  const [allMessages, setAllMessages] = useState([]); // All messages
  const [visibleRange, setVisibleRange] = useState({ start: 0, end: 50 });
  
  // Only render messages in visible range
  const visibleMessages = useMemo(() => {
    return allMessages.slice(visibleRange.start, visibleRange.end);
  }, [allMessages, visibleRange]);
  
  const onViewableItemsChanged = useRef(({ viewableItems }) => {
    if (viewableItems.length > 0) {
      const start = Math.max(0, viewableItems[0].index - 25);
      const end = Math.min(
        allMessages.length,
        viewableItems[viewableItems.length - 1].index + 25
      );
      setVisibleRange({ start, end });
    }
  }).current;
  
  return (
    <FlatList
      data={visibleMessages}
      renderItem={renderItem}
      onViewableItemsChanged={onViewableItemsChanged}
      viewabilityConfig={{
        itemVisiblePercentThreshold: 50,
      }}
    />
  );
};
```

---

### **Optimization #5: Stable Callbacks**

```javascript
// ❌ PROBLEM: New callback functions on every render
const BadChat = () => {
  const [messages, setMessages] = useState([]);
  
  return (
    <FlatList
      data={messages}
      renderItem={({ item }) => (
        <ChatMessage
          message={item}
          onPress={() => handlePress(item.id)} // New function every render!
          onLongPress={() => handleLongPress(item.id)} // New function!
        />
      )}
    />
  );
};

// ✅ SOLUTION: Stable callbacks with useCallback
const OptimizedChat = () => {
  const [messages, setMessages] = useState([]);
  const [selectedMessage, setSelectedMessage] = useState(null);
  
  // Stable callback - created once
  const handleMessagePress = useCallback((messageId) => {
    console.log('Pressed message:', messageId);
    // Handle message press
  }, []);
  
  const handleMessageLongPress = useCallback((messageId) => {
    setSelectedMessage(messageId);
    // Show action sheet
  }, []);
  
  // Stable render function
  const renderItem = useCallback(({ item }) => (
    <ChatMessage
      message={item}
      onPress={handleMessagePress}
      onLongPress={handleMessageLongPress}
    />
  ), [handleMessagePress, handleMessageLongPress]);
  
  // Stable key extractor
  const keyExtractor = useCallback((item) => item.id, []);
  
  return (
    <FlatList
      data={messages}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
    />
  );
};

// Even better: Pass message ID, handle in context
const ChatMessageWithContext = React.memo(({ message }) => {
  const { handlePress, handleLongPress } = useMessageActions();
  
  return (
    <TouchableOpacity
      onPress={() => handlePress(message.id)}
      onLongPress={() => handleLongPress(message.id)}
    >
      <View>{/* message UI */}</View>
    </TouchableOpacity>
  );
});
```

---

### **Optimization #6: getItemLayout**

```javascript
// ✅ Provide getItemLayout for huge performance boost
const AVERAGE_MESSAGE_HEIGHT = 60;

const FixedHeightChat = () => {
  const getItemLayout = useCallback((data, index) => ({
    length: AVERAGE_MESSAGE_HEIGHT,
    offset: AVERAGE_MESSAGE_HEIGHT * index,
    index,
  }), []);
  
  return (
    <FlatList
      data={messages}
      renderItem={renderItem}
      getItemLayout={getItemLayout}
      // FlatList can now:
      // - Calculate scroll position instantly
      // - Know visible items without measuring
      // - Skip layout calculations
    />
  );
};

// For variable height messages (more complex):
const VariableHeightChat = () => {
  // Pre-calculate or estimate heights
  const messageHeights = useRef(new Map());
  
  const getItemLayout = useCallback((data, index) => {
    // Use cached height or estimate
    const height = messageHeights.current.get(data[index].id) || 60;
    
    // Calculate offset by summing previous heights
    let offset = 0;
    for (let i = 0; i < index; i++) {
      offset += messageHeights.current.get(data[i].id) || 60;
    }
    
    return { length: height, offset, index };
  }, []);
  
  const onLayout = useCallback((event, messageId) => {
    const { height } = event.nativeEvent.layout;
    messageHeights.current.set(messageId, height);
  }, []);
  
  return (
    <FlatList
      data={messages}
      renderItem={({ item }) => (
        <View onLayout={(e) => onLayout(e, item.id)}>
          <ChatMessage message={item} />
        </View>
      )}
      getItemLayout={getItemLayout}
    />
  );
};
```

---

### **Optimization #7: RecyclerListView for Maximum Performance**

```javascript
// ✅ For very high message throughput, use RecyclerListView
import { RecyclerListView, DataProvider, LayoutProvider } from 'recyclerlistview';

const HighThroughputChat = () => {
  const [dataProvider, setDataProvider] = useState(
    new DataProvider((r1, r2) => r1.id !== r2.id)
  );
  
  // Define layout
  const layoutProvider = useMemo(() => new LayoutProvider(
    (index) => {
      const message = dataProvider.getDataForIndex(index);
      return message.type || 'text'; // e.g., 'text', 'image', 'system'
    },
    (type, dim) => {
      switch (type) {
        case 'text':
          dim.width = SCREEN_WIDTH;
          dim.height = 60;
          break;
        case 'image':
          dim.width = SCREEN_WIDTH;
          dim.height = 200;
          break;
        case 'system':
          dim.width = SCREEN_WIDTH;
          dim.height = 40;
          break;
      }
    }
  ), [dataProvider]);
  
  // Add new message
  const addMessage = useCallback((newMessage) => {
    setDataProvider(prevProvider => {
      const currentData = prevProvider.getAllData();
      return prevProvider.cloneWithRows([...currentData, newMessage]);
    });
  }, []);
  
  // Row renderer
  const rowRenderer = useCallback((type, data) => {
    switch (type) {
      case 'text':
        return <TextMessage message={data} />;
      case 'image':
        return <ImageMessage message={data} />;
      case 'system':
        return <SystemMessage message={data} />;
      default:
        return null;
    }
  }, []);
  
  return (
    <RecyclerListView
      dataProvider={dataProvider}
      layoutProvider={layoutProvider}
      rowRenderer={rowRenderer}
      // RecyclerListView aggressively recycles views
      // Much better performance than FlatList for 1000+ messages
    />
  );
};
```

---

### **Optimization #8: Optimize Images/Media**

```javascript
// ✅ Optimize media messages
import FastImage from 'react-native-fast-image';

const MediaMessage = React.memo(({ message }) => {
  const [imageLoaded, setImageLoaded] = useState(false);
  
  // Use thumbnail for preview
  const thumbnailUrl = `${message.imageUrl}?w=400&h=400&q=80`;
  
  return (
    <View style={styles.mediaContainer}>
      {/* Blurhash placeholder */}
      {!imageLoaded && message.blurhash && (
        <Blurhash
          blurhash={message.blurhash}
          style={styles.placeholder}
        />
      )}
      
      <FastImage
        source={{
          uri: thumbnailUrl,
          priority: FastImage.priority.normal,
          cache: FastImage.cacheControl.immutable,
        }}
        style={styles.image}
        onLoadEnd={() => setImageLoaded(true)}
        resizeMode={FastImage.resizeMode.cover}
      />
      
      <Text style={styles.caption}>{message.caption}</Text>
    </View>
  );
});

// Preload images ahead of scroll
const PreloadingChat = () => {
  const onViewableItemsChanged = useRef(({ viewableItems, changed }) => {
    // Preload images just outside viewport
    const visibleIndices = viewableItems.map(item => item.index);
    const maxIndex = Math.max(...visibleIndices);
    
    // Preload next 5 images
    const imagesToPreload = messages
      .slice(maxIndex + 1, maxIndex + 6)
      .filter(msg => msg.type === 'image')
      .map(msg => ({
        uri: `${msg.imageUrl}?w=400&h=400&q=80`,
      }));
    
    FastImage.preload(imagesToPreload);
  }).current;
  
  return (
    <FlatList
      data={messages}
      renderItem={renderItem}
      onViewableItemsChanged={onViewableItemsChanged}
      viewabilityConfig={{
        itemVisiblePercentThreshold: 50,
      }}
    />
  );
};
```

---

### **Optimization #9: Debounce Expensive Operations**

```javascript
// ✅ Debounce typing indicators and read receipts
import { useDebouncedCallback } from 'use-debounce';

const ChatInput = () => {
  const [text, setText] = useState('');
  
  // Debounce typing indicator
  const sendTypingIndicator = useDebouncedCallback(() => {
    socket.emit('typing', { roomId, userId });
  }, 1000);
  
  const handleChangeText = (newText) => {
    setText(newText);
    sendTypingIndicator();
  };
  
  return (
    <TextInput
      value={text}
      onChangeText={handleChangeText}
      placeholder="Type a message..."
    />
  );
};

// Batch read receipts
const ReadReceiptManager = () => {
  const unreadMessages = useRef(new Set());
  const sendReceiptsTimeoutRef = useRef(null);
  
  const markAsRead = useCallback((messageId) => {
    unreadMessages.current.add(messageId);
    
    if (sendReceiptsTimeoutRef.current) {
      clearTimeout(sendReceiptsTimeoutRef.current);
    }
    
    // Batch send after 2 seconds
    sendReceiptsTimeoutRef.current = setTimeout(() => {
      const messageIds = Array.from(unreadMessages.current);
      socket.emit('markAsRead', { messageIds });
      unreadMessages.current.clear();
    }, 2000);
  }, []);
  
  return { markAsRead };
};
```

---

## **Phase 3: Complete Production Implementation**

```javascript
import React, { useEffect, useState, useCallback, useRef, useMemo } from 'react';
import {
  FlatList,
  View,
  Text,
  TextInput,
  TouchableOpacity,
  KeyboardAvoidingView,
  Platform,
  ActivityIndicator,
  StyleSheet,
} from 'react-native';
import FastImage from 'react-native-fast-image';
import { io } from 'socket.io-client';

// ==================== Message Component ====================
const ChatMessage = React.memo(({ 
  message, 
  currentUserId,
  onPress,
  onLongPress 
}) => {
  const isOwnMessage = useMemo(
    () => message.userId === currentUserId,
    [message.userId, currentUserId]
  );
  
  const formattedTime = useMemo(
    () => formatTimestamp(message.timestamp),
    [message.timestamp]
  );
  
  const containerStyle = useMemo(
    () => [
      styles.messageContainer,
      isOwnMessage && styles.ownMessageContainer,
    ],
    [isOwnMessage]
  );
  
  const bubbleStyle = useMemo(
    () => [
      styles.messageBubble,
      isOwnMessage && styles.ownMessageBubble,
    ],
    [isOwnMessage]
  );
  
  const handlePress = useCallback(() => {
    onPress?.(message.id);
  }, [message.id, onPress]);
  
  const handleLongPress = useCallback(() => {
    onLongPress?.(message.id);
  }, [message.id, onLongPress]);
  
  return (
    <TouchableOpacity
      onPress={handlePress}
      onLongPress={handleLongPress}
      activeOpacity={0.7}
    >
      <View style={containerStyle}>
        {!isOwnMessage && (
          <Text style={styles.username}>{message.username}</Text>
        )}
        <View style={bubbleStyle}>
          {message.type === 'image' && (
            <FastImage
              source={{
                uri: message.imageUrl,
                priority: FastImage.priority.normal,
              }}
              style={styles.messageImage}
              resizeMode={FastImage.resizeMode.cover}
            />
          )}
          <Text style={[
            styles.messageText,
            isOwnMessage && styles.ownMessageText,
          ]}>
            {message.text}
          </Text>
          <View style={styles.messageFooter}>
            <Text style={[
              styles.timestamp,
              isOwnMessage && styles.ownTimestamp,
            ]}>
              {formattedTime}
            </Text>
            {isOwnMessage && (
              <MessageStatusIcon status={message.status} />
            )}
          </View>
        </View>
      </View>
    </TouchableOpacity>
  );
}, (prevProps, nextProps) => {
  // Custom comparison for optimal re-render prevention
  return (
    prevProps.message.id === nextProps.message.id &&
    prevProps.message.text === nextProps.message.text &&
    prevProps.message.status === nextProps.message.status &&
    prevProps.message.timestamp === nextProps.message.timestamp &&
    prevProps.currentUserId === nextProps.currentUserId
  );
});

// Message status indicator
const MessageStatusIcon = React.memo(({ status }) => {
  switch (status) {
    case 'sending':
      return <ActivityIndicator size="small" color="#fff" />;
    case 'sent':
      return <Text style={styles.statusIcon}>✓</Text>;
    case 'delivered':
      return <Text style={styles.statusIcon}>✓✓</Text>;
    case 'read':
      return <Text style={[styles.statusIcon, styles.readIcon]}>✓✓</Text>;
    default:
      return null;
  }
});

// ==================== Main Chat Screen ====================
const OptimizedChatScreen = ({ route, navigation }) => {
  const { roomId, currentUserId } = route.params;
  
  // State
  const [messages, setMessages] = useState([]);
  const [inputText, setInputText] = useState('');
  const [isNearBottom, setIsNearBottom] = useState(true);
  const [unreadCount, setUnreadCount] = useState(0);
  const [isLoadingOlder, setIsLoadingOlder] = useState(false);
  const [hasMore, setHasMore] = useState(true);
  
  // Refs
  const flatListRef = useRef(null);
  const socketRef = useRef(null);
  const messageBuffer = useRef([]);
  const flushTimeoutRef = useRef(null);
  const isFlushingRef = useRef(false);
  
  // ==================== Message Batching ====================
  const flushMessages = useCallback(() => {
    if (isFlushingRef.current || messageBuffer.current.length === 0) {
      return;
    }
    
    isFlushingRef.current = true;
    
    setMessages(prevMessages => {
      const newMessages = [...prevMessages, ...messageBuffer.current];
      messageBuffer.current = [];
      
      // Keep only last 200 messages to prevent memory issues
      return newMessages.slice(-200);
    });
    
    // Update unread count if not at bottom
    if (!isNearBottom) {
      setUnreadCount(prev => prev + messageBuffer.current.length);
    }
    
    // Auto-scroll if at bottom
    if (isNearBottom) {
      requestAnimationFrame(() => {
        flatListRef.current?.scrollToEnd({ animated: false });
      });
    }
    
    requestAnimationFrame(() => {
      isFlushingRef.current = false;
    });
  }, [isNearBottom]);
  
  // ==================== Socket Connection ====================
  useEffect(() => {
    socketRef.current = io('wss://chat.example.com', {
      query: { roomId, userId: currentUserId },
      transports: ['websocket'],
    });
    
    // Handle incoming messages
    socketRef.current.on('message', (newMessage) => {
      messageBuffer.current.push(newMessage);
      
      // Clear existing flush timeout
      if (flushTimeoutRef.current) {
        clearTimeout(flushTimeoutRef.current);
      }
      
      // Immediate flush if buffer is large
      if (messageBuffer.current.length >= 10) {
        flushMessages();
      } else {
        // Delayed flush to batch rapid messages
        flushTimeoutRef.current = setTimeout(flushMessages, 50);
      }
    });
    
    // Load initial messages
    socketRef.current.emit('getHistory', { roomId, limit: 50 });
    socketRef.current.on('history', (history) => {
      setMessages(history);
      
      // Scroll to bottom after initial load
      setTimeout(() => {
        flatListRef.current?.scrollToEnd({ animated: false });
      }, 100);
    });
    
    // Handle message status updates
    socketRef.current.on('messageStatus', ({ messageId, status }) => {
      setMessages(prev => 
        prev.map(msg => 
          msg.id === messageId ? { ...msg, status } : msg
        )
      );
    });
    
    return () => {
      if (flushTimeoutRef.current) {
        clearTimeout(flushTimeoutRef.current);
      }
      flushMessages(); // Flush any remaining messages
      socketRef.current?.disconnect();
    };
  }, [roomId, currentUserId, flushMessages]);
  
  // ==================== Load Older Messages ====================
  const loadOlderMessages = useCallback(async () => {
    if (isLoadingOlder || !hasMore) return;
    
    setIsLoadingOlder(true);
    
    try {
      const oldestMessageId = messages[0]?.id;
      
      socketRef.current.emit('getOlderMessages', {
        roomId,
        beforeId: oldestMessageId,
        limit: 50,
      });
      
      socketRef.current.once('olderMessages', (olderMessages) => {
        if (olderMessages.length === 0) {
          setHasMore(false);
        } else {
          setMessages(prev => [...olderMessages, ...prev]);
        }
        setIsLoadingOlder(false);
      });
    } catch (error) {
      console.error('Failed to load older messages:', error);
      setIsLoadingOlder(false);
    }
  }, [roomId, messages, isLoadingOlder, hasMore]);
  
  // ==================== Send Message ====================
  const sendMessage = useCallback(() => {
    if (!inputText.trim()) return;
    
    const tempId = `temp-${Date.now()}`;
    const newMessage = {
      id: tempId,
      text: inputText.trim(),
      userId: currentUserId,
      timestamp: Date.now(),
      status: 'sending',
    };
    
    // Optimistic update
    setMessages(prev => [...prev, newMessage]);
    
    // Clear input
    setInputText('');
    
    // Scroll to bottom
    setTimeout(() => {
      flatListRef.current?.scrollToEnd({ animated: true });
    }, 100);
    
    // Send to server
    socketRef.current.emit('message', {
      tempId,
      text: newMessage.text,
      roomId,
    });
    
    // Server will respond with real message ID
    socketRef.current.once('messageSent', ({ tempId, realId }) => {
      setMessages(prev =>
        prev.map(msg =>
          msg.id === tempId
            ? { ...msg, id: realId, status: 'sent' }
            : msg
        )
      );
    });
  }, [inputText, currentUserId, roomId]);
  
  // ==================== Scroll Handling ====================
  const handleScroll = useCallback((event) => {
    const { contentOffset, contentSize, layoutMeasurement } = event.nativeEvent;
    
    const distanceFromBottom =
      contentSize.height - contentOffset.y - layoutMeasurement.height;
    
    const nearBottom = distanceFromBottom < 100;
    
    setIsNearBottom(nearBottom);
    
    if (nearBottom) {
      setUnreadCount(0);
    }
  }, []);
  
  const scrollToBottom = useCallback(() => {
    flatListRef.current?.scrollToEnd({ animated: true });
    setUnreadCount(0);
  }, []);
  
  // ==================== Callbacks ====================
  const keyExtractor = useCallback((item) => item.id, []);
  
  const renderItem = useCallback(({ item }) => (
    <ChatMessage
      message={item}
      currentUserId={currentUserId}
    />
  ), [currentUserId]);
  
  const getItemLayout = useCallback((data, index) => ({
    length: 80, // Average message height
    offset: 80 * index,
    index,
  }), []);
  
  // ==================== Render ====================
  return (
    <KeyboardAvoidingView
      style={styles.container}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
      keyboardVerticalOffset={Platform.OS === 'ios' ? 90 : 0}
    >
      <FlatList
        ref={flatListRef}
        data={messages}
        renderItem={renderItem}
        keyExtractor={keyExtractor}
        getItemLayout={getItemLayout}
        
        // Performance optimizations
        initialNumToRender={20}
        maxToRenderPerBatch={10}
        windowSize={5}
        removeClippedSubviews={true}
        
        // Scroll optimizations
        onScroll={handleScroll}
        scrollEventThrottle={400}
        
        // Load older messages
        onEndReached={loadOlderMessages}
        onEndReachedThreshold={0.1}
        
        // Loading indicator
        ListHeaderComponent={
          isLoadingOlder ? (
            <View style={styles.loadingContainer}>
              <ActivityIndicator />
            </View>
          ) : null
        }
        
        // Empty state
        ListEmptyComponent={
          <View style={styles.emptyContainer}>
            <Text style={styles.emptyText}>
              No messages yet. Start the conversation!
            </Text>
          </View>
        }
      />
      
      {/* Unread messages badge */}
      {!isNearBottom && unreadCount > 0 && (
        <TouchableOpacity
          style={styles.unreadBadge}
          onPress={scrollToBottom}
          activeOpacity={0.8}
        >
          <Text style={styles.unreadText}>
            ↓ {unreadCount} new message{unreadCount > 1 ? 's' : ''}
          </Text>
        </TouchableOpacity>
      )}
      
      {/* Input */}
      <View style={styles.inputContainer}>
        <TextInput
          style={styles.input}
          value={inputText}
          onChangeText={setInputText}
          placeholder="Type a message..."
          placeholderTextColor="#999"
          multiline
          maxLength={1000}
        />
        <TouchableOpacity
          style={[
            styles.sendButton,
            !inputText.trim() && styles.sendButtonDisabled,
          ]}
          onPress={sendMessage}
          disabled={!inputText.trim()}
        >
          <Text style={styles.sendButtonText}>Send</Text>
        </TouchableOpacity>
      </View>
    </KeyboardAvoidingView>
  );
};

// ==================== Helper Functions ====================
const formatTimestamp = (timestamp) => {
  const date = new Date(timestamp);
  const now = new Date();
  const diffMs = now - date;
  
  if (diffMs < 60000) return 'Just now';
  if (diffMs < 3600000) return `${Math.floor(diffMs / 60000)}m`;
  if (diffMs < 86400000) return `${Math.floor(diffMs / 3600000)}h`;
  
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
};

// ==================== Styles ====================
const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: '#fff',
  },
  messageContainer: {
    paddingHorizontal: 16,
    paddingVertical: 4,
    alignItems: 'flex-start',
  },
  ownMessageContainer: {
    alignItems: 'flex-end',
  },
  username: {
    fontSize: 12,
    color: '#8E8E93',
    marginBottom: 2,
    marginLeft: 4,
  },
  messageBubble: {
    backgroundColor: '#E5E5EA',
    borderRadius: 18,
    paddingHorizontal: 14,
    paddingVertical: 8,
    maxWidth: '75%',
  },
  ownMessageBubble: {
    backgroundColor: '#007AFF',
  },
  messageText: {
    fontSize: 16,
    lineHeight: 20,
    color: '#000',
  },
  ownMessageText: {
    color: '#fff',
  },
  messageImage: {
    width: 200,
    height: 200,
    borderRadius: 12,
    marginBottom: 4,
  },
  messageFooter: {
    flexDirection: 'row',
    alignItems: 'center',
    marginTop: 4,
  },
  timestamp: {
    fontSize: 11,
    color: '#8E8E93',
  },
  ownTimestamp: {
    color: 'rgba(255, 255, 255, 0.7)',
  },
  statusIcon: {
    fontSize: 12,
    color: 'rgba(255, 255, 255, 0.7)',
    marginLeft: 4,
  },
  readIcon: {
    color: '#4CD964',
  },
  unreadBadge: {
    position: 'absolute',
    bottom: 80,
    alignSelf: 'center',
    backgroundColor: '#007AFF',
    paddingHorizontal: 16,
    paddingVertical: 8,
    borderRadius: 20,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 4,
    elevation: 5,
  },
  unreadText: {
    color: '#fff',
    fontSize: 14,
    fontWeight: '600',
  },
  inputContainer: {
    flexDirection: 'row',
    padding: 8,
    borderTopWidth: 1,
    borderTopColor: '#E5E5EA',
    backgroundColor: '#fff',
    alignItems: 'flex-end',
  },
  input: {
    flex: 1,
    backgroundColor: '#F2F2F7',
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 8,
    fontSize: 16,
    maxHeight: 100,
  },
  sendButton: {
    marginLeft: 8,
    backgroundColor: '#007AFF',
    borderRadius: 20,
    paddingHorizontal: 20,
    paddingVertical: 10,
    justifyContent: 'center',
  },
  sendButtonDisabled: {
    backgroundColor: '#C7C7CC',
  },
  sendButtonText: {
    color: '#fff',
    fontSize: 16,
    fontWeight: '600',
  },
  loadingContainer: {
    padding: 20,
    alignItems: 'center',
  },
  emptyContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 20,
  },
  emptyText: {
    fontSize: 16,
    color: '#8E8E93',
    textAlign: 'center',
  },
});

export default OptimizedChatScreen;
```

---

## **Performance Results**

### **Before Optimization:**
```
- 10 messages/second arriving
- 10 renders/second
- FlatList re-rendering all 1000 items
- JS FPS: 20-30
- UI FPS: 30-40
- Noticeable lag when typing
- Scroll jank
- Memory: Growing unbounded
```

### **After Optimization:**
```
- 10 messages/second arriving
- 1-2 renders/second (batched)
- Only visible items re-render (memoized)
- JS FPS: 58-60
- UI FPS: 58-60
- Smooth typing
- Buttery smooth scroll
- Memory: Stable at ~200 messages
```

---

## **Interview Summary**

*"For optimizing real-time chat with frequent updates, my approach focuses on six key areas: aggressive memoization of message components to prevent unnecessary re-renders, batching incoming messages into single state updates to reduce render frequency, intelligent scroll handling that only auto-scrolls when the user is at the bottom, limiting messages in memory to prevent unbounded growth, providing getItemLayout for instant scroll calculations, and using stable callbacks to avoid breaking memoization. For extremely high-throughput scenarios, I'd consider RecyclerListView which recycles views more aggressively than FlatList. The key is measuring first to identify the actual bottleneck, then applying targeted optimizations systematically."*
