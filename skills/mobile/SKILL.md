---
name: mobile
description: "Apply when building mobile applications: lifecycle management, navigation, offline support, performance, accessibility, and platform conventions. Trigger on mentions of iOS, Android, React Native, Flutter, mobile app, SwiftUI, Jetpack Compose, or mobile UI. Platform-specific idioms are in references/."
---

# Mobile Development Skill

Mobile development operates under constraints that fundamentally differ from server or desktop programming. The device has limited CPU, memory, and battery. The network connection is unreliable. The operating system aggressively manages resources and will kill your app without warning. Users interact through touch on small screens, expect instant responsiveness, and assume the app will remember exactly where they left off. Every architectural decision must account for these realities.

This skill covers the cross-platform principles that apply regardless of whether you are building with Swift, Kotlin, Dart, or TypeScript. The patterns here draw from official platform guidance (Apple Human Interface Guidelines, Google Material Design, Google Guide to App Architecture) and from established references in mobile engineering (Big Nerd Ranch guides for both iOS and Android, Theresa Neil's Mobile Design Pattern Gallery, and WCAG 2.1 mobile accessibility standards). Platform-specific idioms, APIs, and tooling details live in the reference files below.

Building a quality mobile application means internalizing that you are a guest on the user's device. You share CPU time with every other app. You share battery with the user's entire day. You share screen space with notifications, system UI, and the user's attention span. Respect these constraints and the result is an app that feels native, fast, and trustworthy.

> **Platform-specific references:**
>
> - `references/ios.md` — iOS idioms (Swift, UIKit, SwiftUI)
> - `references/android.md` — Android idioms (Kotlin, Jetpack Compose, Android Architecture)
> - `references/react-native.md` — React Native idioms (TypeScript, navigation, native modules)
> - `references/flutter.md` — Flutter idioms (Dart, widgets, state management)

---

## 1. **Mobile-First Constraints**

Mobile devices are not small desktops. Both the iOS and Android Big Nerd Ranch guides open with this premise: you are working with constrained CPU, limited memory, finite battery, and an unreliable network. Apple's Human Interface Guidelines and Google's Material Design guidelines reinforce this from the design side — design for the device the user is holding, not for the workstation where you write code.

**Battery** is a shared resource the user monitors consciously. Minimize background work. Batch network requests rather than making many small calls. Avoid polling; use push notifications or platform-specific background fetch APIs instead. Every wake-up of the radio or CPU drains battery, and users will uninstall apps they identify as battery hogs.

**Memory** on mobile is aggressively managed by the operating system. Both iOS and Android will kill background apps to reclaim memory, and your app will not receive a warning when this happens. The Big Nerd Ranch Android guide emphasizes that your process will be killed — not might be, will be. Design every screen so that its state can be saved and restored from scratch. Never assume your app has been running continuously.

**Network** connections are unreliable by nature. Users move between Wi-Fi and cellular, pass through tunnels, board airplanes, and enter buildings with poor reception. Design for airplane mode, for slow connections, and for requests that fail mid-flight. The app must remain usable and must never show a blank screen simply because the network is unavailable.

**Screen size** varies enormously across the mobile landscape, from compact phones to large tablets and foldables. Content must adapt fluidly. Apple HIG prescribes adaptive layouts using size classes. Material Design defines responsive layout grids. Do not design a single fixed layout and hope it scales — design a system that responds to the available space.

---

## 2. **Application Lifecycle**

The application lifecycle is the single most important concept that distinguishes mobile from other platforms. The Big Nerd Ranch Android guide dedicates significant attention to the Android lifecycle because it is genuinely complex: an Activity or Fragment can be destroyed and recreated at any time due to configuration changes (screen rotation, locale change, dark mode toggle) or process death. The Big Nerd Ranch iOS guide covers the iOS app states — not running, inactive, active, background, and suspended — and the transitions between them that your app must handle gracefully.

Google's Guide to App Architecture makes the architectural response clear: separate UI state from business logic. ViewModels survive configuration changes. Saved state handles process death. The UI layer should be a stateless function of the current state, not a stateful object that accumulates data over its lifetime. When the system destroys and recreates your Activity or ViewController, the user must see exactly the same screen they left.

The user's mental model is simple: they switched to another app and came back. They expect to find their partially typed form, their scroll position, their selected filters — everything intact. If your app loses state on recreation, the user perceives it as a crash. Treat lifecycle management not as a platform quirk to work around but as a core architectural constraint that shapes how you structure every component.

Save early and save often. Persist critical state to disk, not just to in-memory objects. Use platform-provided mechanisms (onSaveInstanceState on Android, state restoration on iOS) for transient UI state, and use persistent storage for anything the user would be upset to lose.

---

## 3. **Navigation and Routing**

Theresa Neil's Mobile Design Pattern Gallery catalogs the standard mobile navigation patterns: stack-based navigation (push and pop), tab bars, navigation drawers, and modals. These are not arbitrary choices — they map to deeply ingrained user expectations. Apple's HIG prescribes standard navigation controllers and warns against inventing novel navigation schemes. Material Design defines predictable back behavior and surface hierarchy. Users must always know where they are, how they got there, and how to get back.

**Deep linking** means every meaningful screen in your app should be reachable via a URL. This is not optional — it enables notifications, sharing, marketing campaigns, and cross-app integration. Both Apple (Universal Links) and Google (App Links) provide official mechanisms. Design your navigation architecture around routes from the start; retrofitting deep links onto a navigation system that was not designed for them is painful.

**State restoration** across navigation is essential. When a user navigates back to a previous screen, they expect to find their scroll position, form inputs, and selections intact. Popping a screen off the navigation stack should not discard the state of the screens below it. This requires deliberate architectural choices about where state lives and how long it persists.

**Avoid deep nesting.** Users lose their sense of position beyond three or four levels of navigation depth. If your navigation hierarchy is deeply nested, consider flattening it with tabs, modals, or search. Neil's pattern gallery provides alternatives for every common deep-nesting scenario. The back button is not a substitute for clear information architecture.

---

## 4. **Offline-First and Data Persistence**

Both Big Nerd Ranch guides establish a foundational truth: the network is not guaranteed. An offline-first architecture treats the local database as the single source of truth and treats the network as a synchronization mechanism. The UI reads from the local store. The network layer writes to the local store. The UI never reads directly from the network.

**Cache aggressively.** Show stale data while refreshing in the background. A screen that displays cached content instantly and then updates when fresh data arrives feels fast and reliable. A screen that shows a loading spinner while waiting for the network feels slow and fragile. Google's App Architecture guide recommends Room (or SQLite) for structured data and DataStore for key-value preferences.

**Queue mutations for sync.** When the user creates, updates, or deletes data while offline, queue those operations and execute them when connectivity returns. This requires careful consideration of conflict resolution. Last-write-wins is the simplest strategy and is appropriate for many cases. When it is not, you need explicit merge logic — but do not over-engineer this until you have evidence that conflicts actually occur in your use case.

**Sync strategies** must be chosen deliberately. Push-based sync (server notifies the client) is more battery-efficient than pull-based polling. Incremental sync (only transmit changes since the last sync) is more bandwidth-efficient than full sync. Choose based on your data characteristics — how often it changes, how large it is, and how critical freshness is.

The goal is simple: the app works without the network. When the network is available, the app is better. When the network disappears, the app continues working.

---

## 5. **Networking and API Consumption**

The Big Nerd Ranch guides for both platforms are unambiguous: never block the main thread on a network call. Network requests are inherently slow and unreliable. Performing them on the main thread freezes the UI, and the operating system will kill your app if the main thread is unresponsive for too long (the watchdog timer on iOS, ANR on Android).

Google's Guide to App Architecture prescribes the repository pattern: an abstraction layer that provides data to the UI without exposing whether it came from the network, a local database, or an in-memory cache. The UI requests data from the repository. The repository decides where to get it. This separation makes offline support, caching, and testing straightforward.

**Pagination** is essential for any list that could grow large. Load data in pages using cursor-based pagination rather than offset-based, as offset pagination breaks when items are inserted or deleted during loading. Implement infinite scroll that fetches the next page as the user approaches the bottom of the list.

**Retry with exponential backoff** should be the default behavior for failed network requests. Transient failures are common on mobile networks. Retrying immediately floods the server and drains battery. Back off exponentially with jitter to spread retry attempts across time.

**Request deduplication** prevents multiple identical requests from being issued simultaneously. When three UI components all request the same data, the repository should make one network call and deliver the result to all three callers. This is especially important during screen transitions and configuration changes.

**Image loading and caching** deserves dedicated attention because images dominate mobile network traffic and memory usage. Resize images to the display size before rendering. Use appropriate formats (WebP, HEIF) for compression. Cache aggressively at both the disk and memory levels. Never load a full-resolution image into a thumbnail view.

---

## 6. **Performance and Responsiveness**

Apple's Human Interface Guidelines set the expectation: 60 frames per second is the baseline for smooth interaction, and 120fps on ProMotion displays. Material Design specifies that transitions and animations must feel fluid and natural. The Big Nerd Ranch guides translate this into engineering practice: never do heavy work on the main thread. UI jank — dropped frames during scrolling, delayed response to taps, stuttering animations — is the most immediately visible performance problem in any mobile app.

**Lazy loading** means you do not load what is not visible. Images below the fold, data for tabs the user has not opened, and content behind expandable sections should all be loaded on demand. Eager loading wastes bandwidth, memory, and startup time.

**List virtualization** is mandatory for any list longer than a few dozen items. Both iOS (UITableView, UICollectionView) and Android (RecyclerView) provide view recycling out of the box. The system creates only enough views to fill the visible area and recycles them as the user scrolls. Never create a view for every item in a large dataset.

**Startup time** must be treated as a first-class metric. Users expect cold start to complete in under two seconds. Defer everything that is not needed for the initial screen. Load data asynchronously. Initialize heavy services lazily. Measure startup time in your CI pipeline and set a budget that triggers alerts when exceeded.

**Memory leaks** are insidious on mobile because the operating system kills apps that consume too much memory. The most common source is strong reference cycles: closures capturing self, callbacks holding references to destroyed Activities, observers that are never removed. Use weak references where appropriate, cancel subscriptions when the lifecycle ends, and profile memory usage regularly.

**Image optimization** extends beyond network efficiency into rendering performance. Decode images off the main thread. Downsample large images to the size they will actually be displayed. Use the platform's recommended image loading libraries, which handle these concerns automatically.

---

## 7. **Platform Conventions and Accessibility**

Apple's Human Interface Guidelines and Google's Material Design guidelines exist because users build muscle memory around platform conventions. iOS users expect swipe-to-go-back, pull-to-refresh, standard navigation bar placement, and specific gesture behaviors. Android users expect the system back button to work predictably, the navigation drawer to follow Material patterns, and the share sheet to behave consistently. Violating these conventions does not make your app feel unique — it makes your app feel broken.

WCAG 2.1 Mobile Accessibility guidelines, together with platform-specific guidance, establish non-negotiable requirements. VoiceOver on iOS and TalkBack on Android must be able to navigate every screen in your app. Every interactive element needs an accessibility label. Every image that conveys meaning needs a description. Screen reader users navigate sequentially — your layout must make sense in a linear reading order, not just visually.

**Dynamic type and font scaling** are required, not optional. Both iOS and Android allow users to set their preferred text size in system settings. Your app must respect these settings. Text that does not scale with system preferences is inaccessible to users with low vision. Design layouts that accommodate text at 200% of the default size without breaking.

**Touch targets** must meet minimum size requirements: 44x44 points on iOS (per Apple HIG) and 48x48 density-independent pixels on Android (per Material Design). Small touch targets frustrate all users and make the app unusable for users with motor impairments. Padding around interactive elements counts toward the touch target even if the visual element is smaller.

**Color contrast** must meet WCAG AA minimums — 4.5:1 for normal text, 3:1 for large text. Do not use color as the sole indicator of state or meaning. Users with color vision deficiency need additional cues such as icons, patterns, or text labels.

**Dark mode** and system appearance settings must be supported. Both platforms provide APIs for detecting and responding to appearance changes. Hard-coded colors that look fine in light mode may be invisible in dark mode. Use semantic colors provided by the platform, which adapt automatically.

---

## 8. **App Distribution and Updates**

Mobile apps are versioned artifacts shipped through app stores, and this distribution model creates constraints that do not exist in web development. The Big Nerd Ranch guides address this reality: once a version is in users' hands, you cannot silently patch it. Users update on their own schedule, which means multiple versions of your app will be active simultaneously.

**Semantic versioning** communicates intent. Maintain backward-compatible API contracts with your backend — the server must support clients that are several versions behind. Never ship a backend change that breaks older app versions without a migration path. Version your API endpoints explicitly.

**Feature flags** enable gradual rollout and reduce risk. Ship new features behind flags that can be toggled remotely. Roll out to a small percentage of users first, monitor crash rates and metrics, and increase the rollout incrementally. Feature flags also enable instant rollback without requiring users to download an update.

**Forced updates** are necessary when a critical bug or a breaking API change makes an old version of the app dangerous or non-functional. Both platforms provide mechanisms to detect the current app version and prompt or require an update. Use forced updates sparingly — they disrupt the user — but do not hesitate when security or data integrity is at risk.

**Crash reporting** must be integrated from day one, before the first beta build. Tools like Firebase Crashlytics or Sentry provide stack traces, device information, and reproduction context that are impossible to obtain from user reports alone. Monitor crash-free rates as a primary quality metric. A crash-free rate below 99% demands immediate attention.

**Beta testing** pipelines reduce the blast radius of defects. Apple's TestFlight and Google Play Console's internal and closed testing tracks let you distribute pre-release builds to testers before public release. Establish a consistent beta cycle and never skip it under schedule pressure.

**App review compliance** is a hard gate on distribution. Both Apple's App Store Review Guidelines and Google Play's Developer Program Policies define rules that can result in rejection or removal. Familiarize yourself with these rules before building features that touch sensitive areas: in-app purchases, user data collection, health data, background location, notifications, and content moderation. A rejection late in a release cycle is costly.

---

## Applying This Skill

When reviewing or writing mobile application code, verify that each change respects mobile-first constraints: lifecycle awareness, offline resilience, main-thread safety, and platform conventions. Check that navigation is predictable and deep-linkable. Confirm that accessibility requirements are met — screen reader support, sufficient touch targets, dynamic type compliance. Evaluate network code for caching, retry logic, and deduplication. Ensure state survives both configuration changes and process death. Validate that the app respects battery and memory budgets. Confirm that the distribution strategy accounts for multiple active versions and includes crash reporting from the start.

Consult the platform-specific reference files for implementation idioms:

- `references/ios.md` for Swift, UIKit, and SwiftUI patterns
- `references/android.md` for Kotlin, Jetpack Compose, and Android Architecture Components
- `references/react-native.md` for TypeScript, React Navigation, and native module integration
- `references/flutter.md` for Dart, widget composition, and state management approaches
