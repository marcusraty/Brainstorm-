# Automatic Watch join for a Live Workout

Research for the "zero-thought Workouts" goal: the user never picks a device. A **Live** Workout exists on both devices, and the Watch should join by itself whenever it is worn and available, either at the start or partway through, and run the HealthKit workout session (`HKWorkoutSession`) there. Targets iOS 17 / watchOS 10 minimum. Terms follow `CONTEXT.md` (**Workout**, **Set**, **Routine**) plus **Live**, **Finished** and **Disconnected** as used in the brief.

This builds on [`watch-workout-and-sync.md`](watch-workout-and-sync.md), which already covers the session lifecycle, mirroring, `startWatchApp` basics and WatchConnectivity semantics. None of that is repeated here.

Sources were read on 2026-09-23 (watchOS 27 / iOS 27 are current). Claims marked **(unverified)** have no first-party source. Quotes from Apple SDK headers come from the iOS 26.0 SDK `HealthKit.framework/Headers`, read through the [xybp888/iOS-SDKs](https://github.com/xybp888/iOS-SDKs) mirror. The text is Apple's, but the mirror is not an Apple site.

## Answer

| # | Question | Verdict |
|---|---|---|
| 1 | Does `startWatchApp(toHandle:)` start a Watch session with no interaction on the Watch? | **Yes.** It is the one sanctioned way to do that. The Watch app is launched or woken in the background and may start the session there. Apple documents no wrist or lock preconditions and no rate limit. The completion can **hang when the Watch is off** (developer report), so add your own timeout. |
| 2 | Can the iPhone notice that the Watch has become available while our Watch app isn't running? | **No.** No WatchConnectivity callback reports "worn" or "in range" unless our Watch app is running. The only option is blind retries of `startWatchApp`, which are allowed (no documented limit) but only while the iPhone app itself is running. |
| 3 | Can the Watch learn in the background that a Workout is Live and start its own session? | **It can learn, but it cannot start.** A background WatchConnectivity wake works, but watchOS 10+ refuses to start or prepare a workout session from the background (`errorBackgroundWorkoutSessionNotAllowed`). From a background wake the Watch can only ask for **one tap**, via a local notification, a complication, or the Live Activity. |
| 4 | Can a Watch session end with an end date in the past? | **Allowed by the API.** `stopActivity(with:)` only requires the date to be on or after the start date. What happens to live samples collected after that date is **not documented**, so test it on a device. If the Watch never ran the Workout, Apple says not to create one retroactively. |
| 5 | Apple guidance and platform features for automatic starts | Apple's own rule: "If the user starts a workout in the iOS companion, and then opens your watchOS app, the watchOS app should **automatically start a workout session**." watchOS 11.1 lets a tap on the iPhone Live Activity in the Smart Stack open our Watch app. watchOS 26 can suggest workout apps in the Smart Stack. watchOS 27 adds nothing relevant. "Return to App" and "Auto-Launch" are user settings, not APIs. |

**Feasibility in one line:** fully automatic join works whenever the **iPhone app is running** at a moment when the Watch is reachable (`startWatchApp`). Otherwise the best available outcome is one tap on the Watch, and the Watch app joins by itself as soon as it opens.

---

## 1. `HKHealthStore.startWatchApp(toHandle:)`

### What it does
- "Launches or wakes the companion watchOS app to create a new workout session." After launching, "the system calls the handle method on the watchOS app's delegate and passes the provided workout configuration." [startWatchApp](https://developer.apple.com/documentation/healthkit/hkhealthstore/startwatchapp(with:completion:))
- On the Watch: "the system launches or wakes the corresponding Watch app **in the background** and calls this method. Use this method to configure an HKWorkoutSession object in your Watch app, and then call start." [WKApplicationDelegate.handle(_:)](https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handle(_:)-1pfoc)
- It was designed for no interaction on the Watch. WWDC16: "your watch will be in a workout state, **without the user having to intervene in its user interface** … If the watch application is not already running, it will be launched for you." [WWDC16 235 transcript](https://nonstrict.eu/wwdcindex/wwdc2016/235/). The Apple video page has been removed, so this is a third-party copy of Apple's transcript.
- Once the session is running, the app keeps running and "when the user raises their wrist, your app reappears". So the next glance at the Watch shows our Workout. [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
- Apple DTS confirms that this is the *only* sanctioned way for an iOS app to launch its Watch app, and only for real workout apps: "if your app isn't a workout app, you cannot use the mentioned API … your app risks a rejection in the App Review process." With the Workout processing background mode, the Watch app "will run in the background, until the workout session ends." [Forum 787130, DTS Engineer, Jun 2025](https://developer.apple.com/forums/thread/787130). An Apple Frameworks Engineer also said there is no general way to launch a Watch app from iOS, which is what Maps does. [Forum 734362](https://developer.apple.com/forums/thread/734362)

### Preconditions
Documented:
- A paired and **currently active** Apple Watch. The header says "on the currently active Apple Watch". [HKHealthStore.h, iOS 26 SDK](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKHealthStore.h)
- The Watch app is installed, and the iOS `WCSession` is activated. Apple's own recipe: "First, you need to check with WatchConnectivity if you have an activated session, and if you have a watch application installed." [WWDC16 235](https://nonstrict.eu/wwdcindex/wwdc2016/235/). Use `WCSession.isPaired` and `isWatchAppInstalled`. Both are reported from the iPhone's records, so they are true even when the Watch is off.
- The Watch target has the **Workout processing** background mode. [WWDC16 235](https://nonstrict.eu/wwdcindex/wwdc2016/235/), [forum 787130](https://developer.apple.com/forums/thread/787130)

Not documented **(unverified)**:
- Whether the Watch must be **on the wrist** or **unlocked**. No Apple document or session says either way, so test it on a device. Note that the session itself runs in the background, and a locked Watch still runs Apple's own workouts. The open question is whether the launch goes ahead.
- Whether the Watch must be in Bluetooth range, or whether Wi-Fi or cloud relay is enough.

### What the completion reports
- As documented: `success` is "true if the watch app launched successfully; otherwise, false", and `error` carries the reason. [startWatchApp](https://developer.apple.com/documentation/healthkit/hkhealthstore/startwatchapp(with:completion:)). No error codes specific to out-of-range or powered-off are documented. The `HKError.Code` list has nothing for it. [HKError.Code](https://developer.apple.com/documentation/healthkit/hkerror/code)
- In practice: "Power the watch off … The startWatchApp method's completion **never fires**." [Forum 688922](https://developer.apple.com/forums/thread/688922) *(developer report, no Apple reply)*. **Design for a completion that never arrives.** Race it against our own timeout, for example 10–15 s, and treat a timeout as "Watch unavailable, try again later".
- Do **not** follow that thread's suggestion to check `isReachable` first. On iOS, `isReachable` is true only when "the corresponding WatchKit extension **is running**" ([isReachable](https://developer.apple.com/documentation/watchconnectivity/wcsession/isreachable)). That is false in exactly the case `startWatchApp` exists to handle.
- Other reported failures: "Application … is installing or uninstalling, and cannot be launched" (`FBSOpenApplicationErrorDomain` 6) ([forum 52835](https://developer.apple.com/forums/thread/52835)). On iOS 14 / watchOS 7 the configuration sometimes arrived as a default (`.other`, unknown location) ([forum 661024](https://developer.apple.com/forums/thread/661024), [forum 662634](https://developer.apple.com/forums/thread/662634)). The Watch app should therefore rely on its own synced Live Workout state, not on the configuration's fields. *(Developer reports. It is not known whether this is fixed.)*

### Rate limits and repeat calls
- **No rate limit, and no guidance against calling it repeatedly, is documented** (absence, unverified). The only documented restriction is the App Review use-case rule above.
- Each successful call runs `handle(_:)` again, so the Watch app must be idempotent. If a session for this Live Workout already exists, ignore the call. *(design inference)*

## 2. Mid-Workout join: detecting that the Watch has become available

### What the iPhone can observe
| Signal (iOS) | Fires when | Tells us "Watch worn or in range"? |
|---|---|---|
| `sessionReachabilityDidChange` / `isReachable` | The counterpart's reachability changes. "A session is reachable when the iOS app or WatchKit extension to which it belongs is active and running." On iOS this also needs "the corresponding WatchKit extension is running." [delegate](https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate/sessionreachabilitydidchange(_:)), [isReachable](https://developer.apple.com/documentation/watchconnectivity/wcsession/isreachable) | **No.** It only fires once our Watch app is already running. |
| `sessionWatchStateDidChange` | `isPaired`, `isWatchAppInstalled`, `isComplicationEnabled` or `watchDirectoryURL` change. [doc](https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate/sessionwatchstatedidchange(_:)) | **No.** Those are install and pairing facts, not presence. |
| `activationDidComplete` / `sessionDidBecomeInactive` / `sessionDidDeactivate` | With Auto Switch, when the user "puts on a **different** Apple Watch". [WCSessionDelegate](https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate) | Only for switching Watches, not for the same Watch coming back. |
| `workoutSessionMirroringStartHandler` | The Watch started, or **re-established**, mirroring. The system launches the iOS app in the background if needed. [doc](https://developer.apple.com/documentation/healthkit/hkhealthstore/workoutsessionmirroringstarthandler) | Only once the Watch **already runs our session**. This already covers the "joined, went out of range, came back" case (see the earlier research, §2). |

**Conclusion:** no documented iOS API reports that the Watch was put on or came back into range while our Watch app is not running.

### Is retrying `startWatchApp` viable?
- It is **allowed**: no documented limit (§1). It is the only lever for a Watch that has never joined.
- It is **only possible while the iPhone app is running.** On iOS 17–18 the iPhone has no workout session of its own before mirroring, and the `workout-processing` background mode is listed for **watchOS only** ([Configuring background execution modes](https://developer.apple.com/documentation/xcode/configuring-background-execution-modes)). So a Live Workout on the iPhone does not keep our iPhone app running in the background, and a background polling timer is not a documented option. Whether an iOS 26 iPhone-owned `HKWorkoutSession` keeps the app running in the background is not stated in the docs **(unverified)**.
- A practical retry policy *(design inference)*: call it at the start, then again at natural moments when the iPhone app is in the foreground. Examples are each Set logged, the app returning to the foreground, and the rest-timer ending. Use a back-off such as at most once every 30–60 s, allow one call in flight, add the timeout from §1, and stop once a mirrored session arrives.
- **iOS 26 caveat:** if the iPhone runs its own primary `HKWorkoutSession` (iOS 26+) and the Watch then starts one and mirrors it back, Apple does not document how the two interact. "Another primary workout session has started or is already ongoing by this or another application" is the header text for `errorAnotherWorkoutSessionStarted` ([HKDefines.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKDefines.h)). **(unverified)** Keeping the Watch as the only session owner avoids the question.

## 3. The Watch learning about a Live Workout on its own

### Learning in the background works
- A `WKWatchConnectivityRefreshBackgroundTask` is triggered "whenever the paired device sends data" through `updateApplicationContext`, `transferUserInfo`, `transferCurrentComplicationUserInfo` or `transferFile`, and "the system launches your app in the background". [WKWatchConnectivityRefreshBackgroundTask](https://developer.apple.com/documentation/watchkit/wkwatchconnectivityrefreshbackgroundtask)
- Delivery is opportunistic: application context is sent "when the opportunity arises, with the goal of having the data ready to use by the time the counterpart wakes up" ([updateApplicationContext](https://developer.apple.com/documentation/watchconnectivity/wcsession/updateapplicationcontext(_:))), and "the system may delay transfers slightly to improve power usage" ([WCSession](https://developer.apple.com/documentation/watchconnectivity/wcsession)). How quickly this happens after the Watch comes back into range is not documented **(unverified)**.
- `transferCurrentComplicationUserInfo` is higher priority, but it needs our complication on the active face and is capped at 50 per day ([remainingComplicationUserInfoTransfers](https://developer.apple.com/documentation/watchconnectivity/wcsession/remainingcomplicationuserinfotransfers)). It is only legitimate if the complication really shows the Live Workout.

### Starting a session from the background is blocked
- `HKError.Code.errorBackgroundWorkoutSessionNotAllowed` (iOS 17 / watchOS 10+): "**A workout session is not allowed to start or prepare when this app is in the background.**" [HKDefines.h header comment](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKDefines.h). The [doc page](https://developer.apple.com/documentation/healthkit/hkerror/code/errorbackgroundworkoutsessionnotallowed) has no description.
- The one background exception is the `startWatchApp` → `handle(_:)` path, which Apple documents as a background launch that starts a session (§1).
- So a WatchConnectivity wake can **not** start the HealthKit session. It can:
  1. store the Live Workout (ID, start date, Routine) so that the next launch joins immediately;
  2. post a **local notification** ("Workout in progress, tap to track on Watch"). `UNUserNotificationCenter` is on watchOS 3+ ([doc](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)). The tap opens the app in the foreground, where starting is allowed;
  3. reload a complication or widget that shows the Live Workout.
- **A relay (Watch wakes → tells iPhone → iPhone calls `startWatchApp`) is not a documented path** *(unverified)*. From a background refresh the Watch's `isReachable` is expected to be false (it needs the foreground or "high priority in the background (for example, during a workout session…)", [isReachable](https://developer.apple.com/documentation/watchconnectivity/wcsession/isreachable)), so `sendMessage` fails. Background transfers from the Watch are delivered when the iOS app next wakes ("When the app wakes up, it is notified of any data that arrived while it was inactive", [WatchConnectivity](https://developer.apple.com/documentation/watchconnectivity)) and are not documented to launch it. It may be worth a quick device test, but don't depend on it.

### Starting on open is what Apple asks for
"If the user starts a workout in the iOS companion, and then opens your watchOS app, the watchOS app should **automatically start a workout session for the workout in progress**. … you should set the workout's `startDate` to the iOS workout's start. Also, if your app calculates its own calories, you can retroactively give credit for the calories burned before the workout session began." [Running workout sessions — Coordinate with the companion app](https://developer.apple.com/documentation/healthkit/running-workout-sessions)

So any foreground launch should join with no further tap: from the app icon, notification, complication, Smart Stack Live Activity (§5) or watchOS 26 suggestion. Call `beginCollection(at:)` with the Live Workout's start date. The builder's start date is simply "the start date of the workout", with no rule against a past date ([HKWorkoutBuilder.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKWorkoutBuilder.h)).

## 4. Ending with an end date in the past

- `stopActivity(with:)`: "The end date for the workout session. **This must be equal to or after the start date.**" No rule relates it to *now*. [stopActivity(with:)](https://developer.apple.com/documentation/healthkit/hkworkoutsession/stopactivity(with:))
- `endCollection(withEnd:)`: "Stops the collection of data, sets the workout's end date, and deactivates the workout builder." It takes any `Date`. [endCollection](https://developer.apple.com/documentation/healthkit/hkworkoutbuilder/endcollection(withend:completion:))
- The builder class explicitly supports past workouts: "Samples, events, and metadata may be added to a builder either during a live workout session or to create a workout that occurred in the past." [HKWorkoutBuilder.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKWorkoutBuilder.h)
- **Not documented (unverified):** whether `HKLiveWorkoutBuilder` drops, trims or keeps live samples (heart rate, energy) timestamped after the past end date, and whether `elapsedTime` and statistics reflect the earlier date. Test on a device: end with `now − 20 min` and inspect the saved `HKWorkout` and its associated samples. If extra samples stay attached, the fallback is to disable collection of those types with `HKLiveWorkoutDataSource.disableCollection` as soon as the Finished message arrives. *(design inference)*
- The same applies when the **iPhone** ends a reconnected mirrored session with a past date by calling `stopActivity(with:)` on it. Whether the date carries over to the primary session is not documented **(unverified)**. Sending "Finished at T" over the data channel and letting the Watch call `stopActivity(with: T)` on its primary session avoids the question.
- Guard rails from Apple:
  - "If the user doesn't start and stop a workout session in your watchOS app, **don't try to retroactively create a workout session on Apple Watch.**" [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions). If the Watch first hears about a Workout after it is already Finished, it should not start a session at all.
  - HIG: "Discard extremely brief workout sessions. If a session ends a few seconds after it starts, either discard the data automatically or ask people…" [HIG Workouts](https://developer.apple.com/design/human-interface-guidelines/workouts). This applies when the Watch joins and immediately learns that the Workout was Finished.

## 5. Apple guidance and watchOS 11 / 26 / 27 features

- **HIG Workouts** says nothing about auto-starting. It covers in-session UI, summaries and discarding brief sessions. [HIG Workouts](https://developer.apple.com/design/human-interface-guidelines/workouts). The binding guidance is the HealthKit "Coordinate with the companion app" rule quoted in §3.
- **watchOS 11.1: open the Watch app from the iPhone Live Activity.** iPhone Live Activities appear in the Watch Smart Stack automatically. With `WKSupportsLiveActivityLaunchAttributeTypes` in the Watch app's Info.plist (an empty array means all activities), "If the person taps the Live Activity in the Smart Stack … the system launches the watchOS app and identifies that Live Activity as the reason for the launch." [WKSupportsLiveActivityLaunchAttributeTypes](https://developer.apple.com/documentation/bundleresources/information-property-list/wksupportsliveactivitylaunchattributetypes) (watchOS 11.1+), [WWDC24 10068](https://developer.apple.com/videos/play/wwdc2024/10068/). Alerting Live Activity updates can bring up the Smart Stack by themselves: "if it's currently at the watch face, the system automatically launches the Smart Stack, displays your alert". Combined with auto-join on open, this is a **one-tap** join on watchOS 11.1+, and it needs no iPhone app running beyond the Live Activity.
- **Auto-Launch settings (user-controlled).** Third-party write-ups describe a watchOS 11 setting under Settings → Smart Stack (formerly General → Auto-Launch) → Live Activities, with a per-app choice of Smart Stack, App or Off, where "App" opens the full Watch app. A developer reports it working for media apps. [pocket-lint](https://www.pocket-lint.com/how-to-control-live-activities-in-watchos-11/), [forum 787130](https://developer.apple.com/forums/thread/787130). Apple's support page only says "Tap Smart Stack, tap Live Activities, then set any of the options" [Apple Support](https://support.apple.com/guide/watch/view-live-activities-bz1vx4mwqbw9/watchos). Whether a third-party workout app's Live Activity offers "App", and whether that launch counts as foreground so a session may start, is **unverified**. If it does, it would give a user-opt-in **zero-tap** join, so it is worth a device test. "Auto-Launch Audio Apps" only applies to Now Playing audio **(unverified, no Apple page found)** and is irrelevant here.
- **Return to App** is a per-app user setting offered only for "Audiobooks, Maps, Mindfulness, Music, Now Playing, Podcasts, Stopwatch, Timers, Voice Memos, and Workout" [Apple Support: display settings](https://support.apple.com/guide/watch/adjust-the-display-settings-apd127ec93ac/watchos). We don't need it: an app with an active workout session already reappears on wrist raise.
- **watchOS 26 Smart Stack workout suggestions:** a HealthKit workout app "may be suggested in the Smart Stack based on a person's routine. They can tap on it to quickly get their workout started." It needs the correct `HKWorkoutActivityType` and accurate start and end times. [WWDC25 334](https://developer.apple.com/videos/play/wwdc2025/334/). This is routine-based, not triggered by a Live iPhone Workout, so it helps the Watch-first start rather than the join.
- **watchOS 27 / iOS 27 (WWDC26):** the health additions are workout zones (heart rate and power) and menopause data. No new session hand-off, launch or WatchConnectivity APIs were found. [WWDC26 watchOS guide](https://developer.apple.com/wwdc26/guides/watchos/), [What's new in watchOS](https://developer.apple.com/watchos/whats-new/), [HKWorkoutSession topics](https://developer.apple.com/documentation/healthkit/hkworkoutsession) *(absence, unverified)*.

## Resulting join strategy *(synthesis, not Apple's recommendation)*

1. **At start on the iPhone:** if `isPaired && isWatchAppInstalled`, call `startWatchApp` with a timeout. Publish the Live Workout (ID, start date, Routine) with `updateApplicationContext`, and start a Live Activity.
2. **While Live, with no mirrored session, and the iPhone app in the foreground:** retry `startWatchApp` at natural moments with back-off (§2).
3. **Watch, on any launch:** if the application context holds a Live Workout and no session is running, join automatically with `beginCollection(at: liveStart)` and mirror (§3). `handle(_:)` must be idempotent.
4. **Watch, on a background WatchConnectivity wake:** store the state and post a local notification or reload the complication. Don't try to start a session (§3).
5. **Watch app declares `WKSupportsLiveActivityLaunchAttributeTypes`** so one tap in the Smart Stack joins (§5).
6. **Finished while Disconnected:** the Watch ends with `stopActivity(with: finishedAt)` / `endCollection(withEnd: finishedAt)`. Test how samples are handled (§4). If the Watch never ran the Workout, create nothing.

**To verify on devices:** `startWatchApp` with the Watch locked, off-wrist, out of Bluetooth range but on Wi-Fi, and powered off (does the completion ever fire?). A past `endCollection` date with live samples. Whether a Watch app in a background refresh can wake the iPhone. Whether the per-app Live Activity "App" auto-launch applies to our app.

## Sources
- startWatchApp: https://developer.apple.com/documentation/healthkit/hkhealthstore/startwatchapp(with:completion:)
- WKApplicationDelegate handle(_:): https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handle(_:)-1pfoc
- Running workout sessions (Coordinate with the companion app): https://developer.apple.com/documentation/healthkit/running-workout-sessions
- HKError.Code: https://developer.apple.com/documentation/healthkit/hkerror/code
- HealthKit headers, iOS 26.0 SDK (HKDefines.h, HKHealthStore.h, HKWorkoutSession.h, HKWorkoutBuilder.h), via mirror: https://github.com/xybp888/iOS-SDKs/tree/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers
- stopActivity(with:): https://developer.apple.com/documentation/healthkit/hkworkoutsession/stopactivity(with:)
- endCollection(withEnd:): https://developer.apple.com/documentation/healthkit/hkworkoutbuilder/endcollection(withend:completion:)
- workoutSessionMirroringStartHandler: https://developer.apple.com/documentation/healthkit/hkhealthstore/workoutsessionmirroringstarthandler
- WCSession: https://developer.apple.com/documentation/watchconnectivity/wcsession
- WatchConnectivity overview: https://developer.apple.com/documentation/watchconnectivity
- WCSessionDelegate: https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate
- sessionReachabilityDidChange: https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate/sessionreachabilitydidchange(_:)
- sessionWatchStateDidChange: https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate/sessionwatchstatedidchange(_:)
- isReachable: https://developer.apple.com/documentation/watchconnectivity/wcsession/isreachable
- updateApplicationContext: https://developer.apple.com/documentation/watchconnectivity/wcsession/updateapplicationcontext(_:)
- remainingComplicationUserInfoTransfers: https://developer.apple.com/documentation/watchconnectivity/wcsession/remainingcomplicationuserinfotransfers
- WKWatchConnectivityRefreshBackgroundTask: https://developer.apple.com/documentation/watchkit/wkwatchconnectivityrefreshbackgroundtask
- Configuring background execution modes: https://developer.apple.com/documentation/xcode/configuring-background-execution-modes
- UNUserNotificationCenter: https://developer.apple.com/documentation/usernotifications/unusernotificationcenter
- WKSupportsLiveActivityLaunchAttributeTypes: https://developer.apple.com/documentation/bundleresources/information-property-list/wksupportsliveactivitylaunchattributetypes
- HIG Workouts: https://developer.apple.com/design/human-interface-guidelines/workouts
- WWDC16 235 Building Great Workout Apps (transcript copy): https://nonstrict.eu/wwdcindex/wwdc2016/235/
- WWDC24 10068 Bring your Live Activity to Apple Watch: https://developer.apple.com/videos/play/wwdc2024/10068/
- WWDC25 334 What's new in watchOS 26: https://developer.apple.com/videos/play/wwdc2025/334/
- WWDC26 watchOS guide: https://developer.apple.com/wwdc26/guides/watchos/
- What's new in watchOS: https://developer.apple.com/watchos/whats-new/
- Apple Support, display settings / Return to Clock: https://support.apple.com/guide/watch/adjust-the-display-settings-apd127ec93ac/watchos
- Apple Support, Live Activities on Apple Watch: https://support.apple.com/guide/watch/view-live-activities-bz1vx4mwqbw9/watchos
- Apple Support, Smart Stack: https://support.apple.com/guide/watch/see-widgets-in-the-smart-stack-apdecf142fb9/watchos
- Forums: https://developer.apple.com/forums/thread/787130 (DTS), https://developer.apple.com/forums/thread/734362 (Frameworks Engineer), https://developer.apple.com/forums/thread/688922 , https://developer.apple.com/forums/thread/52835 , https://developer.apple.com/forums/thread/661024 , https://developer.apple.com/forums/thread/662634 , https://developer.apple.com/forums/thread/737243
- Third-party: https://www.pocket-lint.com/how-to-control-live-activities-in-watchos-11/
