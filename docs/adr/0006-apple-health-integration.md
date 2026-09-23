# Apple Health: one tagged Workout record, minimal reads, permissions at onboarding

Every Workout is saved to Health as Traditional Strength Training, with the heart rate and active energy the Watch session collects and, where supported (iOS 18 / watchOS 11+), the effort rating. Who writes the record follows ADR-0003. The record runs from the Start tap to the Finish tap, even if heart rate has gaps. It carries our Workout UUID (`HKMetadataKeyExternalUUID`) plus a sync identifier and version (`HKMetadataKeySyncIdentifier` / `HKMetadataKeySyncVersion`), so upgrading a plain iPhone record to the Watch's full one, or editing a Finished Workout's times, re-saves with a higher version and Health replaces the record rather than duplicating it. Set details never go into Health: it has no concept of Sets, and metadata would expose our data to every app with Health access. We read only heart rate and energy for a Workout's time window (history charts), Apple's estimated effort (to pre-fill the rating), and date of birth (to estimate max heart rate). Heart-rate data is read from Health when shown, never copied into our database.

## Consequences

- **Permissions** are requested once, during onboarding, after a screen explaining why. Denial never blocks the app: Workouts still work without Health saving or heart rate, and screens say "No data. Check Health permissions", because an app can't tell whether read access was denied.
- **Heart-rate zones:** Apple's zones on iOS / watchOS 27+; below that, 5 zones as percentages of max heart rate, which is the user's entered value or else 220 − age.
- **Discard deletes** the heart-rate and energy samples our session wrote, so a discarded Workout doesn't count towards Activity rings.
- **Parked (out of scope for this map):** recovery / readiness (no resting HR, HRV or sleep reads, no background delivery) and body weight / measurements (no body-mass reads; bodyweight Exercises record only added or assisted weight).
