## **React Navigation vs React Router: Fundamental Architectural Differences**

### **The Core Philosophical Difference**

**React Router (Web):**
```
URL-driven navigation
The URL is the source of truth
History stack = Browser history API
```

**React Navigation (Mobile):**
```
State-driven navigation
Navigation state is the source of truth
History stack = JavaScript state object
```

This fundamental difference cascades into everything else.

---

## **1. Navigation Paradigm: URL vs State**

### **React Router - URL as Source of Truth**

```javascript
// React Router
import { BrowserRouter, Routes, Route, useNavigate } from 'react-router-dom';

const App = () => (
  <BrowserRouter>
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/profile/:userId" element={<Profile />} />
      <Route path="/settings" element={<Settings />} />
    </Routes>
  </BrowserRouter>
);

const Home = () => {
  const navigate = useNavigate();
  
  const goToProfile = () => {
    navigate('/profile/123'); // Changes URL
    // URL change → React Router detects → Renders Profile
  };
  
  return <button onClick={goToProfile}>Go to Profile</button>;
};

// The URL bar shows: https://myapp.com/profile/123
// Browser history tracks URL changes
// User can bookmark, share URL
// Back button = browser's back button
```

**Key characteristics:**
- URL is serializable and shareable
- Browser manages history
- Refresh maintains state via URL
- Deep linking is native

### **React Navigation - State as Source of Truth**

```javascript
// React Navigation
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

const App = () => (
  <NavigationContainer>
    <Stack.Navigator>
      <Stack.Screen name="Home" component={Home} />
      <Stack.Screen name="Profile" component={Profile} />
      <Stack.Screen name="Settings" component={Settings} />
    </Stack.Navigator>
  </NavigationContainer>
);

const Home = ({ navigation }) => {
  const goToProfile = () => {
    navigation.navigate('Profile', { userId: 123 });
    // Updates navigation state object
    // State change → React Navigation detects → Renders Profile
  };
  
  return <Button title="Go to Profile" onPress={goToProfile} />;
};

// Navigation state (internal JavaScript object):
{
  routes: [
    { name: 'Home', key: 'home-1' },
    { name: 'Profile', key: 'profile-2', params: { userId: 123 } }
  ],
  index: 1, // Currently on Profile
}

// No URL bar in native apps
// React Navigation manages history stack
// Back button = Android hardware button or iOS gesture
```

**Key characteristics:**
- Navigation state is a JavaScript object
- React Navigation manages the stack
- No URLs (native apps don't have URL bars)
- Deep linking requires explicit configuration

---

## **2. History Management: Browser API vs JavaScript State**

### **React Router - Browser History API**

```javascript
// React Router leverages browser's history
import { useNavigate, useLocation } from 'react-router-dom';

const Component = () => {
  const navigate = useNavigate();
  const location = useLocation();
  
  // Browser maintains the stack:
  // [/, /profile, /settings]
  
  navigate('/profile');  // Uses history.pushState()
  navigate(-1);          // Uses history.back()
  navigate('/settings', { replace: true }); // Uses history.replaceState()
  
  // Browser handles:
  // - Back/forward buttons
  // - History persistence across refreshes
  // - Session history
  
  console.log(location.pathname); // '/settings'
  console.log(location.search);   // '?filter=active'
  console.log(location.hash);     // '#section-2'
};

// Browser automatically manages:
window.history.length; // Number of entries
window.history.state;  // Current state
```

**Browser gives you for free:**
- Back/forward buttons work automatically
- History persists across page refreshes
- URL sharing and bookmarking
- Search engine indexing (for web crawlers)

### **React Navigation - JavaScript State Object**

```javascript
// React Navigation manages state internally
import { useNavigation, useRoute, useNavigationState } from '@react-navigation/native';

const Component = () => {
  const navigation = useNavigation();
  const route = useRoute();
  
  // React Navigation maintains state object:
  const navState = useNavigationState(state => state);
  console.log(navState);
  /*
  {
    stale: false,
    type: 'stack',
    key: 'stack-1',
    index: 2,
    routeNames: ['Home', 'Profile', 'Settings'],
    routes: [
      { key: 'home-1', name: 'Home' },
      { key: 'profile-2', name: 'Profile', params: { userId: 123 } },
      { key: 'settings-3', name: 'Settings' }
    ]
  }
  */
  
  navigation.navigate('Profile'); // Updates state.routes, state.index
  navigation.goBack();            // Decrements state.index
  navigation.replace('Settings'); // Replaces current route in state.routes
  
  console.log(route.name);   // 'Settings'
  console.log(route.params); // { userId: 123 }
  console.log(route.key);    // 'settings-3'
};

// React Navigation must manually handle:
// - Android back button (BackHandler API)
// - iOS swipe gesture (pan responders)
// - State persistence (AsyncStorage)
// - Deep linking (URL → state transformation)
```

**React Navigation must implement:**
```javascript
// Android back button handling
import { BackHandler } from 'react-native';

useEffect(() => {
  const backHandler = BackHandler.addEventListener('hardwareBackPress', () => {
    if (navigation.canGoBack()) {
      navigation.goBack();
      return true; // Prevent default (app exit)
    }
    return false; // Allow default (app exit)
  });
  
  return () => backHandler.remove();
}, [navigation]);
```

---

## **3. Navigation Patterns: Routes vs Screens**

### **React Router - Route Matching**

```javascript
// React Router uses declarative route matching
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/users" element={<UserList />} />
  <Route path="/users/:id" element={<UserDetail />} />
  <Route path="/users/:id/edit" element={<UserEdit />} />
  <Route path="/posts/:postId/comments/:commentId" element={<Comment />} />
  <Route path="*" element={<NotFound />} /> {/* Catch-all */}
</Routes>

// Nested routes are flat in URL structure
<Route path="/dashboard" element={<Dashboard />}>
  <Route path="analytics" element={<Analytics />} /> {/* /dashboard/analytics */}
  <Route path="settings" element={<DashSettings />} /> {/* /dashboard/settings */}
</Route>

// Multiple routes can render simultaneously (not in stack)
<div className="app">
  <Sidebar /> {/* Always visible */}
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/profile" element={<Profile />} />
  </Routes>
</div>
```

**Route matching behavior:**
```javascript
// URL: /users/123/edit
// React Router matches: /users/:id/edit
// Renders: <UserEdit />
// Accessible via: useParams() → { id: '123' }

// Can have multiple <Routes> in different components
// Each independently matches against current URL
```

### **React Navigation - Screen Stacks**

```javascript
// React Navigation uses explicit navigator configurations
const Stack = createNativeStackNavigator();
const Tab = createBottomTabNavigator();
const Drawer = createDrawerNavigator();

// Stack Navigator - screens stack on top of each other
<Stack.Navigator>
  <Stack.Screen name="Home" component={Home} />
  <Stack.Screen name="UserList" component={UserList} />
  <Stack.Screen name="UserDetail" component={UserDetail} />
  <Stack.Screen name="UserEdit" component={UserEdit} />
</Stack.Navigator>

// Navigation creates a literal stack:
// When you navigate to UserDetail, Home is still mounted behind it
[Home] → navigate to UserList → [Home, UserList] → [Home, UserList, UserDetail]

// Nested navigators create complex hierarchies
const App = () => (
  <NavigationContainer>
    <Tab.Navigator>
      <Tab.Screen name="HomeTab" component={HomeStack} />
      <Tab.Screen name="ProfileTab" component={ProfileStack} />
    </Tab.Navigator>
  </NavigationContainer>
);

const HomeStack = () => (
  <Stack.Navigator>
    <Stack.Screen name="Home" component={Home} />
    <Stack.Screen name="Details" component={Details} />
  </Stack.Navigator>
);

// Navigation structure:
{
  TabNavigator: {
    index: 0,
    routes: [
      {
        name: 'HomeTab',
        state: {
          StackNavigator: {
            index: 1,
            routes: [
              { name: 'Home' },
              { name: 'Details' } // Current screen
            ]
          }
        }
      },
      { name: 'ProfileTab' }
    ]
  }
}
```

**Screen behavior differences:**
```javascript
// REACT ROUTER: Components unmount when route changes
// Home → Profile: Home unmounts, Profile mounts

// REACT NAVIGATION: Previous screens stay mounted (by default)
// Home → Profile: Home stays mounted (paused), Profile mounts on top
// Useful for: Maintaining scroll position, form state, etc.

// Can configure unmounting:
<Stack.Navigator screenOptions={{ detachPreviousScreen: true }}>
  {/* Previous screens will unmount */}
</Stack.Navigator>
```

---

## **4. Parameter Passing: Query Strings vs Params Object**

### **React Router - URL Parameters and Query Strings**

```javascript
// URL-based parameter passing
import { useParams, useSearchParams, useLocation } from 'react-router-dom';

// Route params (part of path)
const UserProfile = () => {
  const { userId } = useParams(); // From /users/:userId
  console.log(userId); // "123"
  
  return <div>User {userId}</div>;
};

// Query params (after ?)
const SearchResults = () => {
  const [searchParams, setSearchParams] = useSearchParams();
  
  const query = searchParams.get('q');      // ?q=react
  const filter = searchParams.get('filter'); // &filter=active
  const page = searchParams.get('page');     // &page=2
  
  // URL: /search?q=react&filter=active&page=2
  
  // Update query params
  setSearchParams({ q: 'react', filter: 'completed', page: '3' });
  // URL becomes: /search?q=react&filter=completed&page=3
  
  return <div>Search: {query}</div>;
};

// Location state (not in URL, passed during navigation)
const Component = () => {
  const navigate = useNavigate();
  const location = useLocation();
  
  // Navigate with state
  navigate('/profile', { 
    state: { from: '/dashboard', userId: 123 } 
  });
  
  // Access state (not visible in URL)
  console.log(location.state); // { from: '/dashboard', userId: 123 }
};

// All parameters must be strings (URL constraint)
// Complex objects must be serialized (JSON.stringify)
```

**URL structure:**
```
https://app.com/users/123/posts/456?sort=date&filter=active#comments

Protocol: https://
Domain: app.com
Path params: /users/123/posts/456
Query params: ?sort=date&filter=active
Hash: #comments
```

### **React Navigation - Params Object**

```javascript
// JavaScript object-based parameter passing
import { useRoute, useNavigation } from '@react-navigation/native';

const UserProfile = () => {
  const route = useRoute();
  const navigation = useNavigation();
  
  // Params are JavaScript objects (can be any type!)
  console.log(route.params);
  /*
  {
    userId: 123,                    // Number (not string!)
    user: {                         // Complex object
      name: 'John',
      email: 'john@example.com',
      settings: { theme: 'dark' }
    },
    callback: () => console.log('Done'), // Function!
    timestamp: new Date(),          // Date object
    buffer: new Uint8Array([1,2,3]) // Binary data
  }
  */
  
  return <Text>User {route.params.userId}</Text>;
};

// Navigate with any JavaScript value
const HomeScreen = ({ navigation }) => {
  const navigateToProfile = () => {
    navigation.navigate('Profile', {
      userId: 123,
      user: {
        name: 'John',
        avatar: require('./avatar.png'), // Image reference
        permissions: ['read', 'write']
      },
      onSave: (data) => {
        // Callback function
        console.log('Saved:', data);
        saveToDatabase(data);
      }
    });
  };
  
  return <Button title="View Profile" onPress={navigateToProfile} />;
};

// Update params of current screen
const EditScreen = ({ navigation, route }) => {
  const updateParams = () => {
    navigation.setParams({
      edited: true,
      lastModified: new Date()
    });
    // Merges with existing params
  };
  
  return <Button title="Mark as Edited" onPress={updateParams} />;
};
```

**Key differences:**
```javascript
// REACT ROUTER: Everything is strings
const params = useParams(); // { id: "123" }
const numericId = parseInt(params.id); // Must parse

// REACT NAVIGATION: Native JavaScript types
const params = route.params; // { id: 123 }
// Already correct type!

// REACT ROUTER: Complex objects need serialization
navigate('/profile?data=' + encodeURIComponent(JSON.stringify(complexObject)));
// Later: JSON.parse(decodeURIComponent(searchParams.get('data')))

// REACT NAVIGATION: Pass objects directly
navigation.navigate('Profile', { data: complexObject });
// Later: route.params.data (original object)
```

---

## **5. Persistence and Deep Linking**

### **React Router - Built-in Persistence**

```javascript
// Browser automatically persists navigation state
// Refresh the page → URL remains → App recreates state from URL

// Example: User is on /users/123/edit
// User refreshes browser (F5)
// Browser requests: https://app.com/users/123/edit
// React Router matches route and renders UserEdit with params

// No additional code needed for basic persistence!

// Deep linking is native:
<a href="https://myapp.com/users/123">Direct link to user</a>
// Click link → Browser navigates → React Router handles
```

**Server-side rendering (SSR) support:**
```javascript
// React Router works seamlessly with SSR
import { StaticRouter } from 'react-router-dom/server';

// Server renders based on requested URL
function handleRequest(req, res) {
  const context = {};
  const html = ReactDOMServer.renderToString(
    <StaticRouter location={req.url} context={context}>
      <App />
    </StaticRouter>
  );
  res.send(html);
}
```

### **React Navigation - Manual Persistence**

```javascript
// React Navigation requires explicit persistence setup
import AsyncStorage from '@react-native-async-storage/async-storage';
import { NavigationContainer } from '@react-navigation/native';

const PERSISTENCE_KEY = 'NAVIGATION_STATE';

const App = () => {
  const [isReady, setIsReady] = useState(false);
  const [initialState, setInitialState] = useState();
  
  // Restore navigation state on app launch
  useEffect(() => {
    const restoreState = async () => {
      try {
        const savedStateString = await AsyncStorage.getItem(PERSISTENCE_KEY);
        const state = savedStateString ? JSON.parse(savedStateString) : undefined;
        setInitialState(state);
      } finally {
        setIsReady(true);
      }
    };
    
    restoreState();
  }, []);
  
  if (!isReady) {
    return <SplashScreen />;
  }
  
  return (
    <NavigationContainer
      initialState={initialState}
      onStateChange={(state) => {
        // Save state whenever it changes
        AsyncStorage.setItem(PERSISTENCE_KEY, JSON.stringify(state));
      }}
    >
      <RootNavigator />
    </NavigationContainer>
  );
};

// Now: User navigates to Profile screen → App closes
// Later: User reopens app → Restored to Profile screen
```

**Deep linking requires configuration:**
```javascript
// Configure URL structure mapping
const linking = {
  prefixes: ['myapp://', 'https://myapp.com'],
  config: {
    screens: {
      Home: '',
      Profile: 'users/:userId',
      Settings: {
        path: 'settings',
        screens: {
          General: 'general',
          Privacy: 'privacy',
        }
      },
    },
  },
};

const App = () => (
  <NavigationContainer linking={linking}>
    <RootNavigator />
  </NavigationContainer>
);

// Now deep links work:
// myapp://users/123 → Navigates to Profile with userId: 123
// https://myapp.com/settings/privacy → Navigates to Settings > Privacy

// But must handle:
// - URL → Navigation state conversion
// - Navigation state → URL conversion
// - Custom URL schemes (iOS: Info.plist, Android: AndroidManifest.xml)
```

**iOS URL Scheme (Info.plist):**
```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>myapp</string>
    </array>
  </dict>
</array>
```

**Android Intent Filter (AndroidManifest.xml):**
```xml
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="myapp" />
  <data android:scheme="https" android:host="myapp.com" />
</intent-filter>
```

---

## **6. Navigation Transitions and Animations**

### **React Router - Page Transitions (Manual)**

```javascript
// React Router doesn't provide transitions out of the box
// Must use libraries like react-transition-group or framer-motion

import { CSSTransition, TransitionGroup } from 'react-transition-group';
import { useLocation } from 'react-router-dom';

const AnimatedRoutes = () => {
  const location = useLocation();
  
  return (
    <TransitionGroup>
      <CSSTransition
        key={location.pathname}
        timeout={300}
        classNames="fade"
      >
        <Routes location={location}>
          <Route path="/" element={<Home />} />
          <Route path="/profile" element={<Profile />} />
        </Routes>
      </CSSTransition>
    </TransitionGroup>
  );
};

// CSS for transitions
/*
.fade-enter {
  opacity: 0;
}
.fade-enter-active {
  opacity: 1;
  transition: opacity 300ms;
}
.fade-exit {
  opacity: 1;
}
.fade-exit-active {
  opacity: 0;
  transition: opacity 300ms;
}
*/
```

**Challenges with web transitions:**
- Both old and new routes must be in DOM simultaneously
- Layout shifts can be jarring
- Browser back/forward don't trigger custom animations
- Performance considerations (painting, reflow)

### **React Navigation - Native Transitions (Built-in)**

```javascript
// React Navigation provides platform-native transitions automatically

// Stack Navigator - iOS slide, Android elevation
<Stack.Navigator
  screenOptions={{
    // iOS: slide from right, swipe to go back
    // Android: fade + elevation change
    animation: 'default',
  }}
>
  <Stack.Screen name="Home" component={Home} />
  <Stack.Screen name="Profile" component={Profile} />
</Stack.Navigator>

// Customize per screen
<Stack.Screen
  name="Modal"
  component={ModalScreen}
  options={{
    presentation: 'modal', // iOS modal from bottom
    animation: 'slide_from_bottom',
  }}
/>

// Custom animations
<Stack.Screen
  name="Details"
  component={Details}
  options={{
    transitionSpec: {
      open: {
        animation: 'timing',
        config: { duration: 300 }
      },
      close: {
        animation: 'spring',
        config: { stiffness: 1000, damping: 500 }
      }
    },
    cardStyleInterpolator: ({ current, layouts }) => {
      return {
        cardStyle: {
          transform: [
            {
              translateX: current.progress.interpolate({
                inputRange: [0, 1],
                outputRange: [layouts.screen.width, 0],
              }),
            },
          ],
        },
      };
    },
  }}
/>

// Gesture handling built-in
// - iOS: Swipe from left edge to go back
// - Android: Gesture navigation support
// All handled automatically!
```

**Platform-specific transitions:**
```javascript
// React Navigation automatically adapts to platform

// On iOS:
Home → Profile: Slides in from right with shadow
Profile → Home (back): Slides out to right
Modal presentation: Slides up from bottom

// On Android:
Home → Profile: Fades in with elevation change
Profile → Home (back): Fades out with elevation change
Back button: Hardware button triggers animation
```

---

## **7. Nested Navigation**

### **React Router - Outlet Pattern**

```javascript
// React Router uses <Outlet /> for nested routes
import { Outlet } from 'react-router-dom';

<Routes>
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route index element={<DashboardHome />} />
    <Route path="analytics" element={<Analytics />} />
    <Route path="settings" element={<Settings />} />
  </Route>
</Routes>

// DashboardLayout.js
const DashboardLayout = () => {
  return (
    <div className="dashboard">
      <Sidebar />
      <div className="content">
        <Outlet /> {/* Child route renders here */}
      </div>
    </div>
  );
};

// URL: /dashboard → Renders DashboardHome in <Outlet />
// URL: /dashboard/analytics → Renders Analytics in <Outlet />
// Sidebar stays mounted, only <Outlet /> content changes
```

**Nested route context:**
```javascript
// Parent and child share context naturally
const DashboardLayout = () => {
  const [theme, setTheme] = useState('light');
  
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Sidebar />
      <Outlet /> {/* Children can access theme */}
    </ThemeContext.Provider>
  );
};
```

### **React Navigation - Nested Navigators**

```javascript
// React Navigation nests navigators inside navigators
const Tab = createBottomTabNavigator();
const Stack = createNativeStackNavigator();
const Drawer = createDrawerNavigator();

// Root level: Drawer
const App = () => (
  <NavigationContainer>
    <Drawer.Navigator>
      <Drawer.Screen name="Home" component={TabNavigator} />
      <Drawer.Screen name="Settings" component={SettingsStack} />
    </Drawer.Navigator>
  </NavigationContainer>
);

// Second level: Tabs (inside drawer)
const TabNavigator = () => (
  <Tab.Navigator>
    <Tab.Screen name="Feed" component={FeedStack} />
    <Tab.Screen name="Profile" component={ProfileStack} />
  </Tab.Navigator>
);

// Third level: Stacks (inside tabs)
const FeedStack = () => (
  <Stack.Navigator>
    <Stack.Screen name="FeedList" component={FeedList} />
    <Stack.Screen name="FeedDetail" component={FeedDetail} />
  </Stack.Navigator>
);

// Navigation structure:
{
  Drawer: {
    routes: [{
      name: 'Home',
      state: {
        Tab: {
          routes: [{
            name: 'Feed',
            state: {
              Stack: {
                routes: [
                  { name: 'FeedList' },
                  { name: 'FeedDetail' } // Current screen
                ]
              }
            }
          }, {
            name: 'Profile'
          }]
        }
      }
    }]
  }
}
```

**Navigating across nested navigators:**
```javascript
// From FeedDetail, navigate to ProfileStack
navigation.navigate('Profile'); // Goes to Profile tab

// Navigate deep into another navigator
navigation.navigate('Feed', {
  screen: 'FeedDetail',
  params: { postId: 123 },
});

// Navigate to root
navigation.navigate('Home');

// Navigate to parent navigator
navigation.navigate('Home', {
  screen: 'Feed',
  params: { screen: 'FeedList' }
});
```

---

## **8. Access to Navigation Object**

### **React Router - Hooks Anywhere**

```javascript
// React Router hooks work in any component
import { useNavigate, useLocation, useParams } from 'react-router-dom';

// No need to pass props, just use hooks
const AnyComponent = () => {
  const navigate = useNavigate();
  const location = useLocation();
  const params = useParams();
  
  return (
    <button onClick={() => navigate('/home')}>
      Go Home
    </button>
  );
};

// Works because BrowserRouter provides context at root
```

### **React Navigation - Props or Hooks**

```javascript
// React Navigation: screens get navigation prop automatically
const HomeScreen = ({ navigation, route }) => {
  return (
    <Button
      title="Go to Profile"
      onPress={() => navigation.navigate('Profile')}
    />
  );
};

// Non-screen components need hooks
import { useNavigation, useRoute } from '@react-navigation/native';

const ChildComponent = () => {
  const navigation = useNavigation();
  const route = useRoute();
  
  return (
    <Button
      title="Go Back"
      onPress={() => navigation.goBack()}
    />
  );
};

// Alternative: withNavigation HOC (class components)
import { withNavigation } from '@react-navigation/native';

class MyComponent extends React.Component {
  render() {
    return (
      <Button
        title="Navigate"
        onPress={() => this.props.navigation.navigate('Screen')}
      />
    );
  }
}

export default withNavigation(MyComponent);
```

---

## **9. Code Splitting and Lazy Loading**

### **React Router - Native Code Splitting**

```javascript
// React Router integrates with React.lazy naturally
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

const Home = lazy(() => import('./screens/Home'));
const Profile = lazy(() => import('./screens/Profile'));
const Settings = lazy(() => import('./screens/Settings'));

const App = () => (
  <Suspense fallback={<LoadingSpinner />}>
    <Routes>
      <Route path="/" element={<Home />} />
      <Route path="/profile" element={<Profile />} />
      <Route path="/settings" element={<Settings />} />
    </Routes>
  </Suspense>
);

// Webpack/Vite automatically code-splits
// Each route in separate bundle
// Loaded on demand when user navigates
```

**Benefits on web:**
- Smaller initial bundle size
- Faster Time to Interactive (TTI)
- Better Core Web Vitals scores
- Progressive loading

### **React Navigation - Limited Code Splitting**

```javascript
// React Native doesn't have native code splitting like web
// All JavaScript bundles into single file by default

// Can manually implement with dynamic imports:
const ProfileScreen = () => {
  const [ProfileComponent, setProfileComponent] = useState(null);
  
  useEffect(() => {
    import('./screens/Profile').then(module => {
      setProfileComponent(() => module.default);
    });
  }, []);
  
  if (!ProfileComponent) {
    return <LoadingSpinner />;
  }
  
  return <ProfileComponent />;
};

// But this doesn't create separate bundles in React Native
// All code is in app.bundle.js

// Real code splitting requires:
// - Metro bundler configuration
// - Custom loading mechanism
// - Manual bundle management
// Much more complex than web
```

**Why code splitting is harder in React Native:**
- No HTTP/2 for parallel bundle loading
- App bundle is local file (not fetched over network)
- Native bridge overhead for loading remote code
- Security/App Store restrictions on remote code execution

**Alternative: RAM bundles (indexed):**
```javascript
// metro.config.js
module.exports = {
  serializer: {
    createModuleIdFactory: () => {
      return (path) => hash(path); // Consistent module IDs
    },
  },
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: true, // Lazy require() calls
      },
    }),
  },
};

// Components loaded on-demand within single bundle
```

---

## **10. Testing Approaches**

### **React Router - Test with MemoryRouter**

```javascript
import { render, screen } from '@testing-library/react';
import { MemoryRouter, Routes, Route } from 'react-router-dom';
import userEvent from '@testing-library/user-event';

test('navigation works', async () => {
  render(
    <MemoryRouter initialEntries={['/']}>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </MemoryRouter>
  );
  
  const link = screen.getByRole('link', { name: /profile/i });
  await userEvent.click(link);
  
  expect(screen.getByText(/Profile Page/i)).toBeInTheDocument();
});

// Test with specific initial route
test('profile page loads', () => {
  render(
    <MemoryRouter initialEntries={['/profile/123']}>
      <Routes>
        <Route path="/profile/:id" element={<Profile />} />
      </Routes>
    </MemoryRouter>
  );
  
  expect(screen.getByText(/User 123/i)).toBeInTheDocument();
});
```

### **React Navigation - Test with NavigationContainer**

```javascript
import { render, fireEvent } from '@testing-library/react-native';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

test('navigation works', () => {
  const { getByText, queryByText } = render(
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={Home} />
        <Stack.Screen name="Profile" component={Profile} />
      </Stack.Navigator>
    </NavigationContainer>
  );
  
  fireEvent.press(getByText(/Go to Profile/i));
  
  expect(queryByText(/Profile Page/i)).toBeTruthy();
});

// Test with initial route
test('profile page loads', () => {
  const { getByText } = render(
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Profile">
        <Stack.Screen 
          name="Profile" 
          component={Profile}
          initialParams={{ userId: 123 }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  );
  
  expect(getByText(/User 123/i)).toBeTruthy();
});

// Mock navigation
test('component with navigation', () => {
  const mockNavigate = jest.fn();
  const navigation = { navigate: mockNavigate };
  
  const { getByText } = render(
    <MyComponent navigation={navigation} />
  );
  
  fireEvent.press(getByText(/Navigate/i));
  expect(mockNavigate).toHaveBeenCalledWith('Profile', { userId: 123 });
});
```

---

## **11. Performance Considerations**

### **React Router Performance**

```javascript
// PROS:
// ✅ Only active route renders (unmounts others)
// ✅ Code splitting reduces bundle size
// ✅ Browser optimizations (preload, prefetch)

// CONS:
// ❌ Route changes can be expensive (mount/unmount)
// ❌ No built-in state persistence
// ❌ CSS transitions can be janky

// Optimization: Keep parent mounted
<div className="app">
  <Header /> {/* Stays mounted */}
  <Routes>
    <Route path="/" element={<Home />} />
    <Route path="/profile" element={<Profile />} />
  </Routes>
  <Footer /> {/* Stays mounted */}
</div>
```

### **React Navigation Performance**

```javascript
// PROS:
// ✅ Previous screens stay mounted (smooth back navigation)
// ✅ Native transitions (60 FPS)
// ✅ Gesture support built-in

// CONS:
// ❌ More memory usage (multiple screens mounted)
// ❌ All JavaScript in single bundle (initially)
// ❌ Bridge overhead for native animations

// Optimization: Unmount previous screens
<Stack.Navigator
  screenOptions={{
    detachPreviousScreen: true, // Unmount when not visible
  }}
>
  <Stack.Screen name="Home" component={Home} />
  <Stack.Screen name="Profile" component={Profile} />
</Stack.Navigator>

// Freeze previous screens to prevent unnecessary updates
import { enableFreeze } from 'react-native-screens';
enableFreeze(true);
```

---

## **Summary Table: Key Differences**

| Aspect | React Router | React Navigation |
|--------|-------------|------------------|
| **Environment** | Web (browser) | Native (iOS/Android) |
| **Source of Truth** | URL | JavaScript state |
| **History** | Browser API | Custom stack |
| **Persistence** | Automatic (URL) | Manual (AsyncStorage) |
| **Parameters** | Strings only | Any JavaScript type |
| **Transitions** | Manual (libraries) | Built-in (native) |
| **Back Button** | Browser button | Hardware/gesture |
| **Deep Linking** | Native | Requires config |
| **Code Splitting** | Easy (webpack) | Complex |
| **Mounting** | Unmounts old route | Keeps previous mounted |

---

## **When to Use Which?**

**React Router:**
- Web applications
- SEO is important
- Shareable URLs needed
- Server-side rendering
- Public content
- Browser as platform

**React Navigation:**
- Mobile applications (iOS/Android)
- Native app experience
- Platform-specific transitions
- Complex navigation (tabs + stacks + drawer)
- Gesture navigation
- Private/authenticated apps

---

## **Interview Talking Point:**

*"React Router and React Navigation differ fundamentally because they're designed for different platforms. React Router is URL-driven, leveraging the browser's history API and URLs as the source of truth, which provides automatic persistence, SEO, and shareable links. React Navigation is state-driven, maintaining a JavaScript navigation state object because native apps don't have URLs or browser history. This means React Navigation must manually implement features like deep linking, persistence, and back button handling. React Router unmounts components on navigation for memory efficiency, while React Navigation keeps previous screens mounted for smooth native transitions. The choice between them isn't about preference—it's determined by your platform: web uses React Router, mobile uses React Navigation."*

---
