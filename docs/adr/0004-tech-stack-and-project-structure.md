# Tech stack: SwiftUI, Swift 6, MVVM over SwiftData services, shared models in a submodule

Both apps are SwiftUI throughout; UIKit appears only inside small wrapper views for things SwiftUI can't do, such as the AdMob banner. Every target uses Swift 6 language mode with strict concurrency checking from the first commit, because Live Workout changes arrive from two devices on background delegate threads, which is where data races hide. The apps use MVVM: each screen has a view model, and view models reach data only through services that wrap SwiftData (they own the `ModelContext`), so views never touch SwiftData or use `@Query` directly. The code shared by the iPhone and Watch apps (the SwiftData models, the Workout rules, and the change messages the two devices exchange for a Live Workout under ADR-0003) lives in its own repository, pulled into this one as a git submodule and consumed as Swift packages. Everything used by only one app stays in that app. The Xcode project is a plain, checked-in `.xcodeproj`; app code lives in this repo.

## Considered Options

- **Plain SwiftUI + `@Observable` with `@Query` in views**: less code, but rejected in favour of MVVM with a service layer, so data access has one place to live and test.
- **TCA**: rejected as a large dependency that fights SwiftData.
- **A local package inside this repo instead of a submodule**: simpler (one repo, no submodule pinning) but rejected; the shared models get their own repository.
- **XcodeGen / Tuist**: rejected; a plain project plus packages keeps project-file conflicts rare without another tool.

## Consequences

- **Dependencies:** none by default. The only v1 third-party libraries are Google Mobile Ads and Google's consent library (UMP), pinned exactly via Swift Package Manager and confined to the iPhone app target so they can't reach Workout or Health data. No third-party analytics or crash reporting; use MetricKit and Xcode Organizer. Any new dependency needs its own ADR.
- **Submodule cost:** changing a shared model means a commit in the shared repo, then bumping the submodule pointer here; clones need `--recurse-submodules`.
- **Where backend code lives** is left to "Choose the Community backend and account-free identity".
