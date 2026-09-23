# Liquid Glass availability and switchable Style

Research for issue #6. Sources checked 2026-09-23 (iOS 27 / watchOS 27 are shipping; Xcode 27 is current). Terms follow `CONTEXT.md`: **Style** is **Classic** or **Glass**.

## Answer

- **Glass on iOS 26+ / watchOS 26+ is first-party.** SwiftUI has `glassEffect(_:in:)`, `Glass` (`.regular`, `.clear`, `.identity`, `.tint`, `.interactive`), `GlassEffectContainer`, `glassEffectID` / `glassEffectUnion` / `glassEffectTransition`, and the `.glass` / `.glassProminent` button styles. All are available on iOS 26.0 and watchOS 26.0. [1][2][3][4][5]
- **System controls switch to Glass automatically** when you build with the iOS 26+ SDK. This covers bars, tab bars, sheets, popovers, menus, toggles and sliders. On watchOS the change appears even without rebuilding. [6][8]
- **There is no durable way to get a native Classic look on iOS 26+.** `UIDesignRequiresCompatibility` is a temporary Info.plist key for the whole app, and the system **ignores it when you build with the iOS 27 SDK**. It is not listed for watchOS. It is a bundle key, so it is set at build time and can't be switched at runtime. [7]
- **Glass on iOS 17–25 can only be approximated**, by layering `Material` (blur plus vibrancy), a tint, a highlight stroke and a shadow. Refraction, lensing and the fluid morphing have no public equivalent before iOS 26. [9][10]
- **Accessibility and performance:** read `accessibilityReduceTransparency` and switch to opaque fills in the custom Glass path. Standard components adapt on their own. Keep the number of glass or blur surfaces small, group glass with `GlassEffectContainer`, and never give a `UIVisualEffectView` an alpha below 1. [6][10][11][12]
- **Structure:** set a `Style` environment value with `@Entry` near the root, and give each role one modifier or `ButtonStyle` (surface, primary button, bar) that branches on Style and `#available`. Views never branch themselves. [13]

> **Warning about the "switchable on every version" decision.** Glass on iOS 17–25 is feasible as an approximation. **Classic on iOS 26/27 is not natively achievable.** Once the app builds with Xcode 27, which App Store rules will eventually require, every system-owned control renders as Liquid Glass whatever the app's Style is. This includes navigation bars, tab bars, sheets, menus, alerts, toggles and sliders. Classic on iOS 26+ can only restyle **app-owned** surfaces: custom buttons, cards, backgrounds, and possibly custom-drawn bars. The same limit applies on watchOS 26+, where there is no opt-out key at all. The Style decision either needs rewording ("Classic on 26+ = Classic-flavoured custom surfaces within system Glass chrome") or needs custom replacements for system chrome, which Apple advises against. [6][7][8]

## Detail

### 1. Liquid Glass APIs (iOS 26 / watchOS 26)

| API | What it does | Availability |
|---|---|---|
| `View.glassEffect(_:in:)` | Draws a Liquid Glass shape behind a view and applies glass foreground effects. Defaults to `.regular` in a `Capsule`. | iOS/watchOS 26.0 [1] |
| `Glass` (`.regular`, `.clear`, `.identity`, `.tint(_:)`, `.interactive(_:)`) | Configures the material. `.identity` leaves the content "unaffected as if no glass effect was applied", which lets you switch glass off without changing the view tree. `.clear` needs a dimming layer to stay legible. | iOS/watchOS 26.0 [3][14] |
| `GlassEffectContainer(spacing:)` | Renders several glass shapes together "improving rendering performance", and lets them blend and morph. | iOS/watchOS 26.0 [2] |
| `glassEffectID(_:in:)`, `glassEffectUnion(id:namespace:)`, `glassEffectTransition(_:)` (`.matchedGeometry`, `.materialize`) | Morphing and unions inside a container. | iOS 26.0 [9] |
| `PrimitiveButtonStyle.glass` / `.glassProminent` | Glass button styles. `.glassProminent` corresponds to `.borderedProminent`. | iOS/watchOS 26.0 [4][5] |
| `UIGlassEffect` (UIKit) | Glass for `UIVisualEffectView`. | iOS 26.0, **not watchOS** [15] |

WWDC25 "Build a SwiftUI app with the new design" says glass "can not sample other glass", so nearby glass elements belong in one container. [8]

HIG: use Liquid Glass only for the **functional layer** (controls and navigation), not the content layer, where standard materials remain the recommendation. Use it sparingly. [10]

### 2. Automatic adoption

- "If your app uses standard components from SwiftUI, UIKit, or AppKit, your interface picks up the latest look and feel … build your app with the latest SDKs." Bars, sheets, popovers and controls "automatically adopt this material". [6]
- Apple tells apps to remove custom backgrounds on bars, sheets and popovers because they "might overlay or interfere with Liquid Glass". [6][8]
- **watchOS:** "Liquid Glass changes are minimal in watchOS, so they appear automatically when you open your app on the latest release even if you don't build against the latest SDK." Apple advises adopting the standard toolbar APIs and button styles from watchOS 10. [6]

### 3. Opting out: `UIDesignRequiresCompatibility`

From the key's documentation [7]:
- "A Boolean value that indicates whether the system runs the app using a compatibility mode for UI." If `YES`, the app displays "as it looks when built against previous versions of the SDKs".
- "**Temporarily** use this key while reviewing and refining your app's UI."
- "The system **ignores this key when you build for iOS 27 or later**, iPadOS 27 or later, Mac Catalyst 27 or later, macOS 27 or later, or tvOS 27 or later."
- Platforms listed: iOS, iPadOS, macOS and tvOS 26.0. **watchOS is not listed.**

Consequences:
- **Per-app:** yes. It is a single Info.plist key for the whole process.
- **Temporary:** yes, and already ended for any app built with Xcode 27.
- **Runtime-switchable:** no. *(Inference, unverified: Info.plist is part of the signed bundle and is read by the system. Apple documents no per-window or per-scene toggle.)*
- The App Store will eventually require the iOS 27 SDK. Secondary sources say around April 2027 [16]. *(Unverified: Apple has not published a date that I found. This fits Apple's usual yearly SDK requirement.)*
- iOS 27 also reportedly adds a user-facing clear-to-tinted glass slider in Settings [17]. *(Secondary source.)* Apple's own docs already say people "can choose a preferred look for Liquid Glass in their device's settings". [6][10]

### 4. Approximating Glass on iOS 17–25 (and watchOS 10–11)

- **`Material`** (`.ultraThin`, `.thin`, `.regular`, `.thick`, `.ultraThick`, `.bar`). This is platform blur blending that looks like "heavily frosted glass". Foreground content gets **vibrancy** automatically unless a custom `foregroundStyle` is set (hierarchical styles keep it). Materials blur only the app's own content. Available from iOS 15 / watchOS 8. [11]
- **UIKit:** `UIVisualEffectView` with `UIBlurEffect` and `UIVibrancyEffect`. The HIG lists these as the "standard materials". [10][12]
- **Credible recipe (design inference, not an Apple recipe):** `background(.ultraThinMaterial, in: Capsule())`, plus a faint white gradient `strokeBorder` for the specular edge, a soft shadow, and an optional `tint.opacity(…)` overlay. Refraction, lensing, light response to motion, and morphing between shapes are not reproducible with public APIs before iOS 26. *(Unverified that any Apple source endorses emulation. Apple only documents the two material families.)*
- **Performance:** `UIVisualEffectView` with alpha below 1 on itself or any superview forces an offscreen pass, and effects "look incorrect or not show up at all". Snapshots must be taken of the window. [12] Apple warns that many glass effects or containers "can degrade performance", so "limit the use of Liquid Glass effects onscreen at the same time". [9] Treat blur the same way: few surfaces, not per row in lists. *(The per-row blur cost is an inference.)*

### 5. Accessibility

- `accessibilityReduceTransparency`: "UI (mainly window) backgrounds should not be semi-transparent; they should be opaque." Available on iOS 13 and watchOS 6. [13b]
- System Glass components adapt on their own to Reduce Transparency, Increase Contrast, Reduce Motion and the user's preferred glass look. Custom elements must be tested with those settings. [6][10]
- In the custom Glass path on iOS 17–25, swap the material for an opaque system background (for example `Color(.secondarySystemBackground)`) when Reduce Transparency is on. Keep system vibrant or semantic label colours on materials. Avoid quaternary label on thin materials. [10]

### 6. Structuring a switchable Style without forking views

The recommended pattern is an inference built from documented primitives:

```swift
enum Style { case classic, glass }

extension EnvironmentValues {
    @Entry var appStyle: Style = .classic   // set at root from OS default + user setting
}

struct StyledSurface: ViewModifier {
    @Environment(\.appStyle) private var style
    @Environment(\.accessibilityReduceTransparency) private var reduceTransparency
    var shape: some Shape = RoundedRectangle(cornerRadius: 16)

    func body(content: Content) -> some View {
        switch style {
        case .classic:
            content.background(Color(.secondarySystemBackground), in: shape)
        case .glass:
            if reduceTransparency {
                content.background(Color(.secondarySystemBackground), in: shape)
            } else if #available(iOS 26, watchOS 26, *) {
                content.glassEffect(.regular, in: shape)
            } else {
                content.background(.ultraThinMaterial, in: shape) // + highlight stroke
            }
        }
    }
}
```

- `@Entry` creates the environment key [13]. Views apply `.modifier(StyledSurface())` and a Style-aware `ButtonStyle` (which maps to `.glass` / `.glassProminent` on 26+ [4][5] and to bordered or custom styles otherwise). Only these few "role" modifiers branch on Style and OS version.
- On 26+, `Glass.identity` [14] lets a single `glassEffect` call switch glass off without changing view identity. This helps with animated Style switches.
- **Limit:** this pattern only controls app-owned surfaces. Navigation bars, tab bars, sheets, alerts and system controls follow the SDK, not the Style (see the warning in the Answer). `toolbarBackgroundVisibility(_:for:)` (iOS 18+) [18] can change bar backgrounds, but Apple advises against custom bar backgrounds on 26+ [6]. *(Unverified whether a visible bar background on iOS 26 restores anything close to the Classic look.)*

## Sources

1. glassEffect(_:in:): https://developer.apple.com/documentation/swiftui/view/glasseffect(_:in:)
2. GlassEffectContainer: https://developer.apple.com/documentation/swiftui/glasseffectcontainer
3. Glass: https://developer.apple.com/documentation/swiftui/glass
4. PrimitiveButtonStyle.glass: https://developer.apple.com/documentation/swiftui/primitivebuttonstyle/glass
5. PrimitiveButtonStyle.glassProminent: https://developer.apple.com/documentation/swiftui/primitivebuttonstyle/glassprominent
6. Adopting Liquid Glass: https://developer.apple.com/documentation/technologyoverviews/adopting-liquid-glass
7. UIDesignRequiresCompatibility: https://developer.apple.com/documentation/bundleresources/information-property-list/uidesignrequirescompatibility
8. WWDC25 "Build a SwiftUI app with the new design": https://developer.apple.com/videos/play/wwdc2025/323/
9. Applying Liquid Glass to custom views: https://developer.apple.com/documentation/swiftui/applying-liquid-glass-to-custom-views
10. HIG, Materials: https://developer.apple.com/design/human-interface-guidelines/materials
11. SwiftUI Material: https://developer.apple.com/documentation/swiftui/material
12. UIVisualEffectView: https://developer.apple.com/documentation/uikit/uivisualeffectview
13. @Entry macro: https://developer.apple.com/documentation/swiftui/entry()
13b. accessibilityReduceTransparency: https://developer.apple.com/documentation/swiftui/environmentvalues/accessibilityreducetransparency
14. Glass.identity / Glass.clear: https://developer.apple.com/documentation/swiftui/glass/identity , https://developer.apple.com/documentation/swiftui/glass/clear
15. UIGlassEffect: https://developer.apple.com/documentation/uikit/uiglasseffect
16. (secondary) AppleInsider, "Liquid Glass will be mandatory in iOS 27": https://appleinsider.com/articles/26/03/26/stop-holding-out-hope-liquid-glass-will-be-mandatory-in-ios-27
17. (secondary) MacRumors, iOS 27 release: https://www.macrumors.com/2026/09/14/apple-releases-ios-27/
18. toolbarBackgroundVisibility(_:for:): https://developer.apple.com/documentation/swiftui/view/toolbarbackgroundvisibility(_:for:)
