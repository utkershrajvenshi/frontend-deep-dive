## **3. Ten Commonly Used Native Modules**

Here are **10 essential Native Modules** you'll encounter in React Native development, with explanations of what they do and why they require native implementation:

### **1. AsyncStorage**
```javascript
import AsyncStorage from '@react-native-async-storage/async-storage';

// Persistent key-value storage
await AsyncStorage.setItem('userToken', token);
const token = await AsyncStorage.getItem('userToken');
```

**Why native:** Accesses platform-specific persistent storage (iOS: UserDefaults/File System, Android: SharedPreferences/SQLite).

**Common use:** User preferences, authentication tokens, cached data.

---

### **2. CameraRoll / React Native Image Picker**
```javascript
import { launchCamera, launchImageLibrary } from 'react-native-image-picker';

const result = await launchImageLibrary({
  mediaType: 'photo',
  quality: 0.8,
});
```

**Why native:** Requires access to device camera hardware and photo library, platform permissions.

**Common use:** Profile pictures, photo uploads, document scanning.

---

### **3. Geolocation / React Native Geolocation**
```javascript
import Geolocation from '@react-native-community/geolocation';

Geolocation.getCurrentPosition(
  (position) => {
    console.log(position.coords.latitude, position.coords.longitude);
  },
  (error) => console.error(error),
  { enableHighAccuracy: true }
);
```

**Why native:** Accesses GPS hardware, WiFi/cell tower triangulation, requires platform permissions.

**Common use:** Maps, location-based services, delivery tracking.

---

### **4. Push Notifications (React Native Firebase / OneSignal)**
```javascript
import messaging from '@react-native-firebase/messaging';

// Request permission
await messaging().requestPermission();

// Handle notifications
messaging().onMessage(async (remoteMessage) => {
  console.log('Notification:', remoteMessage);
});
```

**Why native:** Integrates with Apple Push Notification Service (APNS) and Firebase Cloud Messaging (FCM), requires native setup.

**Common use:** User engagement, real-time alerts, messaging.

---

### **5. NetInfo**
```javascript
import NetInfo from '@react-native-community/netinfo';

const unsubscribe = NetInfo.addEventListener(state => {
  console.log('Connection type:', state.type);
  console.log('Is connected?', state.isConnected);
});
```

**Why native:** Monitors platform network APIs for connectivity changes.

**Common use:** Offline mode, sync indicators, error handling.

---

### **6. Linking**
```javascript
import { Linking } from 'react-native';

// Open URL in browser
await Linking.openURL('https://example.com');

// Handle deep links
Linking.addEventListener('url', ({ url }) => {
  console.log('Deep link:', url);
});
```

**Why native:** Opens platform-specific apps (browser, phone, email), handles deep linking schemes.

**Common use:** External links, phone calls, email, deep linking, OAuth flows.

---

### **7. PermissionsAndroid / React Native Permissions**
```javascript
import { PermissionsAndroid, Platform } from 'react-native';

const granted = await PermissionsAndroid.request(
  PermissionsAndroid.PERMISSIONS.CAMERA,
  {
    title: 'Camera Permission',
    message: 'App needs camera access',
  }
);

if (granted === PermissionsAndroid.RESULTS.GRANTED) {
  // Use camera
}
```

**Why native:** Interfaces with platform permission systems (iOS: Info.plist, Android: Manifest + runtime permissions).

**Common use:** Camera, location, contacts, microphone access.

---

### **8. Share**
```javascript
import { Share } from 'react-native';

await Share.share({
  message: 'Check out this app!',
  url: 'https://example.com',
  title: 'Share via',
});
```

**Why native:** Uses platform share sheets (iOS: UIActivityViewController, Android: Intent ACTION_SEND).

**Common use:** Social sharing, content distribution.

---

### **9. Clipboard**
```javascript
import Clipboard from '@react-native-clipboard/clipboard';

// Copy to clipboard
Clipboard.setString('Hello World');

// Read from clipboard
const text = await Clipboard.getString();
```

**Why native:** Accesses system clipboard (iOS: UIPasteboard, Android: ClipboardManager).

**Common use:** Copy/paste functionality, sharing codes, copying links.

---

### **10. Vibration**
```javascript
import { Vibration } from 'react-native';

// Simple vibration
Vibration.vibrate(1000); // 1 second

// Pattern: vibrate 500ms, pause 500ms, vibrate 1000ms
Vibration.vibrate([500, 500, 1000]);
```

**Why native:** Controls device haptic/vibration hardware.

**Common use:** Haptic feedback, notifications, game events.

---

### **Honorable Mentions (5 More Important Ones):**

**11. Keychain (iOS) / Secure Storage**
```javascript
import * as Keychain from 'react-native-keychain';

// Store securely
await Keychain.setGenericPassword('username', 'password');

// Retrieve
const credentials = await Keychain.getGenericPassword();
```
**Use:** Secure credential storage (tokens, passwords).

**12. In-App Purchase (IAP)**
```javascript
import * as RNIap from 'react-native-iap';

const products = await RNIap.getProducts(['product_id']);
await RNIap.requestPurchase('product_id');
```
**Use:** Monetization, subscriptions.

**13. Biometric Authentication (FaceID/TouchID/Fingerprint)**
```javascript
import ReactNativeBiometrics from 'react-native-biometrics';

const { success } = await rnBiometrics.simplePrompt({
  promptMessage: 'Confirm fingerprint'
});
```
**Use:** Secure authentication, payment confirmation.

**14. File System Access**
```javascript
import RNFS from 'react-native-fs';

const files = await RNFS.readDir(RNFS.DocumentDirectoryPath);
await RNFS.writeFile(filePath, data, 'utf8');
```
**Use:** File downloads, caching, document management.

**15. SQLite Database**
```javascript
import SQLite from 'react-native-sqlite-storage';

const db = await SQLite.openDatabase({ name: 'mydb.db' });
await db.executeSql('SELECT * FROM users');
```
**Use:** Complex data storage, offline-first apps, relational data.

---

### **Why These Require Native Modules:**

All these modules need native implementation because they:

1. **Access Hardware:** Camera, GPS, sensors, vibration motor
2. **Use Platform APIs:** Permissions, notifications, app store
3. **Require Security:** Keychain, biometrics, secure storage
4. **Need Performance:** SQLite, file operations, cryptography
5. **Platform Integration:** Share sheets, deep linking, system UI

**Interview insight:**

*"Native modules are necessary when JavaScript alone can't access platform-specific APIs, hardware, or features. Common examples include AsyncStorage for persistent data, Geolocation for GPS access, Push Notifications for APNS/FCM integration, and Permissions for runtime permission requests. Understanding when to use native modules versus pure JavaScript is key to building functional React Native applications - essentially, anything that touches device hardware, platform-specific APIs, or requires operating system integration will need a native module."*

---
