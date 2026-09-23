# watchOS workout sessions, mirroring and iPhone/Watch data transfer

Research for issue #5. Targets iOS 17 / watchOS 10 minimum. Terms follow `CONTEXT.md`: **Routine**, **Workout**, **Exercise**, **Set**. "HealthKit workout session" (`HKWorkoutSession`) is Apple's API object. It is not the same thing as our **Workout**, although one **Workout** usually runs inside one session.

Sources were read on 2026-09-23. Anything marked **(unverified)** has no first-party documentation behind it.

## Answer

- **Standalone on the Watch works offline.** Create an `HKWorkoutSession` with `.traditionalStrengthTraining`, attach an `HKLiveWorkoutDataSource` to its `associatedWorkoutBuilder()`, call `startActivity` and `beginCollection`, and at the end call `stopActivity` → `endCollection` → `finishWorkout` → `end`. An active session keeps the app running in the background. It needs the *Workout processing* background mode, and watchOS can suspend the app if it uses too much CPU. The finished `HKWorkout` syncs to the iPhone's Health store by itself. Our own data (Sets, Exercises, Routine link) does not. We have to sync it ourselves.
- **Mirroring (iOS 17 / watchOS 10).** The Watch always owns the **primary** session. It calls `startMirroringToCompanionDevice()`, and the system launches the iPhone app in the background (if it isn't running) and passes a **mirrored** session to `workoutSessionMirroringStartHandler`. Both sides can pause, resume and `stopActivity`, and state stays in sync. Only the Watch has the live builder and saves the HealthKit workout. `sendToRemoteWorkoutSession(data:)` sends custom `Data` in either direction; we would use it for live Set logging. When the iPhone goes out of range, the iPhone gets `didDisconnectFromRemoteDeviceWithError` and its session object becomes invalid. The Watch retries automatically, and on reconnect the iPhone handler fires again with a **new** session instance. On iOS 17–18 the iPhone cannot own a session. iOS 26 adds iPhone-owned workout sessions, but Apple still recommends starting on the Watch and mirroring back.
- **Starting from the iPhone:** `HKHealthStore.startWatchApp(toHandle:)` launches or wakes the Watch app and delivers only an `HKWorkoutConfiguration` to `WKApplicationDelegate.handle(_:)`. The configuration has no field for a Routine ID. We have to get the Routine to the Watch another way: sync it beforehand, or send it over the mirroring channel once that is connected.
- **Crash recovery:** watchOS relaunches the app and calls `handleActiveWorkoutRecovery()`. In it, call `recoverActiveWorkoutSession`, then set up the data source and delegates again. The same API exists on iPhone only from iOS 26.
- **Syncing Routines and completed Workouts:** use **WatchConnectivity** as the fast local path. `updateApplicationContext` sends "latest state" (a good fit for the Routine list), `transferUserInfo` gives queued, in-order, guaranteed delivery (a good fit for completed Workouts or Set deltas), `transferFile` handles large payloads, and `sendMessage` only works when the counterpart is reachable. Use **CloudKit** (SwiftData or `NSPersistentCloudKitContainer`, both available on watchOS) as the durable, independent path. Apple says an independent Watch app "can't use Watch Connectivity as its main source of data". Watch networking is limited to `URLSession`-level APIs.
- **Pitfalls:** mirroring and most WatchConnectivity transfers **do not work in the Simulator**. `sendToRemoteWorkoutSession` has been reported to hang for about 5% of Workouts on watchOS 10/11 (developer forum, no Apple fix). WatchConnectivity payload size limits are not documented. **watchOS 11/26 changed little for this problem.** watchOS 11 shows iPhone Live Activities in the Watch Smart Stack. watchOS 26 adds workout-app suggestions in the Smart Stack and arm64 builds. iOS 26 adds iPhone workout sessions and recovery.

---

## 1. Standalone Workout on Apple Watch

### Lifecycle
1. **Authorise.** Request share permission for `HKQuantityType.workoutType()` plus the types to read. On watchOS 6+ the user authorises on the Watch, so the Watch target needs `NSHealthShareUsageDescription` / `NSHealthUpdateUsageDescription`. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
2. **Configure.** `HKWorkoutConfiguration` with `activityType = .traditionalStrengthTraining` ("strength training exercises primarily using machines or free weights") or `.functionalStrengthTraining`. [traditionalStrengthTraining](https://developer.apple.com/documentation/healthkit/hkworkoutactivitytype/traditionalstrengthtraining). Use the same configuration for the session and for `HKLiveWorkoutDataSource`. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
3. **Create.** `HKWorkoutSession(healthStore:configuration:)` throws if the configuration is invalid. Then `builder = session.associatedWorkoutBuilder()`, `builder.dataSource = HKLiveWorkoutDataSource(...)`, and set both delegates. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
4. **Optional `prepare()`.** This moves the session to `.prepared`, for example during a countdown. The states are `notStarted, prepared, running, paused, stopped, ended`. [prepare()](https://developer.apple.com/documentation/healthkit/hkworkoutsession/prepare()), [HKWorkoutSessionState](https://developer.apple.com/documentation/healthkit/hkworkoutsessionstate)
5. **Start.** `session.startActivity(with:)` and `builder.beginCollection(withStart:)`. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
6. **During the Workout.** `HKLiveWorkoutBuilderDelegate.workoutBuilder(_:didCollectDataOf:)` delivers new samples (heart rate, energy). `builder.statistics(for:)` gives totals and averages. Custom samples and events go in through `add(_:)` and `addWorkoutEvents(_:)`. By default, session events are forwarded to the builder (`shouldCollectWorkoutEvents`). [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
   - HealthKit has no model for Exercises or Sets. Per-Set data (weight × reps) has to live in our own store. At most it can be summarised into `HKWorkout` metadata, or into activities through `beginNewActivity(configuration:date:metadata:)` (listed under [HKWorkoutSession topics](https://developer.apple.com/documentation/healthkit/hkworkoutsession)). *(Whether to map Exercises to workout activities is a design decision, not researched here.)*
7. **End.** Call `stopActivity(with:)`. When the delegate reports `.stopped`, call `endCollection(withEnd:)`, then `finishWorkout()` (which saves the `HKWorkout`), then `end()`. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
8. **One session at a time.** If another app starts a workout, ours receives `errorAnotherWorkoutSessionStarted` and the session ends. [HKWorkoutSession](https://developer.apple.com/documentation/healthkit/hkworkoutsession)

### Offline
Nothing in the lifecycle above needs the iPhone or a network. After the Watch saves a workout, "it syncs to my other devices" through HealthKit's own sync. [WWDC23 10023](https://developer.apple.com/videos/play/wwdc2023/10023/). That sync only carries HealthKit samples, not our Routine, Exercise or Set records.

### Background execution limits
- Add the **Workout processing** background mode, plus **Audio** if the app plays sounds or haptics. Background audio is only valid while a workout session runs. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
- With an active session the app keeps running through wrist-down and when the user switches apps. It keeps receiving sensor data and can alert with audio or haptics, and the watch face shows a workout icon. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
- "If your app uses an excessive amount of CPU while in the background, watchOS may suspend it." Profile with Instruments. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
- While the app is suspended, HealthKit caches incoming remote-session data and delivers it as an array when the app resumes. This is a comment in Apple's [Building a multidevice workout app](https://developer.apple.com/documentation/healthkit/building-a-multidevice-workout-app) sample, `WorkoutManager.swift`.
- Outside a workout, background refresh tasks are "strictly limited" in how often and how long they run, and are not guaranteed. [Keeping your watchOS app's content up to date](https://developer.apple.com/documentation/watchos-apps/keeping-your-watchos-app-s-content-up-to-date)

### Crash / relaunch recovery
- If the app crashes during a session, the system relaunches it and calls `WKApplicationDelegate.handleActiveWorkoutRecovery()` (watchOS 7+). Call `HKHealthStore.recoverActiveWorkoutSession(completion:)` there to get a **new** session object, then "you must access its builder and set up your data source and delegates again". [handleActiveWorkoutRecovery()](https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handleactiveworkoutrecovery()), [recoverActiveWorkoutSession](https://developer.apple.com/documentation/healthkit/hkhealthstore/recoveractiveworkoutsession(completion:))
- The session and builder come back in their previous state, but the live data source has to be set up again. [WWDC25 322](https://developer.apple.com/videos/play/wwdc2025/322/)
- HealthKit recovery covers only the HealthKit session. Our in-progress Workout (Sets logged so far) has to be saved to local storage as it goes, so we can rebuild the UI after a relaunch. *(design inference)*
- On iPhone, `recoverActiveWorkoutSession` is available only from **iOS 26** (via a scene-delegate option `shouldHandleActiveWorkoutRecovery`). [recoverActiveWorkoutSession availability](https://developer.apple.com/documentation/healthkit/hkhealthstore/recoveractiveworkoutsession(completion:)), [WWDC25 322](https://developer.apple.com/videos/play/wwdc2025/322/)

## 2. Mirroring a live Workout to the iPhone

### Setup and roles
- The Watch session is the **primary**; the iPhone copy is the **mirrored** session. [WWDC23 10023](https://developer.apple.com/videos/play/wwdc2023/10023/)
- The Watch calls `startMirroringToCompanionDevice()` (watchOS 10+, **watchOS only**). "If your iOS app isn't running, the system launches it in the background." The call fails on a session that has already ended. [startMirroringToCompanionDevice](https://developer.apple.com/documentation/healthkit/hkworkoutsession/startmirroringtocompaniondevice(completion:))
- The iPhone sets `HKHealthStore.workoutSessionMirroringStartHandler` (iOS 17+) "as soon as your app launches". The handler runs on an arbitrary background queue. Keep a strong reference to the session you receive. [workoutSessionMirroringStartHandler](https://developer.apple.com/documentation/healthkit/hkhealthstore/workoutsessionmirroringstarthandler), [WWDC23 10023](https://developer.apple.com/videos/play/wwdc2023/10023/)
- HealthKit "gives my app 10 seconds to start a live activity and call a handler to start mirroring" after a background launch. [WWDC23 10023](https://developer.apple.com/videos/play/wwdc2023/10023/)
- In Apple's sample, only the Watch has an `HKLiveWorkoutBuilder` ("only available on watchOS" in the sample's comments). The iPhone shows metrics that the Watch sends as archived `HKStatistics` through the data channel. [Building a multidevice workout app](https://developer.apple.com/documentation/healthkit/building-a-multidevice-workout-app) (sample source)

### Who can start, pause and end
- State is kept in sync both ways. When the Watch pauses, the mirrored session pauses. When the iPhone resumes the mirrored session, the Watch's primary delegate sees the state change. [WWDC23 10023](https://developer.apple.com/videos/play/wwdc2023/10023/)
- In the sample, the iPhone UI calls `session.pause()` / `resume()` and `stopActivity(with:)` on the **mirrored** session. The Watch sees `.stopped`, calls `endCollection` / `finishWorkout`, then calls `end()` on the primary session. So the iPhone can effectively end a Workout, but the **Watch saves it**. [sample source](https://developer.apple.com/documentation/healthkit/building-a-multidevice-workout-app)
- Only the Watch can start mirroring (the API is watchOS-only). The iPhone can start a Workout only indirectly, through `startWatchApp` (§3).
- `stopMirroringToCompanionDevice()` (Watch) stops mirroring while the Watch session keeps running. The iPhone then gets `didDisconnectFromRemoteDeviceWithError`. [stopMirroringToCompanionDevice](https://developer.apple.com/documentation/healthkit/hkworkoutsession/stopmirroringtocompaniondevice(completion:))

### Custom data channel
- `sendToRemoteWorkoutSession(data:)` (iOS 17 / watchOS 10) can be sent "from either the mirrored or primary session". The receiver gets `workoutSession(_:didReceiveDataFromRemoteWorkoutSession: [Data])`. [sendToRemoteWorkoutSession](https://developer.apple.com/documentation/healthkit/hkworkoutsession/sendtoremoteworkoutsession(data:completion:)), [HKWorkoutSessionDelegate](https://developer.apple.com/documentation/healthkit/hkworkoutsessiondelegate)
- The payload is opaque `Data`; Apple's samples use `NSKeyedArchiver` or `JSONEncoder`. We would use it for "Set logged", "Set edited" and "rest timer" messages. The sample also sends elapsed time on every state change. [sample source](https://developer.apple.com/documentation/healthkit/building-a-multidevice-workout-app)
- **No size limit, ordering guarantee or delivery guarantee is documented** (unverified). Treat it as best-effort, keep messages small and idempotent (for example, each Set carries a stable ID), and reconcile afterwards over WatchConnectivity or CloudKit.

### Out of range and back
- When the connection drops, the mirrored session's delegate gets `didDisconnectFromRemoteDeviceWithError`, and "the provided workout session is no longer valid". [didDisconnectFromRemoteDeviceWithError](https://developer.apple.com/documentation/healthkit/hkworkoutsessiondelegate/workoutsession(_:diddisconnectfromremotedevicewitherror:))
- "If the primary workout session is still running, it automatically tries to reconnect. If successful, the companion iOS device calls the `workoutSessionMirroringStartHandler` block again, passing in a new, valid `HKWorkoutSession` instance." [same](https://developer.apple.com/documentation/healthkit/hkworkoutsessiondelegate/workoutsession(_:diddisconnectfromremotedevicewitherror:)), [workoutSessionMirroringStartHandler](https://developer.apple.com/documentation/healthkit/hkhealthstore/workoutsessionmirroringstarthandler) ("Your app may receive multiple calls… Each call has its own HKWorkoutSession instance.")
- Consequences for us: the Watch keeps recording Sets while disconnected. On reconnect the iPhone must attach to the new session and ask for a catch-up, for example by requesting the full current Workout state over the data channel. Whether Watch→iPhone messages sent while disconnected are queued or dropped is **not documented (unverified)**, so design for dropped messages.

## 3. Starting a Workout on the Watch from the iPhone
- `HKHealthStore.startWatchApp(toHandle: HKWorkoutConfiguration)` "launches or wakes the companion watchOS app". The Watch gets `WKApplicationDelegate.handle(_ workoutConfiguration:)`, creates the session, starts mirroring and starts the activity. [startWatchApp](https://developer.apple.com/documentation/healthkit/hkhealthstore/startwatchapp(with:completion:)), [handle(_:)](https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handle(_:)-1pfoc)
- Only the `HKWorkoutConfiguration` is passed, not arbitrary data. To start "Routine X", the Routine must already be on the Watch (synced through §5), with its ID sent over `sendToRemoteWorkoutSession` once mirroring connects (or through `sendMessage`). *(design inference from the API shape)*
- WWDC25 recommendation, even with iOS 26 iPhone sessions: "If you have a Watch app, be sure to start the workout there to get all available metrics. Just call Start Watch App… and once you have, be sure to mirror the workout to the iPhone." [WWDC25 322](https://developer.apple.com/videos/play/wwdc2025/322/)

## 4. iPhone-side workout sessions (iOS 26)
- `HKWorkoutSession(healthStore:configuration:)` on iOS/iPadOS is available only from **26.0**. On iOS 17–18, an iPhone app gets an `HKWorkoutSession` only as a mirrored session. [init availability](https://developer.apple.com/documentation/healthkit/hkworkoutsession/init(healthstore:configuration:))
- On iPhone there is no heart-rate sensor (a BLE heart-rate monitor is needed), the phone usually locks (the system prompts once so workout data stays readable while locked), Live Activities and Siri work on the Lock Screen, and crash recovery works through the scene delegate. [WWDC25 322](https://developer.apple.com/videos/play/wwdc2025/322/), [HKWorkoutSession overview](https://developer.apple.com/documentation/healthkit/hkworkoutsession)
- With iOS 17 as the minimum, a phone-only Workout (no Watch) cannot be a HealthKit workout session before iOS 26. It would just be app state, and at most an `HKWorkout` saved afterwards. *(inference)*

## 5. Syncing Routines and completed Workouts between devices

### WatchConnectivity (`WCSession`)
Both apps must activate a session. The iOS app must implement the async-activation and deactivation delegate methods to support switching between Watches, and should check `isPaired` / `isWatchAppInstalled`. [WCSession](https://developer.apple.com/documentation/watchconnectivity/wcsession)

| API | Semantics | Reachability needed | Notes |
|---|---|---|---|
| `updateApplicationContext(_:)` | Latest-state dictionary; **replaces** the previous one; delivered "when the opportunity arises", ready by the time the counterpart wakes | No | Good for "current Routine list / version". [doc](https://developer.apple.com/documentation/watchconnectivity/wcsession/updateapplicationcontext(_:)) |
| `transferUserInfo(_:)` | Queued FIFO, "ensure that it's delivered", continues if the sender is suspended | No | Good for completed Workouts and Set deltas. **Not supported in the Simulator.** [doc](https://developer.apple.com/documentation/watchconnectivity/wcsession/transferuserinfo(_:)) |
| `transferFile(_:metadata:)` | Background file transfer; may be throttled; `outstandingFileTransfers` | No | For large payloads (for example an exported Workout history). **Not supported in the Simulator.** [doc](https://developer.apple.com/documentation/watchconnectivity/wcsession/transferfile(_:metadata:)) |
| `sendMessage` / `sendMessageData` | Immediate, high priority, optional reply; the error handler fires if unreachable | **Yes** (`isReachable`) | From the Watch it wakes the iOS app in the background; from iOS it does **not** wake the Watch app. [doc](https://developer.apple.com/documentation/watchconnectivity/wcsession/sendmessage(_:replyhandler:errorhandler:)) |
| `transferCurrentComplicationUserInfo` | High priority, for complications only | No | 50 per day. [sample](https://developer.apple.com/documentation/watchconnectivity/transferring-data-with-watch-connectivity) |

- `isReachable` on the Watch is true only when the iPhone is in range **and** the Watch app is in the foreground or high-priority background, **for example during a workout session**. [isReachable](https://developer.apple.com/documentation/watchconnectivity/wcsession/isreachable)
- Background transfers "are not delivered immediately… the system may delay transfers slightly to improve power usage". [WCSession](https://developer.apple.com/documentation/watchconnectivity/wcsession)
- The Watch app must complete every `WKWatchConnectivityRefreshBackgroundTask`, or it uses up its budget and crashes. [Transferring data with Watch Connectivity](https://developer.apple.com/documentation/watchconnectivity/transferring-data-with-watch-connectivity)
- **Size limits:** `WCError.payloadTooLarge` exists "for both data dictionaries and files", but Apple **does not publish the numbers**. [payloadTooLarge](https://developer.apple.com/documentation/watchconnectivity/wcerror/code/payloadtoolarge). Community reports put dictionary/message payloads at about 64 KB **(unverified)**, so use `transferFile` for anything large.
- Independent Watch apps "can't use Watch Connectivity as its main source of data". [Creating independent watchOS apps](https://developer.apple.com/documentation/watchos-apps/creating-independent-watchos-apps)

### CloudKit / SwiftData / Core Data + CloudKit on watchOS
- CloudKit is available on watchOS 3+; SwiftData on watchOS 10+; `NSPersistentCloudKitContainer` on watchOS 6+. [CloudKit](https://developer.apple.com/documentation/cloudkit), [SwiftData](https://developer.apple.com/documentation/swiftdata), [NSPersistentCloudKitContainer](https://developer.apple.com/documentation/coredata/nspersistentcloudkitcontainer)
- SwiftData sync uses `NSPersistentCloudKitContainer`. It needs the iCloud capability and the Remote-notifications background mode. It does **not** support `@Attribute(.unique)`, it needs every relationship to be optional, it has no `.deny` delete rule, and the schema is additive-only once promoted to production. [Syncing model data across a person's devices](https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices)
- On watchOS, CloudKit requests go through the paired iPhone, Wi-Fi or cellular. `CKSubscription` notifications work on watchOS 6+, which makes CloudKit "a potential replacement for Watch Connectivity for independent apps". [Keeping your watchOS app's content up to date](https://developer.apple.com/documentation/watchos-apps/keeping-your-watchos-app-s-content-up-to-date)
- CloudKit "provides minimal offline caching" and relies on the network. With SwiftData or Core Data the local store is the offline cache, and sync happens opportunistically. [CloudKit](https://developer.apple.com/documentation/cloudkit)
- How quickly CloudKit sync happens on watchOS (push delivery, background budget) is **not documented (unverified)**. Developers often report slow or late sync on the Watch. So CloudKit should not be the only path for "I just edited a Routine on my phone, now start it on my Watch".

### Independent Watch networking
- `WKRunsIndependentlyOfCompanionApp = YES` ("Supports Running Without iOS App Installation") lets the Watch app install and run without the iOS app. [key](https://developer.apple.com/documentation/bundleresources/information-property-list/wkrunsindependentlyofcompanionapp), [Creating independent watchOS apps](https://developer.apple.com/documentation/watchos-apps/creating-independent-watchos-apps)
- Use `URLSession`, with background sessions when the app may become inactive. [Keeping content up to date](https://developer.apple.com/documentation/watchos-apps/keeping-your-watchos-app-s-content-up-to-date)
- Low-level networking (Network framework, WebSockets, `URLSessionStreamTask`, BSD sockets) is **blocked** except for audio streaming, VoIP calls and tvOS device discovery. `NWConnection` stays `waiting(ENETDOWN)`. "The simulator always allows low-level networking", so test on a device. [TN3135](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchos)

### Suggested combination *(synthesis, not an Apple recommendation)*
1. The local store on each device (SwiftData) is the source of truth for that device.
2. **Routines**, iPhone→Watch: `updateApplicationContext` for the latest Routine set (or a version number), plus CloudKit as the durable backstop.
3. **Live Workout**: the mirroring data channel for real-time Set updates, carrying idempotent IDs.
4. **Completed Workout**, Watch→iPhone: `transferUserInfo` (guaranteed and queued) with the full Workout, plus CloudKit. Deduplicate by Workout UUID. For the HealthKit side, Apple suggests "sync identifiers and version numbers" (`HKMetadataKeySyncIdentifier`) to keep data consistent. [WWDC23 10023](https://developer.apple.com/videos/play/wwdc2023/10023/)

## 6. Known pitfalls and simulator limitations
- **Mirroring does not work in simulators.** Apple DTS: "as of today, workout mirroring is not supported in simulators". The error is HealthKit code 300 "Remote device is unreachable". [forum 765469](https://developer.apple.com/forums/thread/765469). The sample "needs to run on physical devices". [sample](https://developer.apple.com/documentation/healthkit/building-a-multidevice-workout-app)
- The Simulator does not support **`transferUserInfo`, `transferFile` or `transferCurrentComplicationUserInfo`**. [transferUserInfo](https://developer.apple.com/documentation/watchconnectivity/wcsession/transferuserinfo(_:)), [forum 127460](https://developer.apple.com/forums/thread/127460) (DTS)
- Running from Xcode stops the Watch app from suspending normally, which hides background-task bugs. Launch from the Home Screen and log to a file instead. [Transferring data with Watch Connectivity](https://developer.apple.com/documentation/watchconnectivity/transferring-data-with-watch-connectivity)
- `sendToRemoteWorkoutSession` reportedly **never returns** in about 5% of Workouts, stays broken for the rest of that Workout, and affects watchOS 10/11 and iOS 17/18. Apple has not replied, and the only reported workaround is restarting both devices. [forum 769355](https://developer.apple.com/forums/thread/769355) *(developer report, unverified by Apple)*. Mitigation: add timeouts and never block UI on the send.
- `workoutSessionMirroringStartHandler` can fire several times. Replace old session references and never reuse a disconnected session. [doc](https://developer.apple.com/documentation/healthkit/hkhealthstore/workoutsessionmirroringstarthandler)
- Before mirroring existed, Apple said that if the iPhone tries to end a Watch-started workout, the iPhone app should tell the user to end it on the Watch, or else invalid data may be saved. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions). Mirroring's `stopActivity` on the mirrored session is the modern replacement.
- HealthKit delegate callbacks arrive on an anonymous serial background queue. Hop to the main actor. [HKWorkoutSessionDelegate](https://developer.apple.com/documentation/healthkit/hkworkoutsessiondelegate)
- Watch-switching: if the iOS delegate does not implement the activation and deactivation methods, the app opts out of multi-Watch support and is terminated after a switch. [WCSession](https://developer.apple.com/documentation/watchconnectivity/wcsession)

## 7. What changed in watchOS 11 and watchOS 26 / iOS 26
- **watchOS 11 (2024):** iPhone Live Activities appear in the Watch Smart Stack automatically (`.supplementalActivityFamilies([.small])` gives a custom view). WorkoutKit custom-workout improvements (step `displayName`, pool swimming) apply to Apple's Workout app, not to our sessions. Double-tap `handGestureShortcut(.primaryAction)` is useful for "log Set". [WWDC24 10205](https://developer.apple.com/videos/play/wwdc2024/10205/). No changes to the mirroring APIs were found (unverified absence).
  - Possible pitfall: while mirroring, the iPhone's Live Activity could also appear in the Watch Smart Stack next to our running Watch app. Behaviour not verified.
- **watchOS 26 (2025):** Watch workout apps can be suggested in the Smart Stack if they use the correct `HKWorkoutActivityType` and accurate start and end times. arm64 on Series 9+ / Ultra 2 means building with "Standard Architectures". Controls and RelevanceKit also arrived. [WWDC25 334](https://developer.apple.com/videos/play/wwdc2025/334/)
- **iOS/iPadOS 26:** `HKWorkoutSession` and `HKLiveWorkoutBuilder` on iPhone and iPad, `recoverActiveWorkoutSession` on iOS, Lock Screen Siri control, and locked-device data access. [WWDC25 322](https://developer.apple.com/videos/play/wwdc2025/322/), [init availability](https://developer.apple.com/documentation/healthkit/hkworkoutsession/init(healthstore:configuration:))
- No WatchConnectivity changes were found in the watchOS 11 or 26 "what's new" sessions (unverified absence).

## Sources
- Running workout sessions: https://developer.apple.com/documentation/healthkit/running-workout-sessions
- HKWorkoutSession: https://developer.apple.com/documentation/healthkit/hkworkoutsession
- startMirroringToCompanionDevice: https://developer.apple.com/documentation/healthkit/hkworkoutsession/startmirroringtocompaniondevice(completion:)
- stopMirroringToCompanionDevice: https://developer.apple.com/documentation/healthkit/hkworkoutsession/stopmirroringtocompaniondevice(completion:)
- sendToRemoteWorkoutSession: https://developer.apple.com/documentation/healthkit/hkworkoutsession/sendtoremoteworkoutsession(data:completion:)
- workoutSessionMirroringStartHandler: https://developer.apple.com/documentation/healthkit/hkhealthstore/workoutsessionmirroringstarthandler
- didDisconnectFromRemoteDeviceWithError: https://developer.apple.com/documentation/healthkit/hkworkoutsessiondelegate/workoutsession(_:diddisconnectfromremotedevicewitherror:)
- startWatchApp: https://developer.apple.com/documentation/healthkit/hkhealthstore/startwatchapp(with:completion:)
- WKApplicationDelegate handle(_:): https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handle(_:)-1pfoc
- recoverActiveWorkoutSession: https://developer.apple.com/documentation/healthkit/hkhealthstore/recoveractiveworkoutsession(completion:)
- handleActiveWorkoutRecovery: https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handleactiveworkoutrecovery()
- Sample: Building a multidevice workout app: https://developer.apple.com/documentation/healthkit/building-a-multidevice-workout-app
- WWDC23 10023 Build a multi-device workout app: https://developer.apple.com/videos/play/wwdc2023/10023/
- WWDC25 322 Track workouts with HealthKit on iOS and iPadOS: https://developer.apple.com/videos/play/wwdc2025/322/
- WWDC24 10205 What's new in watchOS 11: https://developer.apple.com/videos/play/wwdc2024/10205/
- WWDC25 334 What's new in watchOS 26: https://developer.apple.com/videos/play/wwdc2025/334/
- WCSession: https://developer.apple.com/documentation/watchconnectivity/wcsession
- Transferring data with Watch Connectivity: https://developer.apple.com/documentation/watchconnectivity/transferring-data-with-watch-connectivity
- WCError payloadTooLarge: https://developer.apple.com/documentation/watchconnectivity/wcerror/code/payloadtoolarge
- Creating independent watchOS apps: https://developer.apple.com/documentation/watchos-apps/creating-independent-watchos-apps
- Keeping your watchOS app's content up to date: https://developer.apple.com/documentation/watchos-apps/keeping-your-watchos-app-s-content-up-to-date
- TN3135 Low-level networking on watchOS: https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchos
- SwiftData sync: https://developer.apple.com/documentation/swiftdata/syncing-model-data-across-a-persons-devices
- NSPersistentCloudKitContainer: https://developer.apple.com/documentation/coredata/nspersistentcloudkitcontainer
- Forums: https://developer.apple.com/forums/thread/765469 , https://developer.apple.com/forums/thread/769355 , https://developer.apple.com/forums/thread/127460
