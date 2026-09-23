# Ads, tracking consent and App Store rules in a HealthKit app

Research for issue #7. Question: what constraints apply to showing AdMob banners on every iPhone screen of the Workout Tracker, which uses HealthKit, with a non-consumable "remove ads" In-App Purchase and no ads on Apple Watch?

Researched 2026-09-23 against Apple and Google primary sources. Items marked **(unverified)** were not confirmed in a primary source.

## Answer

- **Ads in a HealthKit app are allowed.** Apple says so outright: "you may still serve advertising in an app that uses the HealthKit framework, but you can't use data from the HealthKit store to serve ads" ([HealthKit: Protecting user privacy][hk-privacy]). Guidelines 5.1.2, 5.1.3(i) and 2.5.18 forbid using health or fitness data, including Workouts, Sets and anything read from HealthKit, to target ads, or giving it to an ad network ([Guidelines][guidelines]). The design rule: **never pass Workout, Exercise or HealthKit data to the Google Mobile Ads SDK** (no keywords, no content URLs, no custom targeting).
- **No ads on Apple Watch is required, not just a choice.** Guideline 2.5.18 limits display ads to the main app binary, "not … watchOS apps" ([Guidelines][guidelines]).
- **Two 2.5.18 rules are easy to miss.** Users must be able to see what was used to target an ad without leaving the app, and apps with ads "must also include the ability for users to report any inappropriate or age-inappropriate ads" ([Guidelines][guidelines]). Plan an in-app "Report an ad" path. Whether AdMob's AdChoices overlay alone satisfies App Review is **(unverified)**.
- **Consent works in layers.** (1) Show Google UMP on every launch. A Google-certified TCF CMP is required for personalised ads in the EEA and UK (from 16 January 2024) and in Switzerland (from 31 July 2024) ([AdMob help][cmp]). (2) Show the ATT prompt before loading ads, using `NSUserTrackingUsageDescription` ([Google iOS 14+][ios14]). (3) Handle US-state opt-outs through UMP/GPP or `gad_rdp` ([US states][us-states]). Ads still serve if the user declines ATT, because SKAdNetwork attribution does not need the IDFA ([Google iOS 14+][ios14]). No feature may depend on granting tracking (5.1.2).
- **The privacy label will show "Data Used to Track You".** The Google Mobile Ads SDK collects IP address (coarse location), Device ID, Advertising Data, Product Interaction, Crash Data and Performance Data, and shares them with third-party advertisers ([Google data disclosure][gma-disclosure]). The SDK ships a privacy manifest from v11.2.0 ([Google data disclosure][gma-disclosure]). Health and Fitness data is declared only if it leaves the device ([App privacy details][privacy-details]).
- **The SDK supports SwiftUI and iOS 17.** It needs iOS 13+ and Xcode 16+, installs with SPM, and shows banners in SwiftUI through a `UIViewRepresentable` wrapper ([Quick start][quickstart], [Banner][banner]).
- **"Remove ads" with StoreKit 2:** read `Transaction.currentEntitlements` at launch, listen to `Transaction.updates`, and keep only `.verified` results. Also provide a Restore Purchases button that calls `AppStore.sync()` (3.1.1) ([currentEntitlements][ce], [sync][sync], [updates][updates], [VerificationResult][vr]). The Watch never shows ads, so it does not need to check the entitlement.

**Risk flag:** none of these rules blocks the plan. App Review is most likely to reject over (a) a missing in-app way to report an ad (2.5.18), (b) any health or Workout data reaching the ad SDK, and (c) gating features on ATT consent. A pure fitness tracker also appears to fall outside the "highly regulated … healthcare" legal-entity rule in 5.1.1(ix) **(unverified)**.

## Detail

### App Store Review Guidelines ([source][guidelines])

- **5.1.1 Data Collection and Storage.** A privacy policy link is required in App Store Connect and inside the app. It must name third parties that receive data, "such as analytics tools, advertising networks and third-party SDKs". Apps "must secure user consent for the collection", "Paid functionality must not be dependent on or require a user to grant access to this data", and users need an easy way to withdraw consent. Apps relying on GDPR legitimate interest must comply fully with that law.
- **5.1.1(ix).** Apps "in highly regulated fields (such as … healthcare …)" should be submitted by a legal entity. A strength-training tracker is fitness, not healthcare, so this probably does not apply **(unverified)**.
- **5.1.2 Data Use and Sharing.** "You must receive explicit permission from users via the App Tracking Transparency APIs to track their activity." Apps "may not require users to enable system functionalities (e.g. … tracking) in order to access functionality". "Data gathered from … HealthKit … may not be used for marketing, advertising or use-based data mining, including by third parties."
- **5.1.3(i) Health and Health Research.** Data "gathered in the health, fitness, and medical research context — including from … HealthKit API, Motion and Fitness" may not be used or disclosed to third parties "for advertising, marketing, or other use-based data mining purposes". "You must disclose the specific health data that you are collecting from the device." Workout and Set data the app records itself arguably falls under the "fitness … context" too, so treat it like HealthKit data.
- **5.1.3(ii).** No false data may be written to HealthKit, and apps "may not store personal health information in iCloud". This matters for sync design, though it is outside this ticket.
- **2.5.18 Advertising.** "Display advertising should be limited to your main app binary, and should not be included in extensions, App Clips, widgets, notifications, keyboards, watchOS apps, etc." Ads must match the age rating. Users must be able to "see all information used to target them for that ad (without requiring the user to leave the app)". Ads may not use "targeted or behavioral advertising based on sensitive user data such as health/medical data (e.g. from the HealthKit APIs)". Apps with ads "must also include the ability for users to report any inappropriate or age-inappropriate ads". Banners are not interstitials, so the close-button rules do not apply to them.
- **3.1.1 In-App Purchase.** Unlocking features such as removing ads must use In-App Purchase. Apps must provide "a restore mechanism for any restorable in-app purchases". The no-trial plan means the "XX-day Trial" tier-0 pattern is not needed.

### HealthKit ([source][hk-privacy])

- "Your app may not use information gained through the use of the HealthKit framework for advertising or similar services. Note that you may still serve advertising in an app that uses the HealthKit framework, but you can't use data from the HealthKit store to serve ads."
- HealthKit data may not go to a third party without express permission, and even then only to one that provides a health or fitness service. It may never be sold to advertising platforms or data brokers.
- HealthKit use must be for health or fitness and must be clear in both marketing text and UI.

### App Tracking Transparency

- The ATT framework is needed "if [the app] collects data about people and shares it with other companies to track them across apps and websites". Add `NSUserTrackingUsageDescription` and call `requestTrackingAuthorization` ([ATT][att]).
- The prompt appears only when the app is active. In the EU a user's answer blocks re-prompting for a year. The prompt does not appear at all when "Allow Apps to Request to Track" is off ([requestTrackingAuthorization][att-req]).
- Google recommends waiting for the ATT completion handler before loading ads so the IDFA can be used if the user allows it. It also recommends adding Google's `SKAdNetworkItems` to Info.plist. SKAdNetwork attributes installs "even when the IDFA is not available" ([Google iOS 14+][ios14]). UMP can show an optional IDFA explainer before the system prompt ([Google iOS 14+][ios14]).
- **Consequence:** banners serve whether the user allows or declines ATT. Declining gives non-personalised or limited ads and lower revenue, but no App Review problem.

### Google consent (UMP, GDPR, US states)

- UMP: call `requestConsentInfoUpdate` "on every app launch", load or present the form if one is required, then check `canRequestAds` before requesting ads. `canRequestAds` is false until the update call completes ([AdMob privacy][admob-privacy]).
- EEA, UK and Switzerland: "a certified CMP integrated with the TCF is required when serving personalized ads to users" (EEA/UK from 16 January 2024, Switzerland from 31 July 2024). UMP is Google's own certified CMP ([AdMob help][cmp]).
- US states: turn on Restricted Data Processing (`gad_rdp` = true in UserDefaults), or signal choices through IAB GPP, which UMP can write. The publisher decides when to apply these ([US states][us-states]).
- Further controls: `publisherPrivacyPersonalizationState` turns off personalisation for all requests. `tagForUnderAgeOfConsent` / age-restricted treatment disables personalised ads and the IDFA. `maxAdContentRating` caps ad content to fit the app's age rating (2.5.18) ([Targeting][targeting]).
- Withdrawing consent (5.1.1): UMP provides a privacy-options entry point that belongs on a Settings screen. The exact API name was not confirmed on the pages read **(unverified)**.

### Privacy manifest and App Store privacy label

- "Collect" means sending data off the device where you or third-party partners can access it for longer than needed to serve the request. Third-party SDKs such as ad networks count as partners ([App privacy details][privacy-details]).
- The Google Mobile Ads SDK collects IP address (approximate location), crash logs, performance data, Device ID (IDFA or app-scoped IDs), advertising data and product interaction. It uses them for third-party advertising and analytics ([Google data disclosure][gma-disclosure]). Expect the app's label to show Device ID, Advertising Data, Product Interaction, Coarse Location and Diagnostics. Several of these fall under "Data Used to Track You" when ATT is granted.
- The SDK includes a privacy manifest from v11.2.0. The developer must still check the combined Xcode privacy report against the label, and mediation adapters need their own manifests ([Google data disclosure][gma-disclosure], [Privacy manifest files][manifest]).
- GoogleMobileAds itself is not on Apple's "SDKs that require a privacy manifest and signature" list, though some Google dependencies (e.g. GoogleUtilities) are ([Third-party SDK requirements][sdk-reqs]). A current SDK version meets this either way.
- Health & Fitness label categories cover HealthKit and Motion and Fitness data. They apply only if the app sends such data off the device, for example to a backend or for Community Exercise features ([App privacy details][privacy-details]).

### SDK support for SwiftUI and iOS 17

- Requirements: Xcode 16.0+ and iOS 13.0+. Swift Package Manager is the recommended install ([Quick start][quickstart]).
- SwiftUI: Google's banner guide shows a `BannerViewContainer: UIViewRepresentable` that wraps `BannerView`, using anchored adaptive sizes such as `largeAnchoredAdaptiveBanner(width:)` ([Banner][banner]). No official native SwiftUI view exists; the wrapper is the supported pattern.
- The SDK does not support watchOS. That does not matter here, since 2.5.18 forbids Watch ads anyway **(unverified, SDK platform list not re-checked)**.

### Non-consumable "remove ads" with StoreKit 2

- **Entitlement check:** `Transaction.currentEntitlements` emits "a transaction for each non-consumable" the customer is entitled to and leaves out refunded or revoked ones. It is available on iOS 15+ and watchOS 8+ ([currentEntitlements][ce]).
- **Verification:** StoreKit verifies the JWS automatically and returns a `VerificationResult`. Unlock only on `.verified`. For more control, verify the `jwsRepresentation` on a server with the App Store Server Library ([VerificationResult][vr]). On-device verification is usually enough for removing ads.
- **Live updates:** start a `Task` at launch that iterates `Transaction.updates`. This catches purchases made on other devices, Ask to Buy, and offer codes, and delivers unfinished transactions ([updates][updates]).
- **Restore:** a reinstall or a new device gets transactions automatically. Still include a Restore Purchases button that calls `AppStore.sync()`, which prompts for App Store sign-in and should only run on an explicit user action ([sync][sync], 3.1.1 [Guidelines][guidelines]).
- **Refunds:** revoked transactions drop out of `currentEntitlements`, so re-checking at launch and on `updates` brings ads back after a refund ([currentEntitlements][ce]).
- **Apple Watch:** the Watch shows no ads (2.5.18), so it never needs the "remove ads" entitlement. If the Watch ever needs it, `currentEntitlements` works on watchOS 8+. Whether a companion Watch app sees the iPhone app's non-consumable purchases without extra setup is **(unverified)**.
- Family Sharing for the non-consumable is optional and set in App Store Connect **(unverified on these pages)**.

## Sources

[guidelines]: https://developer.apple.com/app-store/review/guidelines/
[hk-privacy]: https://developer.apple.com/documentation/healthkit/protecting-user-privacy
[att]: https://developer.apple.com/documentation/apptrackingtransparency
[att-req]: https://developer.apple.com/documentation/apptrackingtransparency/attrackingmanager/requesttrackingauthorization(completionhandler:)
[privacy-details]: https://developer.apple.com/app-store/app-privacy-details/
[manifest]: https://developer.apple.com/documentation/bundleresources/privacy-manifest-files
[sdk-reqs]: https://developer.apple.com/support/third-party-SDK-requirements/
[ce]: https://developer.apple.com/documentation/storekit/transaction/currententitlements
[updates]: https://developer.apple.com/documentation/storekit/transaction/updates
[sync]: https://developer.apple.com/documentation/storekit/appstore/sync()
[vr]: https://developer.apple.com/documentation/storekit/verificationresult
[gma-disclosure]: https://developers.google.com/admob/ios/privacy/data-disclosure
[admob-privacy]: https://developers.google.com/admob/ios/privacy
[ios14]: https://developers.google.com/admob/ios/ios14
[us-states]: https://developers.google.com/admob/ios/privacy/us-states
[targeting]: https://developers.google.com/admob/ios/targeting
[quickstart]: https://developers.google.com/admob/ios/quick-start
[banner]: https://developers.google.com/admob/ios/banner
[cmp]: https://support.google.com/admob/answer/13554116

- App Store Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- HealthKit, Protecting user privacy: https://developer.apple.com/documentation/healthkit/protecting-user-privacy
- App Tracking Transparency: https://developer.apple.com/documentation/apptrackingtransparency
- requestTrackingAuthorization: https://developer.apple.com/documentation/apptrackingtransparency/attrackingmanager/requesttrackingauthorization(completionhandler:)
- App privacy details: https://developer.apple.com/app-store/app-privacy-details/
- Privacy manifest files: https://developer.apple.com/documentation/bundleresources/privacy-manifest-files
- Third-party SDK requirements: https://developer.apple.com/support/third-party-SDK-requirements/
- StoreKit Transaction.currentEntitlements: https://developer.apple.com/documentation/storekit/transaction/currententitlements
- StoreKit Transaction.updates: https://developer.apple.com/documentation/storekit/transaction/updates
- StoreKit AppStore.sync(): https://developer.apple.com/documentation/storekit/appstore/sync()
- StoreKit VerificationResult: https://developer.apple.com/documentation/storekit/verificationresult
- Google Mobile Ads data disclosure: https://developers.google.com/admob/ios/privacy/data-disclosure
- AdMob iOS privacy (UMP): https://developers.google.com/admob/ios/privacy
- AdMob iOS 14+ / ATT: https://developers.google.com/admob/ios/ios14
- AdMob US state privacy: https://developers.google.com/admob/ios/privacy/us-states
- AdMob targeting: https://developers.google.com/admob/ios/targeting
- AdMob quick start: https://developers.google.com/admob/ios/quick-start
- AdMob banner ads: https://developers.google.com/admob/ios/banner
- AdMob certified CMP requirement: https://support.google.com/admob/answer/13554116
