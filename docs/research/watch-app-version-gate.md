# iPhone / Watch app version gate

Research question: to support only **one app version of skew** between the iPhone app and the Watch app, with a **hard gate** beyond that ("You must update the Watch app" or "You must update the iPhone app", and changes wait until the update), what supported mechanisms exist to (1) learn whether the counterpart app is installed and which version it runs, (2) know this while the Workout Connection is Disconnected, (3) get Watch app updates to users, (4) open the App Store or trigger an update from the app, and (5) stay within the App Review Guidelines?

Targets are iOS 17 / watchOS 10 minimum, with the Watch app shipped in the same App Store release as the iPhone app and able to run standalone. Vocabulary follows `CONTEXT.md` (**Workout**, **Live**, **Workout Connection**, **Connected** / **Disconnected**). Related notes: [`watch-workout-and-sync.md`](watch-workout-and-sync.md) (WatchConnectivity semantics) and [`automatic-watch-join.md`](automatic-watch-join.md).

Sources were read on 2026-09-23, when iOS 27 / watchOS 27 are current. Claims marked **(unverified)** have no first-party source.

## Answer

**Facts**

1. **No first-party API reports the counterpart app's version.** `WCSession` exposes only install and pairing facts: `isPaired`, `isWatchAppInstalled` and `isComplicationEnabled` on iPhone, and `isCompanionAppInstalled` on the Watch, which applies to independent Watch apps only ([WCSession topics][wcsession]). HealthKit mirroring (`sendToRemoteWorkoutSession(data:)`) carries opaque `Data` ([doc][mirror-send]). Each side has to **send its own version**, which it reads from `Bundle.main` (`CFBundleShortVersionString` / `CFBundleVersion`).
2. **Both apps always build with the same marketing version.** Xcode refuses the build otherwise: "The value of CFBundleShortVersionString in your WatchKit app's Info.plist … does not match the value in your companion app's Info.plist … These values are required to match" (build error quoted in [forum 699703][f699703] and [forum 702394][f702394]). Skew therefore only appears **after installation**, because the two devices update separately.
3. **The devices update separately.** Since iOS 13 / watchOS 6, "each device is going to download its own app. So the iPhone gets an iPhone app. The watch gets a Watch App" ([WWDC19 208][wwdc19-208]). Updating the iPhone app does not carry the Watch app with it. The Watch app updates through the Watch's own App Store, subject to the Watch's **Automatic Updates** setting ([Apple Support 102629][s-102629], [Watch guide: Get apps][w-apps]). Apple documents no lag between the two **(unverified)**.
4. **Offline, you only know the last version you heard.** `updateApplicationContext` keeps the latest dictionary, and the receiver can read it at any time from `receivedApplicationContext` ([doc][wc-recvctx]). On iPhone, `isWatchAppInstalled` is valid whenever the session is activated ([doc][wc-installed]). Neither side can learn that the other has **since updated** until a message arrives.
5. **No supported way exists to trigger or deep-link a Watch app update.** `SKStoreProductViewController`, `SKOverlay` and `appStoreOverlay` are iOS/iPadOS/Mac only, not watchOS ([SKStoreProductViewController][skpvc], [SKOverlay][skoverlay], [appStoreOverlay][overlay]). On watchOS, `openSystemURL` is documented for phone calls and messages only ([doc][opensystemurl]). An Apple engineer said SwiftUI `Link` on watchOS opens Universal Links that launch apps, or `tel:` ([forum 650324][f650324]). Whether an `apps.apple.com` link opens the Watch App Store is **(unverified)**. StoreKit 2 `AppStore` on watchOS has no product-page or update API ([AppStore][appstore]). On **iPhone**, you can present the iPhone app's product page. **Instructions are the only tool for the Watch side.**
6. **App Review:** no guideline forbids a "please update" gate, but two lines constrain its scope. 3.2.2(x): "Apps must not force users to rate the app, review the app, download other apps, or other store-related actions in order to access functionality". 4.2.3(i): "Your app should work on its own without requiring installation of another app to function" ([Guidelines][guidelines]). Whether a forced update of the same app counts as a "store-related action" is **(unverified)**.

**Recommendation (labelled as such; the decision is yours)**

- **Version handshake:** each side puts `{appVersion, build, syncVersion}` (a small integer `syncVersion` that you bump only when the change-message format changes) into (a) its `updateApplicationContext` dictionary, re-sent on every launch and every WCSession activation, and (b) the header of every change message on both channels (mirroring `Data` and WatchConnectivity). Base the gate on `syncVersion`, and treat "one app version of skew" as "counterpart `syncVersion` within ±1 of mine". A release that doesn't change the protocol then never triggers a gate. **The handshake and gate logic must ship in version 1.** Because older sides need to recognise newer versions, each side must decide for itself that "counterpart > mine + 1 means **I** am out of date".
- **Scope the gate narrowly:** block only the exchange of changes (queue them and show the gate screen at the point where cross-device work happens), not the whole app. The Watch keeps running standalone and the iPhone keeps working alone. This matches 4.2.3(i) and lowers 3.2.2(x) risk. Never gate a **Live** Workout midway: finish it locally and hold its changes.
- **Offline:** gate on the **last-known** counterpart version, and clear the gate as soon as any message with a newer version arrives. Say in the UI that the version is "last seen".
- **Gate screen instructions** (exact Apple wording, current releases):
  - *Update the Watch app (on Apple Watch):* "Open the **App Store** on your Apple Watch, scroll to the bottom, tap **Account**, then tap **Updates**. Tap **Update** next to [App]." To avoid this next time: "On your Apple Watch, open **Settings** > **App Store** and turn on **Automatic Updates**", or "On iPhone, open the **Watch** app > **My Watch** > **App Store** and turn on **Automatic Updates**." ([102629][s-102629], [Watch guide][w-apps]). The Watch needs an internet connection (via iPhone, Wi-Fi or cellular) to update; this is an inference, not documented **(unverified)**.
  - *Update the iPhone app:* iOS 26–27: "Open the **App Store**, tap your picture (top right), tap **App Updates**, then tap **Update** next to [App]." iOS 17–18: "Open the **App Store**, tap your picture (top right), scroll down, then tap **Update** next to [App]." Automatic updates: iOS 18–27 **Settings > Apps > App Store > App Updates**; iOS 17 **Settings > App Store > App Updates** ([iPhone guide: Update apps][i-update], versioned pages). On iPhone, add a button that opens the product page (`SKStoreProductViewController`).
- **Risk to design for:** if a release raises the minimum watchOS, Watches that can't upgrade watchOS can never satisfy the gate, and the iPhone app must not force its own skew past them. How the App Store serves "last compatible version" to such Watches is **(unverified)**. Either don't raise the minimum watchOS casually, or exempt that case (show "Your Apple Watch needs watchOS X" instead).

## Detail

### 1. Detecting the counterpart app and its version

| Mechanism | Side | What it tells you | Source |
|---|---|---|---|
| `WCSession.isPaired` | iPhone | "whether the current iPhone has a paired Apple Watch"; "valid only for a configured session that has been activated successfully" | [isPaired][wc-paired] |
| `WCSession.isWatchAppInstalled` | iPhone | "whether the currently paired and active Apple Watch has installed the app"; same activation caveat | [isWatchAppInstalled][wc-installed] |
| `sessionWatchStateDidChange(_:)` | iPhone | Called when `isPaired`, `isWatchAppInstalled`, `isComplicationEnabled` or `watchDirectoryURL` changes. It fires on install or uninstall, not on **update** | [doc][wc-statechange] |
| `WCSession.isCompanionAppInstalled` + `sessionCompanionAppInstalledDidChange(_:)` | Watch | "whether the paired iPhone has installed the app"; "only valid on independent watchOS apps" (our Watch app runs standalone, so it should be independent) | [isCompanionAppInstalled][wc-companion], [doc][wc-companionchange] |
| `WCSession.isReachable` | both | For live messaging only; the Watch app must be running in the foreground or high-priority background | [isReachable][wc-reachable] |
| `updateApplicationContext(_:)` / `receivedApplicationContext` | both | Latest-state dictionary. It "replaces the previous dictionary", may be called "when the counterpart is not currently reachable", and is delivered "when the opportunity arises" | [update][wc-updatectx], [received][wc-recvctx] |
| `sendMessage(_:replyHandler:errorHandler:)` | both | Immediate request and reply, used when **Connected**. From iOS it "does not wake up the corresponding WatchKit extension" | [doc][wc-sendmsg] |
| `transferUserInfo(_:)` | both | Queued, in-order, guaranteed delivery. Good for queued changes | [doc][wc-userinfo] |
| `HKWorkoutSession.sendToRemoteWorkoutSession(data:completion:)` | both (iOS 17 / watchOS 10) | Opaque `Data` between primary and mirrored sessions; carries no version of its own | [doc][mirror-send] |
| `Bundle.main` Info.plist keys | self | Your own `CFBundleShortVersionString` / `CFBundleVersion`, to send to the other side | [Bundle.main][bundle] |

The WCSession topic list ("Getting the Paired Device Information") contains no version property ([WCSession][wcsession]). No first-party API exposes the counterpart's version, so a handshake is required.

**Build-time guarantee.** A combined build cannot ship mismatched marketing versions. Xcode fails with the "required to match" error above ([forum 699703][f699703]). An Apple-flagged answer notes that since Xcode 13.3 the companion target should use the `MARKETING_VERSION` / `CURRENT_PROJECT_VERSION` build settings ([forum 702394][f702394]; a forum answer, not reference documentation).

### 2. What can be known while Disconnected

- **iPhone:** `isPaired` and `isWatchAppInstalled` are readable once the session is activated ([isWatchAppInstalled][wc-installed]). [`automatic-watch-join.md`](automatic-watch-join.md) notes they come from the iPhone's own records, so they hold even when the Watch is off. The counterpart's version is only as fresh as the last `receivedApplicationContext` ([doc][wc-recvctx]).
- **Watch:** `isCompanionAppInstalled` is available for independent apps ([doc][wc-companion]). Otherwise the Watch has only the last received context.
- Apple says WatchConnectivity "isn't always available" and should be "an opportunistic optimization" ([Keeping your watchOS app's content up to date][keeping]), and that an independent Watch app "can't use Watch Connectivity as its main source of data" ([Creating independent watchOS apps][independent]). So a gate decided while Disconnected must be treated as provisional.
- **Neither side can detect "the other side just updated" until it receives a message.** The new version's first launch should call `updateApplicationContext` immediately. That wakes the counterpart in the background with a `WKWatchConnectivityRefreshBackgroundTask` (see [`automatic-watch-join.md`](automatic-watch-join.md)), which clears the gate as soon as delivery happens. Delivery timing is not documented **(unverified)**.

### 3. How Watch app updates reach users

- **Separate delivery.** WWDC19 208: "the App Store server is going to install whatever it needs to install wherever it needs to install it … each device is going to download its own app" ([WWDC19 208][wwdc19-208]). Apple's docs: "The system downloads and installs the watchOS app directly to Apple Watch for both dependent and independent apps. However, the user can't launch a dependent watchOS app until the iOS app finishes installing on iPhone" ([Creating independent watchOS apps][independent]).
- **Automatic updates on the Watch (watchOS 10–27):** "Go to the Settings app on your Apple Watch. Tap App Store … *Automatic Updates:* Automatically download new versions of your apps when they're available" ([Watch guide, watchOS 27][w-apps]; the same path is on the [watchOS 10 page][w-apps-10]). The same toggle is in the iPhone **Watch** app: "Open the Watch app on your iPhone, scroll to App Store and tap it, then turn on or turn off Automatic Updates" ([102629][s-102629]); the watchOS 27 guide words it as "tap My Watch, then go to App Store" ([Watch guide][w-apps]). The watchOS 10/11 guides add: "To get the most recent versions of your Apple Watch apps, make sure Automatic Updates is also turned on" ([watchOS 10][w-apps-10], [watchOS 11][w-apps-11]).
- **Manual update on the Watch:** "Open the App Store and scroll down to the bottom. Tap Account, then tap Updates. Tap Update next to an app to update only that app, or tap Update All" ([102629][s-102629]). This article is not versioned. I found no per-version Watch guide page for manual updates, so wording differences on watchOS 10–11 are **(unverified)**.
- **Automatic install (not update):** by default, "Apple Watch automatically installs the watch-compatible versions of your iPhone apps (if available)". Manual install is under iPhone Watch app > **My Watch** > **Available Apps** > **Install** (watchOS 27 guide says My Watch > Apps > Available Apps; watchOS 10–26 say My Watch > Available Apps, with the **Automatic App Install** toggle under My Watch > General) ([watchOS 27][w-apps], [watchOS 26][w-apps-26], [watchOS 10][w-apps-10]). Use this for the "Watch app not installed" case.
- **iPhone update paths, by version** ([iPhone guide: Update apps][i-update]):

| iOS | Manual | Automatic toggle |
|---|---|---|
| 17 | "Open the App Store app … Tap [account] or your picture at the top right. Scroll down, then tap Update next to apps you want to update, or tap Update All." | "Go to Settings > App Store. Turn off App Updates." ([17.0 page][i-update-17]) |
| 18 | Same as 17 ("Scroll down, then tap Update …") | "Go to Settings > Apps > App Store. Turn off App Updates." ([18.0 page][i-update-18]) |
| 26 | "Tap [account] or your picture at the top right. Tap App Updates, then tap Update …" | Settings > Apps > App Store > App Updates ([26 page][i-update-26]) |
| 27 | "Tap [account] or your picture. Tap App Updates, then tap Update …" | Settings > Apps > App Store > App Updates ([current page][i-update]) |

- iPhone App Store updates are "automatically updated by default" ([iPhone guide][i-update]). Apple doesn't state the Watch **Automatic Updates** default. Whether the iPhone's **App Updates** list includes Watch apps is **(unverified)**.
- **Lag:** Apple publishes no figure for how soon an automatic update installs on either device **(unverified)**. Plan for skew lasting days. Apple itself sometimes repackages Watch apps ("This app has been updated by Apple to prepare for watchOS 27 compatibility") ([9to5Mac, Aug 2026][9to5], secondary). Such an Apple re-issue changes the build but presumably not `syncVersion` **(unverified)**, which is another reason to gate on `syncVersion`.

### 4. Deep-linking or triggering an update

| Mechanism | watchOS? | Notes | Source |
|---|---|---|---|
| `SKStoreProductViewController` | No (iOS, iPadOS, Mac Catalyst, macOS) | Use it on iPhone to show our own product page | [doc][skpvc] |
| `SKOverlay` / `.appStoreOverlay` | No (iOS, iPadOS, Mac Catalyst, visionOS) | For recommending apps | [SKOverlay][skoverlay], [overlay][overlay] |
| `AppStore` (StoreKit 2) | watchOS 8+ | Subscriptions, review requests, offer codes, sync; no product page or update API | [AppStore][appstore] |
| `WKApplication.openSystemURL(_:)` | watchOS 7+ | "Use this method to initiate phone calls or send messages" | [doc][opensystemurl] |
| SwiftUI `Link` / `openURL` | watchOS 7+ | Apple engineer: "On watchOS, Link can open Universal Links that launch apps on your watch. It also works with other URL schemes like tel:"; web URLs "not supported" | [OpenURLAction][openurl], [forum 650324][f650324] |
| `itms-apps://` or `https://apps.apple.com/app/id…` on watchOS | **(unverified)** | No Apple documentation. Worth a quick on-device test, but don't rely on it | — |
| Programmatically installing or updating | No | Guideline 2.5.2 forbids downloading or installing code that "changes features or functionality of the app" | [Guidelines][guidelines] |

On iPhone, a product-page button is fine. There is no Apple-documented URL that opens the iPhone **Watch** app or the Watch's App Store **(unverified)**.

Checking the latest store version is optional: the iTunes Search API supports lookup by iTunes ID (not documented by bundle ID) and is "limited to approximately 20 calls per minute" ([Search API][searchapi]). The gate doesn't need it, because it compares against the counterpart, not the store.

### 5. App Review considerations

- **3.2.2(x):** "Apps must not force users to rate the app, review the app, download other apps, or other store-related actions in order to access functionality, content, or use of the app." ([Guidelines][guidelines]). A gate that blocks *all* use until the user updates arguably touches this. Many shipping apps have forced-update screens, but I found no Apple statement either way **(unverified)**. Blocking only the cross-device exchange keeps the app usable.
- **4.2.3(i):** "Your app should work on its own without requiring installation of another app to function." ([Guidelines][guidelines]). The iPhone app must remain useful without the Watch app, and the gate must not become "install or update the Watch app to use the iPhone app".
- **2.5.2:** there is no self-updating or downloaded code, so the update must go through the App Store ([Guidelines][guidelines]).
- **2.1 / 4 intro:** App Review expects a complete, working app, and apps "that stop working or offer a degraded experience may be removed" ([Guidelines][guidelines]). Reviewers get matched versions from one submission, so they will never see the gate. Consider a debug switch and a line in the Review Notes explaining it.
- **2.3.12:** "What's New" must describe significant changes ([Guidelines][guidelines]). A release that bumps `syncVersion` can say "Update both your iPhone and Apple Watch app".

## Sources

[wcsession]: https://developer.apple.com/documentation/watchconnectivity/wcsession
[wc-paired]: https://developer.apple.com/documentation/watchconnectivity/wcsession/ispaired
[wc-installed]: https://developer.apple.com/documentation/watchconnectivity/wcsession/iswatchappinstalled
[wc-companion]: https://developer.apple.com/documentation/watchconnectivity/wcsession/iscompanionappinstalled
[wc-statechange]: https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate/sessionwatchstatedidchange(_:)
[wc-companionchange]: https://developer.apple.com/documentation/watchconnectivity/wcsessiondelegate/sessioncompanionappinstalleddidchange(_:)
[wc-reachable]: https://developer.apple.com/documentation/watchconnectivity/wcsession/isreachable
[wc-updatectx]: https://developer.apple.com/documentation/watchconnectivity/wcsession/updateapplicationcontext(_:)
[wc-recvctx]: https://developer.apple.com/documentation/watchconnectivity/wcsession/receivedapplicationcontext
[wc-sendmsg]: https://developer.apple.com/documentation/watchconnectivity/wcsession/sendmessage(_:replyhandler:errorhandler:)
[wc-userinfo]: https://developer.apple.com/documentation/watchconnectivity/wcsession/transferuserinfo(_:)
[mirror-send]: https://developer.apple.com/documentation/healthkit/hkworkoutsession/sendtoremoteworkoutsession(data:completion:)
[bundle]: https://developer.apple.com/documentation/foundation/bundle/main
[independent]: https://developer.apple.com/documentation/watchos-apps/creating-independent-watchos-apps
[keeping]: https://developer.apple.com/documentation/watchos-apps/keeping-your-watchos-app-s-content-up-to-date
[wwdc19-208]: https://developer.apple.com/videos/play/wwdc2019/208
[f699703]: https://developer.apple.com/forums/thread/699703
[f702394]: https://developer.apple.com/forums/thread/702394
[f650324]: https://developer.apple.com/forums/thread/650324
[skpvc]: https://developer.apple.com/documentation/storekit/skstoreproductviewcontroller
[skoverlay]: https://developer.apple.com/documentation/storekit/skoverlay
[overlay]: https://developer.apple.com/documentation/swiftui/view/appstoreoverlay(ispresented:configuration:)
[appstore]: https://developer.apple.com/documentation/storekit/appstore
[opensystemurl]: https://developer.apple.com/documentation/watchkit/wkapplication/opensystemurl(_:)
[openurl]: https://developer.apple.com/documentation/swiftui/openurlaction
[guidelines]: https://developer.apple.com/app-store/review/guidelines/
[searchapi]: https://performance-partners.apple.com/search-api
[s-102629]: https://support.apple.com/en-us/102629
[w-apps]: https://support.apple.com/guide/watch/get-apps-apd99e3c6a68/watchos
[w-apps-26]: https://support.apple.com/guide/watch/apd99e3c6a68/26/watchos/26
[w-apps-11]: https://support.apple.com/guide/watch/apd99e3c6a68/11.0/watchos/11.0
[w-apps-10]: https://support.apple.com/guide/watch/get-apps-apd99e3c6a68/10.0/watchos/10.0
[i-update]: https://support.apple.com/guide/iphone/update-apps-iph98709f167/ios
[i-update-26]: https://support.apple.com/guide/iphone/iph98709f167/26/ios/26
[i-update-18]: https://support.apple.com/guide/iphone/iph98709f167/18.0/ios/18.0
[i-update-17]: https://support.apple.com/guide/iphone/iph98709f167/17.0/ios/17.0
[9to5]: https://9to5mac.com/2026/08/27/apple-is-automatically-updating-some-apple-watch-apps-for-watchos-27-compatibility/

- WCSession and delegate reference: [WCSession][wcsession], [isPaired][wc-paired], [isWatchAppInstalled][wc-installed], [isCompanionAppInstalled][wc-companion], [sessionWatchStateDidChange][wc-statechange], [sessionCompanionAppInstalledDidChange][wc-companionchange], [isReachable][wc-reachable], [updateApplicationContext][wc-updatectx], [receivedApplicationContext][wc-recvctx], [sendMessage][wc-sendmsg], [transferUserInfo][wc-userinfo]
- HealthKit: [sendToRemoteWorkoutSession][mirror-send]
- watchOS app structure: [Creating independent watchOS apps][independent], [Keeping your watchOS app's content up to date][keeping], [WWDC19 208][wwdc19-208]
- Build versioning (forum; Xcode error text): [699703][f699703], [702394][f702394]
- Store UI: [SKStoreProductViewController][skpvc], [SKOverlay][skoverlay], [appStoreOverlay][overlay], [AppStore][appstore], [openSystemURL][opensystemurl], [OpenURLAction][openurl], [forum 650324][f650324], [iTunes Search API][searchapi]
- Apple Support: [102629 Manually update apps][s-102629], [Watch guide: Get apps (27)][w-apps], [(26)][w-apps-26], [(11)][w-apps-11], [(10)][w-apps-10], [iPhone guide: Update apps (27)][i-update], [(26)][i-update-26], [(18)][i-update-18], [(17)][i-update-17]
- [App Review Guidelines, last updated June 8, 2026][guidelines]
- Secondary: [9to5Mac on Apple-reissued Watch apps][9to5]
