# HealthKit data types and APIs for strength Workouts and recovery

Research for issue #8. Target: iOS 17 / watchOS 10 minimum. Researched 2026-09-23 against Apple's developer documentation (read via the documentation's JSON endpoints, so availability figures come straight from Apple's platform metadata) and WWDC session pages. Where a claim comes from the Apple Developer Forums rather than the reference docs, it says so. Claims marked **(unverified)** have no primary source behind them.

## Answer

- **Activity type:** save each Workout as `HKWorkoutActivityType.traditionalStrengthTraining` ("machines or free weights"). `functionalStrengthTraining` ("free weights and body weight") is the other option, and Apple says it gets "optimized calorie calculations" on Watch. Both have existed since iOS 8 / watchOS 2.
- **Recording on Watch (watchOS 10+):** use `HKWorkoutSession`, `HKLiveWorkoutBuilder` and `HKLiveWorkoutDataSource`. The data source collects `heartRate` and `activeEnergyBurned` automatically. Mirror the session to iPhone with `startMirroringToCompanionDevice()` (watchOS 10) and `workoutSessionMirroringStartHandler` (iOS 17). **On iOS 17 to 25 the iPhone cannot run its own live session**: `HKLiveWorkoutBuilder` and `HKWorkoutSession.init` are iOS 26+. An iPhone-only Workout on older iOS is saved afterwards with `HKWorkoutBuilder` (iOS 12+), and it has no heart rate unless an external sensor was paired.
- **Heart-rate zones:** the system zones API (`zoneGroupsByType`, `preferredWorkoutZoneConfiguration(for:)`, live `didUpdateWorkoutZone`) is **iOS 27 / watchOS 27 only**. Below that we must **compute zones ourselves** from live heart-rate samples, for example from max HR and resting HR that we read or ask for. Adopt the system API behind `#available(watchOS 27, *)`.
- **Effort score:** write a `workoutEffortScore` quantity sample (unit `HKUnit.appleEffortScore()`, 0 to 10) and attach it with `HKHealthStore.relateWorkoutEffortSample(_:with:activity:)`. This is **iOS 18 / watchOS 11+ only**, so skip it on iOS 17 / watchOS 10. `estimatedWorkoutEffortScore` is created by the system, and apps only read it.
- **Recovery inputs (read only):** `restingHeartRate`, `heartRateVariabilitySDNN`, `sleepAnalysis` (sleep stages need iOS 16+, so they are fine on iOS 17), and `bodyMass`. Read them with `HKAnchoredObjectQuery` or statistics queries. For background refresh, pair `HKObserverQuery` with `enableBackgroundDelivery` (requires the background-delivery entitlement; at most hourly for most types on watchOS).
- **Authorisation:** users grant read and write separately for each type. **An app cannot tell whether read access was denied**, because a denied type simply looks empty. Only write status is visible, through `authorizationStatus(for:)`.
- **Dedupe:** put our Workout id in `HKMetadataKeyExternalUUID`. For a Workout we might re-save, also set `HKMetadataKeySyncIdentifier` and `HKMetadataKeySyncVersion`, so a later save with a higher version replaces the old one instead of duplicating it. Attach the keys with `HKWorkoutBuilder.addMetadata(_:)` before `finishWorkout()`. Look saved Workouts up with `HKQuery.predicateForObjects(withMetadataKey:allowedValues:)`.

## Detail

### 1. Workout activity type

| Constant | Apple's description | Availability |
|---|---|---|
| `traditionalStrengthTraining` | "strength training exercises primarily using machines or free weights" | iOS 8, watchOS 2 |
| `functionalStrengthTraining` | "strength training, primarily with free weights and body weight". Discussion: "HealthKit provides optimized calorie calculations for this activity based on the data from Apple Watch's sensors." | iOS 8, watchOS 2 |

Sources: https://developer.apple.com/documentation/healthkit/hkworkoutactivitytype/traditionalstrengthtraining , https://developer.apple.com/documentation/healthkit/hkworkoutactivitytype/functionalstrengthtraining

Recommendation: default to `traditionalStrengthTraining`, since it matches a gym Workout logged as Exercises and Sets. We could let a Routine switch to functional. **(unverified)** Apple does not document whether the calorie model for traditional strength training is worse than the one for functional. The note appears only on the functional page.

Every `HKWorkout` has at least one `HKWorkoutActivity` (iOS 16 / watchOS 9), which splits a workout into sub-activities. We could use activities to mark each Exercise's time span, but that is optional. Source: https://developer.apple.com/documentation/healthkit/hkworkoutactivity

### 2. Heart rate and active energy during a Workout

- `HKWorkoutSession` "fine-tunes Apple Watch's sensors for the specified activity. All workout sessions generate high-frequency heart rate samples." Apple Watch runs only one session at a time. If a second one starts, ours ends with `errorAnotherWorkoutSessionStarted`. Source: https://developer.apple.com/documentation/healthkit/hkworkoutsession
- `HKLiveWorkoutDataSource.typesToCollect` automatically collects types such as `activeEnergyBurned`, `basalEnergyBurned` and `heartRate`. We watch them through `workoutBuilder(_:didCollectDataOf:)`. Source: https://developer.apple.com/documentation/healthkit/hkliveworkoutdatasource/typestocollect
- `heartRate` uses count/time units and is a discrete type. `activeEnergyBurned` uses energy units, is cumulative, and "The system automatically records active energy samples on Apple Watch." Both may be condensed by HealthKit. Sources: https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/heartrate , https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/activeenergyburned
- For an iPhone-only Workout with no energy data, `splitTotalEnergy(_:start:end:resultsHandler:)` can split an estimated total into active and resting energy. Source: https://developer.apple.com/documentation/healthkit/hkhealthstore/splittotalenergy(_:start:end:resultshandler:)

**Platform availability matters for iOS 17:**

| API | iOS | watchOS |
|---|---|---|
| `HKWorkoutSession` class | 17.0 | 2.0 |
| `HKWorkoutSession.init(healthStore:configuration:)` | **26.0** | 5.0 |
| `HKLiveWorkoutBuilder`, `HKLiveWorkoutDataSource`, `associatedWorkoutBuilder()` | **26.0** | 5.0 |
| `startMirroringToCompanionDevice()` | n/a | 10.0 |
| `workoutSessionMirroringStartHandler`, `sendToRemoteWorkoutSession(data:)` | 17.0 | 10.0 |
| `HKWorkoutBuilder` (non-live) | 12.0 | 5.0 |

Sources: https://developer.apple.com/documentation/healthkit/hkworkoutsession , https://developer.apple.com/documentation/healthkit/hkworkoutsession/init(healthstore:configuration:) , https://developer.apple.com/documentation/healthkit/hkliveworkoutbuilder , https://developer.apple.com/documentation/healthkit/hkworkoutsession/associatedworkoutbuilder() , https://developer.apple.com/documentation/healthkit/hkworkoutsession/startmirroringtocompaniondevice(completion:) , https://developer.apple.com/documentation/healthkit/hkhealthstore/workoutsessionmirroringstarthandler , https://developer.apple.com/documentation/healthkit/hkworkoutsession/sendtoremoteworkoutsession(data:completion:) , https://developer.apple.com/documentation/healthkit/hkworkoutbuilder

This means:
- On **iOS 17 to 25**, the iPhone can only *mirror* a session that the Watch started. It receives the session and can exchange custom data with `sendToRemoteWorkoutSession`. The Watch saves the `HKWorkout`. Because mirroring may restart after a disconnect, "your app may receive multiple calls" to the start handler.
- On **iOS 26+**, the iPhone can run its own live session. WWDC25 session 322 says: "these devices don't contain a heart rate sensor", but paired Bluetooth HR monitors are supported, and the "iPhone will most likely lock while a workout is running". Source: https://developer.apple.com/videos/play/wwdc2025/322/
- `HKWorkoutBuilder.finishWorkout()` "returns nil if finishing the workout succeeded but the workout sample is not available because the device is locked." Source: https://developer.apple.com/documentation/healthkit/hkworkoutbuilder/finishworkout(completion:)

### 3. Heart-rate zones

- **iOS 27 / watchOS 27+:** HealthKit calculates zones from heart-rate samples added to an `HKWorkoutBuilder` or `HKLiveWorkoutBuilder`: "You don't need to call any additional methods." It uses the person's zone settings from Health, and "For heart rate, the system provides default zones even if the person hasn't configured preferences." The APIs are `HKWorkout.zoneGroupsByType` / `HKWorkoutActivity.zoneGroupsByType` (configuration plus time in each zone), `HKHealthStore.preferredWorkoutZoneConfiguration(for:)`, the live `HKLiveWorkoutBuilderDelegate.workoutBuilder(_:didUpdateWorkoutZone:)`, and `setCustomZoneConfiguration(_:for:)` for zones defined by the app. The zone source is `.system`, `.user` or `.app`. Sources: https://developer.apple.com/documentation/healthkit/accessing-workout-zone-data , https://developer.apple.com/documentation/healthkit/hkworkout/zonegroupsbytype , https://developer.apple.com/documentation/healthkit/hkhealthstore/preferredworkoutzoneconfiguration(for:) , https://developer.apple.com/videos/play/wwdc2026/207/
- **Before 27 (our iOS 17 / watchOS 10 minimum):** HealthKit has no public zones API. The zones in the Apple Workout app (watchOS 9) were not exposed to apps. **(unverified: we found no Apple statement saying so directly. The inference rests on the zones API arriving only in 27. See also the forum question https://developer.apple.com/forums/thread/718549)**. We have to compute zones on the Watch from live `heartRate` statistics, using our own model, such as a percentage of max HR or HR reserve with resting HR read from HealthKit.

### 4. Workout effort score (iOS 18 / watchOS 11+)

- `HKQuantityTypeIdentifier.workoutEffortScore` and `.estimatedWorkoutEffortScore`: iOS 18.0, watchOS 11.0, macOS 15. Unit: `HKUnit.appleEffortScore()` (same availability). Sources: https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/workouteffortscore , https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/estimatedworkouteffortscore , https://developer.apple.com/documentation/healthkit/hkunit/appleeffortscore()
- Relating it to a workout: `func relateWorkoutEffortSample(_ sample: HKSample, with workout: HKWorkout, activity: HKWorkoutActivity?) async throws -> Bool` (iOS 18 / watchOS 11). Pass `activity: nil` for the whole workout. Reading it back: `HKWorkoutEffortRelationshipQuery` / `HKWorkoutEffortRelationship` (iOS 18). Sources: https://developer.apple.com/documentation/healthkit/hkhealthstore/relateworkouteffortsample(_:with:activity:completion:) , https://developer.apple.com/documentation/healthkit/hkworkouteffortrelationshipquery
- Apple's reference pages have no prose for these symbols. The following comes from the **Apple Developer Forums (DTS / engineer answers, secondary)**:
  - Values run from 0 to 10.
  - Request **share and read** access for `workoutType()` and `workoutEffortScore`.
  - Build an `HKQuantitySample` spanning the workout's start and end, then call `relateWorkoutEffortSample` rather than `add` or `builder.add`.
  - `estimatedWorkoutEffortScore` is created by the system only.
  - Source: https://developer.apple.com/forums/thread/763539 and https://developer.apple.com/forums/thread/764884
- On iOS 17 / watchOS 10 we can keep effort only in our own data, and write it to HealthKit when the user is on 18 / 11 or later.

### 5. Body mass and recovery inputs

| Type | Notes | Availability |
|---|---|---|
| `bodyMass` | mass units, discrete | iOS 8 |
| `restingHeartRate` | "estimation of the user's lowest heart rate during periods of rest". The system "may delete earlier samples and replace them with better estimates" for the current or previous day, so re-query with an anchored query instead of caching the first value. | iOS 11, watchOS 4 |
| `heartRateVariabilitySDNN` | SDNN in time units (ms). Recorded automatically on Apple Watch. | iOS 11, watchOS 4 |
| `sleepAnalysis` (category) | `HKCategoryValueSleepAnalysis`: in-bed samples overlap the stage samples (awake, `asleepCore`, `asleepDeep`, `asleepREM`). Stage values arrived in iOS 16 / watchOS 9. Watch data only includes awake periods *between* sleep samples. | iOS 8; stages iOS 16 |

Sources: https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/bodymass , https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/restingheartrate , https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/heartratevariabilitysdnn , https://developer.apple.com/documentation/healthkit/hkcategorytypeidentifier/sleepanalysis , https://developer.apple.com/documentation/healthkit/hkcategoryvaluesleepanalysis , https://developer.apple.com/documentation/healthkit/hkcategoryvaluesleepanalysis/asleepcore

HealthKit provides no readiness or recovery score. We compute it from these inputs. **(unverified: no Apple page says so; we simply found no such type.)**

### 6. Authorisation granularity and read opacity

- "Each data type has two separate permissions, one to read it and one to share it." The permission sheet appears only for types the user has not yet decided on. On watchOS 6+ the sheet appears on the Watch itself. `NSHealthShareUsageDescription` and `NSHealthUpdateUsageDescription` are required, or the app crashes. Source: https://developer.apple.com/documentation/healthkit/hkhealthstore/requestauthorization(toshare:read:)
- `authorizationStatus(for:)` reports **write** status only. "Your app cannot determine whether or not a user has granted permission to read data. If you are not given permission, it simply appears as if there is no data." With write access but no read access, the app sees only its own data. Source: https://developer.apple.com/documentation/healthkit/hkhealthstore/authorizationstatus(for:)
- `getRequestStatusForAuthorization(toShare:read:)` (iOS 12) says whether a sheet *would* be shown, which is useful for deciding when to show our own pre-prompt. Source: https://developer.apple.com/documentation/healthkit/hkhealthstore/getrequeststatusforauthorization(toshare:read:completion:)
- The HealthKit store is encrypted while the device is locked, so background reads can fail. Writes are cached until unlock. Source: https://developer.apple.com/documentation/healthkit/protecting-user-privacy

UX consequence: in the Health-enriched history, treat "no data" as possibly "no permission" and offer a link to settings. Don't claim that access was denied.

### 7. Background delivery and anchored queries

- `HKAnchoredObjectQuery` returns new **and deleted** objects since an anchor, and can keep running with an `updateHandler`. This is the right tool for syncing resting HR, HRV, sleep and body mass into our history, and for catching resting-HR samples the system replaces. Source: https://developer.apple.com/documentation/healthkit/hkanchoredobjectquery
- `HKObserverQuery` plus `enableBackgroundDelivery(for:frequency:)` wakes the app when data changes.
  - Requires the `com.apple.developer.healthkit.background-delivery` entitlement from iOS 15 / watchOS 8; without it the call fails with `errorAuthorizationDenied`.
  - Some types are limited to hourly on iOS.
  - On watchOS "most data types have an hourly maximum frequency", and updates share a budget with background refresh: "four updates … an hour, as long as it has a complication on the active watch face."
  - Register the queries at launch and call the completion handler. After three failures, delivery stops.
  - Not supported in the Simulator.
  - Sources: https://developer.apple.com/documentation/healthkit/hkhealthstore/enablebackgrounddelivery(for:frequency:withcompletion:) , https://developer.apple.com/documentation/healthkit/hkobserverquery

### 8. Custom metadata and duplicate avoidance

- `HKMetadataKeyExternalUUID` (iOS 8): "A unique identifier for an HKObject that is set by its source … You typically use the UUID from the corresponding data entry on your server." Put our Workout id here. It does **not** dedupe on its own. Source: https://developer.apple.com/documentation/healthkit/hkmetadatakeyexternaluuid
- `HKMetadataKeySyncIdentifier` plus `HKMetadataKeySyncVersion` (iOS 11 / watchOS 4): "If the new object has a greater sync version, the system replaces the old object with the new one. If the old object is associated with a workout … the system also replaces the old object in the workout." The two keys must be used together. Sources: https://developer.apple.com/documentation/healthkit/hkmetadatakeysyncidentifier , https://developer.apple.com/documentation/healthkit/hkmetadatakeysyncversion
- Attach metadata with `HKWorkoutBuilder.addMetadata(_:)` (iOS 12 / watchOS 5; `HKLiveWorkoutBuilder` inherits it) before `finishWorkout()`. Custom keys (for example our Routine id) are allowed. Source: https://developer.apple.com/documentation/healthkit/hkworkoutbuilder/addmetadata(_:completion:)
- Find our Workout later with `HKQuery.predicateForObjects(withMetadataKey:allowedValues:)`, which "You may also search using custom keys." Source: https://developer.apple.com/documentation/healthkit/hkquery/predicateforobjects(withmetadatakey:allowedvalues:)
- Suggested scheme: `ExternalUUID = syncIdentifier = <our Workout id>`, with `syncVersion` bumped on each edit.
  - Save from exactly one device, normally the Watch that ran the session; the iPhone saves only for iPhone-only Workouts. That avoids duplicate HKWorkouts when both devices are involved.
  - **(unverified)** Does sync-identifier replacement behave well for `HKWorkout` objects specifically (keeping related effort samples and routes), rather than only for simple samples? Test it on a device.

### 9. iOS 17 vs later: summary

| Capability | iOS 17 / watchOS 10 | iOS 18 / watchOS 11 | iOS 26 | iOS 27 / watchOS 27 |
|---|---|---|---|---|
| Watch live session plus mirroring to iPhone | yes | yes | yes | yes |
| iPhone standalone live session (`HKLiveWorkoutBuilder`) | no | no | yes | yes |
| Effort score write/relate | no | yes | yes | yes |
| System heart-rate zones API | no (compute ourselves) | no | no | yes |
| Sleep stages, RHR, HRV, body mass read | yes | yes | yes | yes |
