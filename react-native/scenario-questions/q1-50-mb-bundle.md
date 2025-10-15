# Question 1: Your app's bundle size is 50MB and users on slow networks are abandoning the app during download. How do you address this?

## **Interview Approach:**

*"A 50MB bundle is definitely problematic. I'd approach this systematically: first audit what's contributing to the size, then apply targeted optimizations. Let me break down my process."*

---

## **Phase 1: Analyze Bundle Composition**

### **Step 1: Generate Bundle Analysis**

```bash
# For React Native, analyze the bundle
npx react-native bundle \
  --platform android \
  --dev false \
  --entry-file index.js \
  --bundle-output bundle.js \
  --assets-dest assets

# Analyze what's in the bundle
npx react-native-bundle-visualizer

# Or use source-map-explorer
npm install -g source-map-explorer
source-map-explorer bundle.js bundle.js.map
```

**What to look for:**
```javascript
// Common culprits in 50MB bundles:
1. Large npm packages (moment.js, lodash full imports)
2. Unoptimized images in the bundle
3. Multiple copies of same library
4. Development code in production
5. Unnecessary native modules
6. Unshaken/unused code
```

### **Step 2: Check Native Binary Size**

```bash
# Android APK analysis
# Build → Analyze APK in Android Studio

# Common findings:
- Native libraries (.so files): 15-20MB
- Resources (images, fonts): 10-15MB
- DEX files (Java/Kotlin code): 5-10MB
- Assets (JS bundle): 5-10MB

# iOS IPA analysis
# Xcode → Organizer → App Thinning Size Report

# Common findings:
- Frameworks: 10-15MB
- App binary: 8-12MB
- Resources: 5-10MB
- Assets: 5-10MB
```

---

## **Phase 2: JavaScript Bundle Optimization**

### **Optimization 1: Remove Unused Dependencies**

```bash
# Find unused dependencies
npx depcheck

# Example findings that bloat bundles:
```

```javascript
// ❌ BAD: Importing entire lodash (70KB+)
import _ from 'lodash';
_.debounce(fn, 300);
_.groupBy(items, 'category');

// ✅ GOOD: Import specific functions
import debounce from 'lodash/debounce';
import groupBy from 'lodash/groupBy';
// Only includes what you use (5KB)

// ❌ BAD: Moment.js (entire library + locales = 300KB+!)
import moment from 'moment';
moment().format('YYYY-MM-DD');

// ✅ GOOD: Use date-fns (tree-shakeable, 10KB per function)
import { format } from 'date-fns';
format(new Date(), 'yyyy-MM-dd');

// Or use native Intl API (0KB!)
new Intl.DateTimeFormat('en-US').format(new Date());
```

**Common bloat packages to replace:**

```javascript
// Heavy packages and their lightweight alternatives:
{
  "moment": "2.29.4",        // 300KB → Replace with date-fns (tree-shakeable)
  "lodash": "4.17.21",       // 70KB → Import specific functions or use native
  "axios": "1.4.0",          // 15KB → Use fetch API (native)
  "uuid": "9.0.0",           // 10KB → Use crypto.randomUUID() (native)
  "validator": "13.9.0",     // 150KB → Write simple validators
  "react-native-vector-icons": "9.2.0" // Entire icon set → Only include used icons
}
```

### **Optimization 2: Enable Hermes and Minification**

```javascript
// android/app/build.gradle
project.ext.react = [
    enableHermes: true,  // Bytecode compilation
    hermesFlags: ["-O", "-output-source-map"], // Optimize
]

// android/gradle.properties
hermesEnabled=true

// iOS Podfile
use_react_native!(
  :hermes_enabled => true,
  :fabric_enabled => false,
)

// metro.config.js
module.exports = {
  transformer: {
    minifierPath: 'metro-minify-terser',
    minifierConfig: {
      compress: {
        drop_console: true, // Remove console.log in production
        dead_code: true,
        drop_debugger: true,
      },
      mangle: true,
      output: {
        comments: false,
      },
    },
  },
};
```

**Impact:**
```
Before Hermes: 
- JS bundle: 10MB
- Parse time: 2-3 seconds

After Hermes:
- Bytecode bundle: 7MB (30% smaller)
- Parse time: <100ms (30x faster)
```

### **Optimization 3: Code Splitting and Lazy Loading**

```javascript
// ❌ BAD: Import everything upfront
import HomeScreen from './screens/HomeScreen';
import ProfileScreen from './screens/ProfileScreen';
import SettingsScreen from './screens/SettingsScreen';
import MapScreen from './screens/MapScreen'; // Heavy! react-native-maps
import VideoPlayer from './screens/VideoPlayer'; // Heavy! video libraries
import ARViewer from './screens/ARViewer'; // Heavy! AR libraries

// All loaded even if user never visits these screens

// ✅ GOOD: Lazy load heavy screens
const ProfileScreen = React.lazy(() => import('./screens/ProfileScreen'));
const SettingsScreen = React.lazy(() => import('./screens/SettingsScreen'));
const MapScreen = React.lazy(() => import('./screens/MapScreen'));
const VideoPlayer = React.lazy(() => import('./screens/VideoPlayer'));
const ARViewer = React.lazy(() => import('./screens/ARViewer'));

// Wrap in Suspense
const App = () => (
  <NavigationContainer>
    <Stack.Navigator>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen name="Profile">
        {(props) => (
          <Suspense fallback={<LoadingScreen />}>
            <ProfileScreen {...props} />
          </Suspense>
        )}
      </Stack.Screen>
    </Stack.Navigator>
  </NavigationContainer>
);

// Better: Use dynamic imports with React Navigation
const App = () => (
  <NavigationContainer>
    <Stack.Navigator>
      <Stack.Screen name="Home" component={HomeScreen} />
      <Stack.Screen 
        name="Profile" 
        getComponent={() => require('./screens/ProfileScreen').default}
      />
      <Stack.Screen 
        name="Map" 
        getComponent={() => require('./screens/MapScreen').default}
      />
    </Stack.Navigator>
  </NavigationContainer>
);
```

**Advanced: Over-The-Air (OTA) Updates with CodePush**

```javascript
// Split bundle into base + feature modules
// Base: Essential screens (Home, Login)
// Features: Loaded on-demand via CodePush

import codePush from 'react-native-code-push';

// Download feature bundles in background
const FeatureScreen = () => {
  const [loaded, setLoaded] = useState(false);
  
  useEffect(() => {
    codePush.sync({
      installMode: codePush.InstallMode.ON_NEXT_RESTART,
      updateDialog: false,
    }, (status) => {
      if (status === codePush.SyncStatus.UPDATE_INSTALLED) {
        setLoaded(true);
      }
    });
  }, []);
  
  if (!loaded) return <DownloadingScreen />;
  return <ActualFeature />;
};
```

### **Optimization 4: Tree Shaking and Dead Code Elimination**

```javascript
// metro.config.js - Enable aggressive tree shaking
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: true, // Inline require() calls for better tree shaking
      },
    }),
  },
};

// ❌ BAD: Default export prevents tree shaking
// utils.js
export default {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
  multiply: (a, b) => a * b,
  divide: (a, b) => a / b,
  // ... 50 more functions
};

// usage.js
import utils from './utils';
utils.add(1, 2);
// ALL 50+ functions included in bundle!

// ✅ GOOD: Named exports enable tree shaking
// utils.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export const multiply = (a, b) => a * b;
export const divide = (a, b) => a / b;

// usage.js
import { add } from './utils';
add(1, 2);
// Only 'add' function included in bundle!
```

---

## **Phase 3: Asset Optimization**

### **Optimization 5: Image Optimization**

```javascript
// ❌ BAD: Large images in bundle
src/
  assets/
    splash.png (2000x2000, 5MB!)
    logo.png (1000x1000, 2MB)
    background.jpg (4000x3000, 8MB)
    icons/
      home.png (512x512, 500KB each × 50 icons = 25MB!)

// Bundle size: 40MB just from images!

// ✅ GOOD: Optimized images
src/
  assets/
    splash.webp (2000x2000, 200KB) // 25x smaller!
    splash@2x.webp (1000x1000, 80KB)
    splash@3x.webp (667x667, 50KB)
    logo.webp (500x500, 50KB)
    // Use vector icons instead of PNGs
```

**Image optimization script:**

```bash
# Install optimization tools
npm install -g imagemin-cli imagemin-webp imagemin-mozjpeg

# Optimize all images
imagemin assets/*.{jpg,png} \
  --out-dir=assets-optimized \
  --plugin=webp --plugin=mozjpeg

# Results:
# Before: 40MB
# After: 2MB (20x reduction!)
```

**Use vector icons instead of images:**

```javascript
// ❌ BAD: PNG icon set (25MB for 50 icons)
<Image source={require('./assets/icons/home.png')} />

// ✅ GOOD: Vector icons (100KB total)
import Icon from 'react-native-vector-icons/MaterialIcons';

<Icon name="home" size={24} color="#000" />

// Configure to only include used icons
// react-native.config.js
module.exports = {
  dependencies: {
    'react-native-vector-icons': {
      platforms: {
        ios: null,
      },
    },
  },
  assets: ['./assets/fonts'], // Only include custom fonts
};
```

### **Optimization 6: Use Remote Assets**

```javascript
// ❌ BAD: All images in bundle
const IMAGES = {
  banner1: require('./assets/banner1.jpg'),
  banner2: require('./assets/banner2.jpg'),
  banner3: require('./assets/banner3.jpg'),
  // ... 50 more images
};
// Bundle: +50MB

// ✅ GOOD: Load from CDN
const IMAGES = {
  banner1: 'https://cdn.myapp.com/banners/1.webp',
  banner2: 'https://cdn.myapp.com/banners/2.webp',
  banner3: 'https://cdn.myapp.com/banners/3.webp',
};

<FastImage 
  source={{ uri: IMAGES.banner1 }}
  style={styles.banner}
/>
// Bundle: +0MB, images loaded on demand

// Preload critical images
useEffect(() => {
  FastImage.preload([
    { uri: IMAGES.banner1 },
    { uri: IMAGES.banner2 },
  ]);
}, []);
```

---

## **Phase 4: Native Binary Optimization**

### **Optimization 7: Remove Unused Native Modules**

```javascript
// Check node_modules for unused native dependencies
// package.json - Remove unused packages

{
  "dependencies": {
    // ❌ Unnecessary native modules (each adds 1-5MB)
    "react-native-camera": "^4.2.1",        // Not using camera
    "react-native-maps": "^1.7.1",          // Not using maps
    "react-native-video": "^5.2.1",         // Not using video
    "@react-native-firebase/app": "^18.0", // Full Firebase (use only needed modules)
  }
}

// Each unused native module adds:
// - Native libraries (.so, .framework files): 2-10MB
// - Resources: 1-5MB
// - JS bridge code: 100-500KB
```

**Firebase optimization example:**

```javascript
// ❌ BAD: Import entire Firebase
import firebase from '@react-native-firebase/app';
import '@react-native-firebase/analytics';
import '@react-native-firebase/auth';
import '@react-native-firebase/firestore';
import '@react-native-firebase/messaging';
import '@react-native-firebase/storage';
// Total: 15-20MB

// ✅ GOOD: Only import what you use
import analytics from '@react-native-firebase/analytics';
import auth from '@react-native-firebase/auth';
// Total: 5-8MB
```

### **Optimization 8: Enable ProGuard/R8 (Android)**

```gradle
// android/app/build.gradle
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
}

// proguard-rules.pro
-dontwarn **
-keep class com.myapp.** { *; }
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}

# Impact: 30-40% reduction in Android APK size
```

### **Optimization 9: App Bundle and Dynamic Features (Android)**

```gradle
// android/app/build.gradle
android {
    bundle {
        language {
            enableSplit = true // Separate APKs per language
        }
        density {
            enableSplit = true // Separate APKs per screen density
        }
        abi {
            enableSplit = true // Separate APKs per CPU architecture
        }
    }
}

// Build App Bundle instead of APK
./gradlew bundleRelease

// Result:
// Universal APK: 50MB
// User downloads: 15-20MB (only their device's assets)
```

### **Optimization 10: Strip Debug Symbols (iOS)**

```ruby
# ios/Podfile
post_install do |installer|
  installer.pods_project.targets.each do |target|
    target.build_configurations.each do |config|
      # Strip debug symbols
      config.build_settings['STRIP_INSTALLED_PRODUCT'] = 'YES'
      config.build_settings['COPY_PHASE_STRIP'] = 'YES'
      config.build_settings['STRIP_STYLE'] = 'non-global'
      
      # Enable bitcode (allows Apple to optimize)
      config.build_settings['ENABLE_BITCODE'] = 'YES'
    end
  end
end

# Impact: 20-30% reduction in iOS IPA size
```

---

## **Phase 5: Progressive Download Strategy**

### **Strategy 1: Implement CodePush for Incremental Updates**

```javascript
import codePush from 'react-native-code-push';

// Split features into core + optional
const App = () => {
  useEffect(() => {
    // Silent background updates
    codePush.sync({
      updateDialog: false,
      installMode: codePush.InstallMode.ON_NEXT_RESTART,
    });
  }, []);
  
  return <MainApp />;
};

export default codePush({
  checkFrequency: codePush.CheckFrequency.ON_APP_RESUME,
})(App);

// Initial download: 15MB (core features)
// Background download: 10MB (additional features)
// Total: 25MB, but user gets app faster
```

### **Strategy 2: On-Demand Resources (iOS)**

```xml
<!-- ios/MyApp/Info.plist -->
<key>NSBundleResourceRequest</key>
<dict>
    <key>NSBundleInitialGuessedTags</key>
    <array>
        <string>level1</string>
    </array>
    <key>NSBundleResourceRequestTags</key>
    <array>
        <string>level2</string>
        <string>level3</string>
    </array>
</dict>
```

```javascript
// Request resources on-demand
import { NativeModules } from 'react-native';

const downloadLevel = async (level) => {
  const request = new NSBundleResourceRequest([`level${level}`]);
  await request.beginAccessingResources();
  // Resources now available
};
```

---

## **Complete Optimization Results**

### **Before Optimization:**
```
JavaScript Bundle: 10MB
  - moment.js: 300KB
  - lodash (full): 70KB
  - Unused libraries: 2MB
  - Unoptimized code: 7.6MB

Native Binary (Android): 25MB
  - Unused native modules: 8MB
  - Debug symbols: 5MB
  - Unoptimized resources: 12MB

Assets: 15MB
  - PNG images: 10MB
  - Large backgrounds: 5MB

Total APK: 50MB
```

### **After Optimization:**
```
JavaScript Bundle: 3MB (70% reduction)
  - date-fns (tree-shaken): 10KB
  - lodash (specific imports): 5KB
  - Code splitting: Lazy loaded 2MB
  - Hermes bytecode: 3MB

Native Binary (Android): 8MB (68% reduction)
  - Removed unused modules: saved 8MB
  - ProGuard enabled: saved 5MB
  - Stripped resources: saved 4MB

Assets: 2MB (87% reduction)
  - WebP images: 500KB
  - Vector icons: 100KB
  - CDN-hosted: 1.4MB saved

Total APK: 13MB (74% reduction!)

With App Bundle: 
User download: 6-8MB per device (85% reduction)
```

---

## **Long-term Strategy**

```javascript
// Implement progressive web app (PWA) fallback
// For users on very slow networks, offer web version first

// 1. Create lightweight web version
// 2. Prompt to install native app after user engagement
// 3. Native app downloads in background

// App Store listing
// "Quick start with web version, upgrade to full app for best experience"

// This reduces abandonment during download
```

---
