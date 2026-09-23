# SwiftData on the iPhone; a thin, local-only store on the Watch

The iPhone keeps the user's data (Routines, Workouts, Sets, Custom Exercises) in SwiftData. It is the only full copy on the user's devices and the one that will sync through iCloud. The Watch keeps a small SwiftData store of its own, never synced to iCloud and not a replica of the iPhone's. It holds only what it needs to run a Workout without the iPhone: the Routines, the Exercises they use and "last time" values, the Live Workout, and each Finished Workout until the iPhone confirms receipt, after which the Watch deletes it. We chose SwiftData over Core Data and over SQLite (GRDB) with CKSyncEngine for its ergonomics. We accept that if it uses SwiftData's built-in CloudKit sync, there are no unique constraints and every relationship is optional. So a record must only ever be created on one device, and the tickets on the Workout lifecycle and the Watch→iPhone handoff settle how.

## Consequences

- **Library Exercises** live in a separate, read-only, never-synced store bundled with the app. Routines and Workouts reference them loosely, by a stable library ID plus a copy of the Exercise name, so history survives the library renaming or dropping an Exercise.
- **Settings** (e.g. Style, units) live in iCloud key-value storage (`NSUbiquitousKeyValueStore`), not in the database. They follow the iCloud account, and first launch on each device can't seed duplicate rows. The Watch gets them from the iPhone with the Routines.
- Watch-standalone history browsing is limited to what the Watch holds; full history lives on the iPhone.
