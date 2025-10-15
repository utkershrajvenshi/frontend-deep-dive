## Answers to the following questions:

- You need to build a feature that works differently on iOS and Android (like iOS using swipe gestures and Android using a FAB button). How do you architect this to keep code maintainable?
- Your app needs to work offline and sync data when the connection returns. The user might make conflicting changes on multiple devices. Design a solution for this scenario.
- You're implementing a multi-step form wizard that should persist progress, allow users to go back/forward, and handle the Android back button correctly. How do you implement this?
- Your app has deep links that can open any screen. Some screens require authentication, some require data to be loaded first. How do you handle all these cases gracefully?
- A screen in your app can be reached from multiple entry points (push notification, deep link, tab navigation, modal). The back behavior should be different depending on the entry point. How do you handle this?
- Your app has real-time notifications that can affect data on any screen. How do you ensure all screens reflect the latest data without causing unnecessary re-renders?
- You're building a banking app. How do you securely store authentication tokens, prevent screenshot capture on sensitive screens, and ensure data isn't logged?
- Your API keys need to be different for dev, staging, and production, but shouldn't be committed to git. How do you manage this in the build process?
- You pushed a release with a critical bug affecting 30% of users. You have a fix ready. Walk me through the fastest way to get this fix to users, considering different app stores' review times.
- Your app works perfectly on the simulator but crashes on real devices. What could be different and how do you debug?

# Answers

## **1. Platform-Specific Features (iOS Swipe vs Android FAB)**

### **Architecture Approach:**

**Strategy: Component Abstraction with Platform Detection**

I'd create an abstraction layer that exposes a consistent API while implementing platform-specific behavior underneath. The key is deciding the right level of abstraction.

**Option 1: Platform-Specific Components (Recommended)**

Create separate component files using React Native's Platform-specific extensions:
- `ActionTrigger.ios.js` - Implements swipe gesture using react-native-gesture-handler
- `ActionTrigger.android.js` - Implements FAB using react-native-paper or custom component

The parent just imports `ActionTrigger` and React Native automatically picks the right file. This keeps platform logic completely separated and makes each implementation independently maintainable.

**Option 2: Single Component with Platform Branches**

Use `Platform.select()` or `Platform.OS` conditionals within a single component. This works for simple differences but becomes messy if the implementations diverge significantly. I'd only use this if the platform differences are truly minimal (like styling tweaks).

**Option 3: Composition Pattern**

Create a wrapper component that conditionally renders either `SwipeAction` or `FABAction` based on platform. This is useful when you want both behaviors to be testable in isolation and potentially reusable elsewhere.

**Why This Matters:**

The anti-pattern is scattering `if (Platform.OS === 'ios')` checks throughout your app. That becomes unmaintainable quickly. By encapsulating platform differences at component boundaries, you:
- Keep business logic platform-agnostic
- Make platform-specific code easy to locate and modify
- Can test each platform independently
- Allow designers to iterate on platform conventions separately

**Integration with Feature Flags:**

I'd also consider whether this needs to be configurable beyond just platform. Maybe certain users prefer FAB on iOS too. In that case, combine platform detection with feature flags or user preferences.

---

## **2. Offline-First with Sync and Conflict Resolution**

### **Architecture Design:**

**Core Strategy: Event Sourcing + CRDT or Operational Transformation**

This is a complex problem requiring several architectural layers:

**Data Layer:**
- Local database as source of truth (WatermelonDB, Realm, or SQLite)
- Every user action creates an immutable event/operation, not just state updates
- Operations are timestamped with vector clocks or Hybrid Logical Clocks (HLC) to establish causal ordering across devices

**Sync Strategy:**

1. **Optimistic Updates**: All user actions immediately update local database
2. **Operation Log**: Each change creates an operation record (insert, update, delete) with:
   - Unique operation ID
   - Timestamp/HLC
   - Entity ID and type
   - Change payload
   - Sync status (pending, synced, conflicted)

3. **Background Sync**: When online, sync service:
   - Uploads pending operations to server
   - Downloads operations from server since last sync
   - Server acts as central coordinator maintaining global operation log

**Conflict Resolution Strategies:**

**For Simple Data (Last-Write-Wins with Timestamp):**
Use timestamps or HLC to determine winner. Works for independent fields.

**For Complex Data (CRDTs):**
Use Conflict-free Replicated Data Types:
- For counters: use CRDT counters that can be incremented/decremented independently
- For text: use Automerge or Yjs for collaborative text editing
- For sets: use CRDT sets that merge additions/deletions deterministically

**For Business Logic Conflicts:**
Implement custom resolution:
- Define conflict detection rules (e.g., editing same field)
- Store conflicting versions
- Present UI for user to resolve (like git merge conflicts)
- Or implement automatic resolution based on business rules (e.g., sum amounts, keep latest timestamp)

**Implementation Approach:**

Use a library like:
- **WatermelonDB** with sync primitives built-in
- **RxDB** with replication plugins
- **PouchDB** with CouchDB sync protocol
- **Realm** with MongoDB Realm Sync

Or build custom:
- Redux or Zustand for state management
- Middleware that logs all operations
- NetInfo to detect connectivity
- Queue system for pending operations
- Reconciliation engine that applies remote operations and detects conflicts

**Edge Cases to Handle:**
- User makes changes while sync is in progress
- Network drops mid-sync (ensure atomic sync batches)
- User deletes data that was modified on another device
- Clock skew between devices (why vector clocks matter)
- Partial sync failures (need retry logic and idempotency)

**Server Requirements:**
- Store global operation log per user
- Implement conflict detection and resolution logic
- Provide API to fetch operations since timestamp
- Handle operation ordering and validation
- Garbage collect old operations after devices sync

---

## **3. Multi-Step Form Wizard with Persistence**

### **State Management Architecture:**

**Core Components:**

1. **Form State Management**: Use React Context or Zustand to maintain form state across steps
2. **Persistence Layer**: AsyncStorage for saving progress
3. **Navigation Control**: Stack navigator with custom back handling
4. **Step Configuration**: Centralized definition of steps, validation, and transitions

**State Structure:**

Maintain a form state object containing:
- Current step index
- Data for each step (keyed by step ID)
- Validation status per step
- Metadata (started timestamp, last saved timestamp)

**Persistence Strategy:**

- Auto-save to AsyncStorage on every step change
- Debounce intermediate saves (e.g., as user types)
- Store with unique key (user ID + form ID)
- On app launch, check for in-progress forms and offer to resume
- Clear persisted data on successful submission or explicit abandonment

**Navigation Architecture:**

Use React Navigation Stack with custom configuration:
- Each step is a screen in the stack
- Configure `gestureEnabled: false` to prevent accidental swipe backs
- Implement custom header with step indicator and explicit back/next buttons

**Android Back Button Handling:**

Use `useEffect` with `BackHandler`:
- On back press, navigate to previous step if not on first step
- On first step, show confirmation dialog ("Exit without saving?")
- Allow override via `hardwareBackPress` listener
- Return true to prevent default back behavior

**Step Validation:**

Implement validation at multiple levels:
- Field-level validation (as user types)
- Step-level validation (before allowing next)
- Final validation before submission

Use a validation schema library (Yup, Zod, or custom) to define rules centrally.

**Progress Indicator:**

Show visual progress:
- Step numbers or progress bar
- Completed/current/upcoming states
- Allow jumping to previously completed steps but not ahead

**Error Handling:**

- Network failures on submission: Queue for retry
- Validation failures: Highlight errors, prevent navigation
- App crashes: Restore from AsyncStorage on relaunch

**Accessibility Considerations:**

- Announce step changes to screen readers
- Ensure focus management when navigating steps
- Provide clear progress context

---

## **4. Deep Linking with Authentication and Data Requirements**

### **Navigation Architecture:**

**Layered Approach:**

**Layer 1: Link Configuration**
Define deep link structure with metadata:
- URL pattern (e.g., `/profile/:userId`)
- Screen name
- Authentication required (boolean)
- Data requirements (list of resources to preload)
- Fallback behavior

**Layer 2: Link Parser**
- Parse incoming URL using Linking API or React Navigation linking config
- Extract route name and parameters
- Determine target screen and requirements

**Layer 3: Guard System**
Implement middleware that checks preconditions before navigation:

**Authentication Guard:**
- Check if user is authenticated (token exists and valid)
- If not: Store deep link target, navigate to login screen
- After successful login: Redirect to original deep link target
- This requires storing the "intended destination" in state

**Data Preloading Guard:**
- Identify required data for target screen
- Show loading state while fetching
- On success: Navigate with preloaded data as params
- On failure: Show error screen with retry option

**Implementation Flow:**

1. App receives deep link (via `Linking.getInitialURL()` or `Linking.addEventListener('url')`)
2. Parse URL to determine target screen and params
3. Check authentication status:
   - Authenticated → Proceed to step 4
   - Not authenticated → Navigate to login, store target
4. Check data requirements:
   - No data needed → Navigate immediately
   - Data needed → Show loading, fetch data
5. Navigate to target screen with params and preloaded data
6. If any step fails, navigate to appropriate error/fallback screen

**State Management:**

Use a navigation service or context to track:
- Pending deep link destination
- Loading state
- Error state
- Authentication status

**Edge Cases:**

- Deep link received while app is foregrounded on different screen
- Multiple deep links in quick succession
- Deep link to non-existent resource (404 handling)
- Deep link with invalid authentication (redirect to login)
- Expired deep link content (show friendly message)
- Deep link requires specific app version (version gating)

**Testing Strategy:**

Test matrix covering:
- Authenticated vs unauthenticated users
- App cold start vs already running
- Valid vs invalid deep links
- Network available vs offline
- Data loads successfully vs fails

**Universal Links (iOS) and App Links (Android):**

Ensure proper configuration:
- iOS: Associated domains entitlement, apple-app-site-association file
- Android: Intent filters in AndroidManifest.xml, assetlinks.json file
- Verify domain ownership to prevent hijacking

---

## **5. Context-Aware Back Navigation**

### **Navigation Context Tracking:**

**Strategy: Track Entry Point and Build Custom Back Behavior**

The core challenge is knowing how the user arrived at the current screen.

**Implementation Approach:**

**Option 1: Navigation State Metadata**
Pass a `navigationContext` parameter when navigating:
- `from: 'tab' | 'modal' | 'deeplink' | 'notification'`
- `returnTo`: Optional screen to return to
- `closeAction`: Custom action on back (close modal, go to specific screen, exit app)

**Option 2: Stack Analysis**
Inspect navigation state to determine entry point:
- Check if there's only one screen in stack (likely deep link or notification)
- Check if current screen is in modal stack
- Check if navigated from specific tab

**Option 3: Custom Navigator**
Create wrapper around React Navigation that tracks context:
- Maintain parallel state tracking entry points
- Expose custom back handlers based on context

**Back Behavior by Entry Point:**

**Push Notification:**
- User opened app via notification directly to screen
- Stack might be: [Screen] (single item)
- Back action: Navigate to home screen instead of closing app

**Deep Link:**
- Similar to notification
- Back action: Navigate to home or last known screen

**Tab Navigation:**
- User navigated within tab structure
- Back action: Normal stack pop, or go to tab root

**Modal:**
- Presented as modal overlay
- Back action: Close modal (dismiss), return to underlying screen

**Implementation Using React Navigation:**

Use `navigation.getState()` to inspect current stack:
- Check `routes.length` - if 1, likely entry from outside
- Check route params for `fromNotification` or `fromDeepLink` flags
- Use `navigation.reset()` to restructure stack if needed

**Custom Back Handler:**

Create a hook `useSmartBack` that:
- Determines entry point from route params or stack analysis
- Returns appropriate back handler function
- Can be used by hardware back button and UI back buttons

**Android Hardware Back Button:**

Use `BackHandler` with context-aware logic:
- If modal: Dismiss modal
- If root of tab: Show exit confirmation or switch to main tab
- If from notification/deep link: Navigate to home instead of exit
- Otherwise: Default back behavior

**User Experience Considerations:**

The goal is making back behavior intuitive:
- User should feel they're "unwinding" their navigation path
- Coming from outside should feel like they can get to a known state (home)
- Modal dismissal should return to where they were before modal
- Tab switching shouldn't confuse back behavior

**State Reset on Deep Entry:**

When user enters via notification/deep link:
- Consider resetting navigation state to: [Home, TargetScreen]
- This ensures back button goes somewhere logical (home)
- Prevents confusing navigation states

---

## **6. Real-Time Notifications Affecting Multiple Screens**

### **State Synchronization Architecture:**

**Core Strategy: Event-Driven Architecture with Reactive State Management**

The challenge is propagating updates to all relevant screens without causing unnecessary re-renders.

**Architecture Layers:**

**1. Global Event Bus / Message Broker:**
- WebSocket connection to backend for real-time events
- Local event emitter for app-wide events
- Events are typed (e.g., `USER_UPDATED`, `POST_LIKED`, `MESSAGE_RECEIVED`)

**2. Centralized State Store:**
- Use Redux, Zustand, or Jotai for global state
- Normalize data (entities by ID in lookup tables)
- State is updated in response to events
- Components subscribe to specific slices

**3. Selective Re-Rendering:**
- Use selectors to derive only needed data
- Memoization at multiple levels (selectors, components)
- React.memo with custom comparison
- Fine-grained subscriptions

**Implementation Pattern:**

**Event Listener Service:**
- Singleton service that maintains WebSocket connection
- Listens for events from server
- Dispatches actions to state store
- Handles reconnection, buffering during offline

**State Normalization:**
- Store entities by ID: `{ users: { [id]: user }, posts: { [id]: post } }`
- Components reference by ID, not nested objects
- Updates to one entity don't trigger re-renders of unrelated components

**Component Subscription:**
- Components subscribe to specific IDs or filtered data
- Use selector hooks: `useUser(userId)` only re-renders if that user changes
- Avoid subscribing to entire collections

**Optimistic Updates with Reconciliation:**
- User actions update local state immediately
- Send action to server
- When server confirms, reconcile local state with server response
- If server response differs, apply conflict resolution

**Preventing Unnecessary Re-Renders:**

**Technique 1: Selector Specificity**
Create narrow selectors that only return changed data. Use libraries like Reselect that memoize selector results.

**Technique 2: Subscription Scoping**
Components should subscribe to the minimal data they need. Use context providers at appropriate levels, not just top-level.

**Technique 3: Reference Equality**
When updating state, maintain reference equality for unchanged objects. Libraries like Immer help with this.

**Technique 4: Debouncing/Throttling**
For high-frequency events (like typing indicators), debounce state updates.

**Handling Different Event Types:**

**Data Mutations (CRUD):**
- Update normalized state
- Components subscribed to that entity re-render

**Ephemeral Events (typing indicators, online status):**
- Store in separate slice of state
- Short TTL, auto-cleanup
- Components opt-in to these updates

**Invalidation Events:**
- Some changes require refetching data
- Invalidate cached queries (React Query, SWR)
- Refetch happens in background

**Implementation with React Query / SWR:**

These libraries provide:
- Cache management with automatic invalidation
- Refetching on focus, reconnect
- Optimistic updates
- Background refetching

When event arrives:
- Invalidate relevant queries
- Libraries automatically refetch in background
- Components re-render with fresh data

**WebSocket Management:**

Use a library like Socket.io-client with:
- Automatic reconnection
- Event acknowledgment
- Room/channel subscriptions
- Buffering during disconnection

**Testing Strategy:**

Mock WebSocket connection and verify:
- Events correctly update state
- Components re-render only when needed
- Offline/online transitions handled gracefully
- Multiple simultaneous events processed correctly

---

## **7. Banking App Security**

### **Comprehensive Security Strategy:**

**Token Storage:**

**Never use AsyncStorage for sensitive data** - it's not encrypted and easily accessible.

**Secure Storage Options:**

**iOS:** Keychain Services
- Use react-native-keychain or react-native-sensitive-info
- Encrypted, requires device unlock
- Survives app reinstall (optional)
- Can be configured to require biometric authentication per access

**Android:** EncryptedSharedPreferences + Android Keystore
- react-native-keychain uses Android Keystore automatically
- Hardware-backed encryption keys
- Keys protected by device lock screen

**Best Practices:**
- Store only refresh tokens securely, not access tokens in memory
- Set token expiration and refresh mechanisms
- Clear tokens on logout, device lock, or timeout
- Consider requiring biometric re-authentication for sensitive operations

**Screenshot Prevention:**

**iOS:**
Use `UIApplication.shared.ignoreSnapshotOnNextApplicationLaunch()` or react-native-screenguard

**Android:**
Set `FLAG_SECURE` on window in MainActivity:
```
getWindow().setFlags(WindowManager.LayoutParams.FLAG_SECURE, WindowManager.LayoutParams.FLAG_SECURE);
```

Use react-native-screenguard or similar library that abstracts this.

**Considerations:**
- Apply to specific screens only (balance security vs UX)
- Show overlay when app enters background (prevent task switcher preview)
- Inform users why screenshots are disabled

**Preventing Data Logging:**

**Production Builds:**
- Strip all console.log calls (metro bundler configuration)
- Use logging library with environment-aware levels
- Ensure no sensitive data in crash reports

**Network Security:**
- Use certificate pinning to prevent MITM attacks
- Libraries: react-native-ssl-pinning
- Pin to specific certificates or public keys
- Implement fallback mechanism for certificate rotation

**Additional Security Measures:**

**Jailbreak/Root Detection:**
- Use react-native-device-info to detect compromised devices
- Warn user or restrict functionality
- Can be bypassed, but raises the bar

**Code Obfuscation:**
- ProGuard (Android) and minification
- Hermes bytecode compilation
- Note: Not foolproof, but makes reverse engineering harder

**Biometric Authentication:**
- Use react-native-biometrics or expo-local-authentication
- Require biometric confirmation before sensitive operations
- Fall back to PIN/password if biometric unavailable

**Session Management:**
- Implement idle timeout
- Lock screen after period of inactivity
- Require re-authentication

**Data Encryption at Rest:**
- Encrypt local database (SQLite with SQLCipher)
- Encrypt cached sensitive data
- Use react-native-encrypted-storage

**API Security:**
- All requests over HTTPS
- Token-based authentication (JWT)
- Request signing to prevent tampering
- Rate limiting on backend

**Secure Communication:**
- End-to-end encryption for truly sensitive data
- Don't rely solely on TLS

**Compliance:**
- Follow PCI-DSS if handling payment cards
- GDPR compliance for EU users
- Implement right to deletion, data export
- Audit logging for security events

---

## **8. Environment-Specific Configuration Management**

### **Build-Time Configuration Strategy:**

**The Right Approach: Environment Variables + Build Configurations**

**Never commit secrets to git.** Use a multi-layered approach:

**Layer 1: Environment Variables**

Use react-native-config or react-native-dotenv:
- Create `.env.development`, `.env.staging`, `.env.production`
- Add to `.gitignore`
- Library injects variables at build time
- Access via `Config.API_KEY` in code

**Layer 2: Build Configurations**

**iOS (Xcode Schemes):**
- Create separate schemes (Dev, Staging, Prod)
- Each scheme has different Build Configuration
- Use xcconfig files with variables
- Variables accessible in Info.plist and code

**Android (Build Variants):**
- Define product flavors in build.gradle (dev, staging, production)
- Each flavor can have different resources, manifests
- Environment variables injected via buildConfigField

**Layer 3: CI/CD Secret Management**

For CI/CD pipelines:
- Store secrets in CI/CD environment variables (GitHub Secrets, CircleCI, etc.)
- Inject into build at runtime
- Never expose in logs

**Distribution:**

Share sensitive environment files securely:
- Use 1Password, AWS Secrets Manager, or HashiCorp Vault
- Share via secure channels with team
- Document in README how to obtain

**App ID and Bundle Identifier:**

Use different bundle IDs per environment:
- `com.company.app.dev`
- `com.company.app.staging`
- `com.company.app` (production)

This allows all versions to coexist on device during testing.

**Firebase/Analytics:**

Use separate Firebase projects per environment:
- GoogleService-Info.plist (iOS) per scheme
- google-services.json (Android) per flavor
- Prevents test data polluting production analytics

**App Icons and Names:**

Visually differentiate environments:
- Different app icons (dev has badge/different color)
- Different app names (MyApp Dev, MyApp Staging, MyApp)
- Easier to identify which build is running

**Dynamic Configuration:**

For non-secret configuration that might change:
- Feature flags via remote config (Firebase Remote Config, LaunchDarkly)
- A/B testing configurations
- Kill switches for problematic features

**Best Practices:**

- Keep `.env` files simple (key-value pairs)
- Document all environment variables in README
- Provide `.env.example` template
- Validate required variables at app startup
- Fail fast if critical variables missing

---

## **9. Emergency Hotfix Deployment**

### **Multi-Channel Rollout Strategy:**

**Fastest Path: Over-The-Air (OTA) Updates**

**CodePush (Microsoft AppCenter):**
- Push JavaScript updates directly to users
- Bypasses app store review
- Can target specific versions or percentage of users
- Users get update on next app restart (or immediate)

**Rollout Plan:**

**Immediate (Minutes to Hours):**
1. Test fix in production-like environment
2. Create CodePush release targeting affected versions
3. Deploy to 10% canary group first
4. Monitor crash reports, user feedback
5. If stable, increase to 50%, then 100% over hours
6. All users updated without app store

**Limitations:**
- Only works for JavaScript changes
- Can't fix native module bugs
- Can't change Info.plist/AndroidManifest
- User must restart app to get update

**Fallback: Native App Store Updates**

If bug is in native code:

**iOS (Apple App Store):**
- Standard review: 24-48 hours
- Expedited review: Request for critical bugs, 2-12 hours
- Contact Apple Developer Support explaining criticality
- Prepare detailed justification (crashes, data loss, security)

**Android (Google Play):**
- Standard review: Usually 1-6 hours, can be up to 24-48 hours
- Staged rollout available: Deploy to percentage of users
- Can pause rollout if issues detected
- No expedited option, but generally faster than iOS

**Rollout Strategy:**

1. **Build and Test Fix:**
   - Create hotfix branch
   - Minimal changes (fix only, no features)
   - Test thoroughly on affected devices
   - Bump version number (patch version)

2. **Android First:**
   - Submit to Google Play immediately
   - Use staged rollout (10% → 50% → 100%)
   - Monitor crash reports (Firebase Crashlytics)
   - Can halt rollout if issues

3. **iOS Concurrent:**
   - Submit to App Store
   - Request expedited review with clear justification
   - Monitor TestFlight for early feedback
   - Prepare for questions from review team

4. **Communication:**
   - Alert affected users via push notification
   - Update status page if you have one
   - Provide workaround in help docs
   - Explain in release notes

**Staged Rollout Best Practices:**

- Start with 10% to catch any new issues
- Monitor for 12-24 hours at each stage
- Key metrics: crash rate, error logs, user feedback
- Have rollback plan (previous version via CodePush or store)

**Preventing Future Issues:**

After hotfix deployed:
- Post-mortem: What went wrong? How did it slip through?
- Improve testing: Add test case that would've caught it
- CI/CD improvements: Automated smoke tests on real devices
- Canary releases: Always deploy gradually, never 100% immediately

**Alternative: Server-Side Mitigation**

For some bugs, can mitigate server-side:
- Feature flags: Disable problematic feature remotely
- API version gating: Reject requests from buggy app versions
- Redirect to error message: "Please update your app"
- This buys time for proper fix

**Communication with Users:**

- In-app message: "Update available to fix issue"
- Push notification for critical fixes
- Email to registered users
- Social media updates
- Clear about what's fixed and why update is important

---

## **10. Simulator vs Real Device Differences**

### **Common Discrepancies:**

**Performance Differences:**
- **Simulator:** Uses Mac's CPU/GPU, much faster
- **Real Device:** ARM processor, less powerful (especially older devices)
- **Impact:** Animations smooth on simulator, janky on device; JavaScript execution faster, masking performance issues

**Memory Constraints:**
- **Simulator:** Mac's RAM (often 16GB+)
- **Real Device:** Limited (2-4GB on many devices)
- **Impact:** Memory leaks unnoticed on simulator; out-of-memory crashes on device

**Storage Differences:**
- **Simulator:** Mac's filesystem, case-insensitive (by default)
- **Real Device:** Case-sensitive filesystems
- **Impact:** `require('./Component')` works on simulator if file is `component.js`, crashes on device

**Native Module Differences:**
- **Simulator:** Some native APIs don't work (Camera, certain sensors)
- **Real Device:** Full hardware access
- **Impact:** Mock implementations work on simulator, fail on device

**Network Conditions:**
- **Simulator:** Mac's fast, stable WiFi
- **Real Device:** Variable cellular, WiFi, slow networks
- **Impact:** Race conditions, timeout issues not caught

**Debugging Setup:**
- **Simulator:** Remote debugging uses Chrome V8 engine
- **Real Device:** Hermes or JSC engine
- **Impact:** Different JS runtime behavior, polyfills

**Debugging Strategies:**

**1. Always Test on Real Devices:**
- Have test matrix: old Android, new Android, old iOS, new iOS
- Test on low-end devices to catch performance issues
- Test on actual network conditions (can throttle)

**2. Use Device Logs:**
- **iOS:** Xcode Console, filter by app
- **Android:** ADB logcat filtered by package name
- Look for native crashes, exceptions

**3. Crashlytics/Sentry:**
- Implement crash reporting in dev builds
- See exact stack traces from device
- Symbolicate crashes to readable format

**4. Memory Profiling:**
- **iOS:** Xcode Instruments (Allocations, Leaks)
- **Android:** Android Studio Profiler
- Monitor memory over time, look for growth

**5. Network Debugging:**
- Use Flipper Network plugin
- See actual requests/responses
- Test with network condition simulation

**6. Remote Debugging:**
- React Native Debugger or Flipper
- But remember: Debugging changes runtime (V8 vs Hermes)
- Disable for final testing

**7. Over-The-Air Testing:**
- Deploy test builds via TestFlight (iOS) or Firebase App Distribution
- Get logs from testers via Crashlytics
- Reproduce issues on actual devices

**Common Device-Specific Issues:**

**Storage Case-Sensitivity:**
- Import paths must match exact case
- Use linter to catch inconsistencies

**Permissions:**
- Simulator often grants all permissions
- Device requires user approval
- Test permission denial paths

**Notch/Safe Area:**
- Simulator previews, but test on actual devices
- Different safe area insets per model

**Performance:**
- Always profile on device, especially low-end
- Simulator masks FPS drops

---
