# Watch force-quit, lost sessions and heart-rate samples

This note covers what happens to a **Live** Workout's HealthKit workout session (`HKWorkoutSession`) and its heart-rate and energy samples when the Watch app is force-quit, when a Workout is **Discarded**, when it is **Finished** with an end date in the past, and when the Watch reboots. Targets iOS 17 / watchOS 10 minimum. Terms follow `CONTEXT.md` (**Workout**, **Set**, **Live**, **Finished**, **Discard**).

This builds on [`watch-workout-and-sync.md`](watch-workout-and-sync.md), which covers crash recovery via `handleActiveWorkoutRecovery()` / `recoverActiveWorkoutSession`, and on [`automatic-watch-join.md`](automatic-watch-join.md) §4, which covers what the API allows for past end dates. Neither is repeated here.

Sources were read on 2026-09-23. Every claim is labelled **Apple** (Apple docs, SDK headers, WWDC, or an Apple engineer on the forums) or **Developer report** (forum posts or blogs by non-Apple developers). Header quotes come from the iOS 26.0 SDK `HealthKit.framework/Headers`, read through the [xybp888/iOS-SDKs](https://github.com/xybp888/iOS-SDKs) mirror. The text is Apple's, but the mirror is not an Apple site.

## Answer

| # | Question | Verdict | Evidence |
|---|---|---|---|
| 1 | User force-quits the Watch app during a Live Workout | **The session probably keeps running; nothing is saved; the app may not be relaunched.** Apple documents recovery only for *crashes*. Developers report that after a user force-quit the session is **not ended**. `handleActiveWorkoutRecovery()` is sometimes not called when the app is next opened, but calling `recoverActiveWorkoutSession` at every launch returns the session. No `HKWorkout` is saved unless someone calls `finishWorkout()`. | Weak to medium: Apple is silent, and developer reports (2018–2024) disagree on whether the app is relaunched automatically. |
| 2 | Are live HR and energy samples written during the session or only at `finishWorkout()`? Do they survive Discard or a lost session? | **Written during the session. They stay in Health after Discard.** Apple: "While the session runs, Apple Watch automatically collects data about the workout, and saves samples to the HealthKit store." The `discardWorkout` header says: "Samples that were added to the workout will not be deleted." After a lost session, samples already saved should stay as loose samples with no `HKWorkout`. That last point is an inference, not documented. | Strong for "during the session" and for Discard. Medium for a lost session (inference from the same docs). |
| 3 | `endCollection(withEnd:)` / `stopActivity(with:)` with a past end date | **Samples after that date still exist in Health** because they were saved while the session ran (see Q2). Whether the builder leaves them out of the saved `HKWorkout`'s associated samples and statistics is **not documented** by Apple and no developer report was found. | Strong that they stay in Health. None on whether they are associated with the workout (test on a device). |
| 4 | Is there recovery after a reboot or a dead battery? | **No, as far as anyone reports.** Apple documents relaunch only after a crash. A developer blog says `handleActiveWorkoutRecovery` "is **not** called when the watch is rebooted". Users of Apple's own Workout app report that a workout is lost when the battery dies before it is saved. | Medium: first-party docs say nothing; developer and user reports are consistent. |

**Design consequence:** treat the Watch session as disposable, and keep our own Workout and Sets as the source of truth (already the plan). On *every* Watch app launch, call `recoverActiveWorkoutSession`, not just inside `handleActiveWorkoutRecovery()`. When we Discard, remember that HR and energy samples are already in Health. If we must remove them, delete them explicitly (see §2).

---

## 1. User force-quit during an active session

### What Apple says
- Apple: `handleActiveWorkoutRecovery()` "Tells the delegate when the app relaunches **after crashing** during an active workout session." [handleActiveWorkoutRecovery()](https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handleactiveworkoutrecovery())
- Apple (header): `recoverActiveWorkoutSessionWithCompletion:` "Recovers an active workout session after a client crash. **If no session is available to be re-attached, nil will be returned.**" [HKHealthStore.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKHealthStore.h). The nil case makes it safe to call at every launch.
- Apple (WWDC18 707): "If your application happens to crash during an active workout, we will automatically relaunch it and give it a chance to recover the workout." [WWDC18 707 transcript (third-party copy)](https://nonstrict.eu/wwdcindex/wwdc2018/707/)
- Apple (engineer, forum 651009, Jun 2020): asked about an app that was killed or force-quit mid-workout, the engineer said: "You would need to implement workout recovery … do not start a new HKWorkoutSession, but call into recoverActiveWorkoutSession(completion:) to recover the original session. The HKLiveWorkoutBuilder associated with the recovered session should have the correct start date and include all pause/resume events. You would still need to attach a new HKLiveWorkoutDataSource." The follow-up question, whether this also applies to a user kill, got no answer. [Forum 651009](https://developer.apple.com/forums/thread/651009)
- **No Apple document mentions a user force-quit.** Session end happens only through `end()` ("the system will exit session mode") or through the system, for example when another app starts a session (`errorAnotherWorkoutSessionStarted`). [HKWorkoutSession.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKWorkoutSession.h)

### Developer reports
- **Session survives, and the callback is not called on relaunch** (jbrunhuber, Oct 2021, accepted answer): "it only works when a crash occurs. When the user closes the App during a workout, **the session still remains** and there's no way to end it when the user relaunches the App." Then: "`handleActiveWorkoutRecovery` won't be called after relaunch, but I was able to successfully restore the session by calling `recoverActiveWorkoutSession` in `applicationDidFinishLaunching`." [Forum 692890](https://developer.apple.com/forums/thread/692890)
- **Force-quit does trigger recovery** (lewis42, Sep 2018): "I tried force quitting the watch app, and doing that is [what] you need to do trigger handleActiveWorkoutRecovery." [Forum 108301](https://developer.apple.com/forums/thread/108301)
- **Callback unreliable, so call it yourself** (maperkins, May 2024): "it appears handleActiveWorkoutRecovery isn't called, so we are … calling it directly" at launch. [Forum 108301](https://developer.apple.com/forums/thread/108301)
- **App UI resets after a swipe-to-close** (danteppc, Jun 2019): after "opening the task manager and swiping the app, and then relaunches it, every view controller is set to their initial state", and `recoverActiveWorkoutSession` "doesn't restore the app to previous state". [Forum 117115](https://developer.apple.com/forums/thread/117115)
- A developer blog recommends checking `recoverActiveWorkoutSession` in `applicationDidFinishLaunching`, and notes that it restores only HealthKit data, not the app's own data. [fatbobman](https://fatbobman.com/en/posts/watchos-development-pitfalls-and-practical-tips)

### Is anything saved?
No. An `HKWorkout` is created only by `finishWorkout()` ("Creates and saves an HKWorkout using samples and events that have been added"). [HKWorkoutBuilder.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKWorkoutBuilder.h). HR and energy samples collected so far are already in Health (§2).

### How users force-quit
- On watchOS 10–26, users can swipe an app away in the app switcher / Dock, or hold the side button and then the Digital Crown. On **watchOS 27**, the Digital Crown double-click no longer opens the app switcher, so the only way is: hold the side button until the power sliders appear, then hold the Digital Crown. [MacRumors, watchOS 27](https://www.macrumors.com/how-to/force-quit-apple-watch-apps-watchos-27/) (secondary). It is not confirmed whether removing an app from the Dock terminates it or only hides it.

### Device test to settle it
Start a Live Workout on watchOS 10 and on the current release. Force-quit with each gesture. Then check: (a) does the green workout indicator stay and does HR keep arriving in Health; (b) does the app relaunch by itself within about 1 minute, and is `handleActiveWorkoutRecovery()` called; (c) when opened manually, does `recoverActiveWorkoutSession` in the app delegate's launch return a session in `.running`?

## 2. When live samples reach the HealthKit store

- Apple: "While the session runs, Apple Watch automatically collects data about the workout, and **saves samples to the HealthKit store**." Also: "while the session runs, Apple Watch automatically saves active energy-burned samples to the HealthKit store." [Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions)
- Apple (header, `addSamples:`): "This method can be called multiple times to add samples incrementally to the builder. The samples will be saved to the database if they have not already been saved." [HKWorkoutBuilder.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKWorkoutBuilder.h)
- Apple (header, class): "An HKWorkoutBuilder is used to **incrementally** create new workouts in the HealthKit database." Same header.
- Apple (header, `discardWorkout`): "Finishes building the workout and discards the result instead of saving it. **Samples that were added to the workout will not be deleted.**" Same header. The docs page abstract says only "discards the current results without saving the workout". [discardWorkout()](https://developer.apple.com/documentation/healthkit/hkworkoutbuilder/discardworkout())
- Apple (header, `finishWorkout`): "Creates and saves an HKWorkout using samples and events that have been added". So at finish, what is new is the `HKWorkout` and its associations, not the samples themselves.
- Developer report: someone called `discardWorkout` and still "observed unexpected workout data being saved to HealthKit" (Dec 2018, no replies). [Forum 112183](https://developer.apple.com/forums/thread/112183)

**Consequence for Discard:** a Discarded Workout leaves its HR and active-energy samples in Health, with our app as the source (they then also count toward Activity rings). If Discard must leave no trace, the app has to delete them itself. `HKHealthStore.deleteObjects(of:predicate:)` with a date range and a source predicate for our app is the likely route. **(Unverified. Check that it catches only our session's samples, and that deleting active energy is allowed.)** The HIG allows either discarding automatically or asking, for very short sessions ([HIG Workouts](https://developer.apple.com/design/human-interface-guidelines/workouts)). It does not say whether samples should be removed.

**Consequence for a lost session** (force-quit that is never recovered, or a reboot): the samples saved so far should remain as loose samples with no `HKWorkout`. This is an **inference** from the "saves samples … while the session runs" statement. No document covers it directly.

## 3. Past end date and samples collected after it

- Apple (header, `stopActivityWithDate:`): after stopping, "Sensor algorithms will be stopped and **no new data will be generated** for this session. However, the system will remain in session mode." So new data stops when `stopActivity` is *called*, not at the date passed in. [HKWorkoutSession.h](https://github.com/xybp888/iOS-SDKs/blob/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers/HKWorkoutSession.h)
- Apple (header): `endCollection` "Sets the workout end date and deactivates the workout builder". `addSamples` constrains only the *start* ("The start date of the samples must be later than the start date of the receiver"). No rule about the end date is documented. Same header.
- Samples from between the past end date and the moment of the call were already saved to Health while the session ran (§2), so **they remain in Health as samples from our app** whatever the builder does.
- **Not documented, and no developer report was found:** whether `finishWorkout()` associates those later samples with the `HKWorkout` (as returned by `HKQuery.predicateForObjects(from: workout)`), and whether `HKWorkout.statistics(for:)` / total energy include them.

### Device test to settle it
Run a session for 10 minutes. Call `stopActivity(with: now − 5 min)`, then `endCollection(withEnd: now − 5 min)`, then `finishWorkout()`. Then compare: (a) HR samples from our source in [end, now] in Health; (b) samples matching `predicateForObjects(from: workout)`; (c) `workout.statistics(for: activeEnergyBurned)` against the sum of energy samples inside [start, end]. If later samples are attached, the fallbacks are: call `disableCollection(for:)` on the data source as soon as the Watch learns the Workout is Finished, or build the workout with a plain `HKWorkoutBuilder` from filtered samples. *(design inference)*

## 4. Reboot or dead battery mid-session

- Apple: recovery is documented only for crashes ([Running workout sessions](https://developer.apple.com/documentation/healthkit/running-workout-sessions) "Recover from crashes", and [handleActiveWorkoutRecovery()](https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handleactiveworkoutrecovery())). Nothing mentions reboot or power loss.
- Developer report (blog): "The `handleActiveWorkoutRecovery` method is **not** called when the watch is rebooted." It advises checking `recoverActiveWorkoutSession` at launch anyway. [fatbobman](https://fatbobman.com/en/posts/watchos-development-pitfalls-and-practical-tips)
- User reports (Apple Community, not Apple staff): with Apple's own Workout app, a workout that had not been saved when the battery died cannot be recovered. [Apple Community 7804047](https://discussions.apple.com/thread/7804047)
- Inference: a session is a running process state that ends with the process, so a reboot most likely ends it and nothing relaunches our app. Samples saved before the power loss are likely still in Health (§2), with no `HKWorkout`. On the next launch, `recoverActiveWorkoutSession` should return nil, and our own stored Live Workout should then be offered for resume or Finish on the iPhone. *(design inference)*

### Device test to settle it
Start a Live Workout and restart the Watch (hold the side button, then Power Off). After boot: does the app launch by itself? Does `recoverActiveWorkoutSession` return nil? Are the HR samples from before the restart in Health?

## Sources
- handleActiveWorkoutRecovery(): https://developer.apple.com/documentation/watchkit/wkapplicationdelegate/handleactiveworkoutrecovery()
- recoverActiveWorkoutSession(completion:): https://developer.apple.com/documentation/healthkit/hkhealthstore/recoveractiveworkoutsession(completion:)
- Running workout sessions: https://developer.apple.com/documentation/healthkit/running-workout-sessions
- discardWorkout(): https://developer.apple.com/documentation/healthkit/hkworkoutbuilder/discardworkout()
- endCollection(withEnd:): https://developer.apple.com/documentation/healthkit/hkworkoutbuilder/endcollection(withend:completion:)
- SDK headers (iOS 26.0, mirror): HKWorkoutBuilder.h, HKWorkoutSession.h, HKHealthStore.h, HKLiveWorkoutBuilder.h, HKLiveWorkoutDataSource.h at https://github.com/xybp888/iOS-SDKs/tree/master/iPhoneOS26.0.sdk/System/Library/Frameworks/HealthKit.framework/Headers
- WWDC18 707 New Ways to Work with Workouts (transcript copy): https://nonstrict.eu/wwdcindex/wwdc2018/707/
- HIG Workouts: https://developer.apple.com/design/human-interface-guidelines/workouts
- Apple Developer Forums: https://developer.apple.com/forums/thread/651009 (Apple engineer), https://developer.apple.com/forums/thread/692890 , https://developer.apple.com/forums/thread/108301 , https://developer.apple.com/forums/thread/117115 , https://developer.apple.com/forums/thread/112183 (developers)
- Developer blog: https://fatbobman.com/en/posts/watchos-development-pitfalls-and-practical-tips
- Apple Community (users): https://discussions.apple.com/thread/7804047
- MacRumors force-quit on watchOS 27: https://www.macrumors.com/how-to/force-quit-apple-watch-apps-watchos-27/
