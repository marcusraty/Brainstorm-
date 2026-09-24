# Live Workout exchange: acked changes while Live, endings carry the whole Workout, latest edit wins, the user reviews conflicting endings

While a Workout is Live, every change (a Set added, edited or deleted; an Exercise added or moved) is sent at once over the HealthKit mirroring channel with its own ID, and the other device acknowledges it; anything unacknowledged waits in a persistent outbox on the sending device and is resent after about 10 seconds or on reconnect, and duplicates are ignored by ID. Either device can do every action at any time. When the devices are apart, both keep logging into their outboxes, and banners say so (the iPhone from Apple's disconnect callback, the Watch when a send fails). On reconnect the outboxes are resent; if the same thing was changed on both devices while apart, the latest edit wins field by field on its timestamp (delete against edit included; a near-tie goes to the Watch). An ending (Finish or Discard, each after an "Are you sure?") goes through WatchConnectivity's guaranteed queue carrying the whole Workout as that device has it, and the receiving side reconciles once: edits combine with the latest winning; endings that disagree (Finish against Discard, or two different Finishes) mark the Workout **Needs review**, and the iPhone shows the two versions side by side for the user to choose, keeping everything until then. We chose this because the mirroring channel drops messages while the devices are apart and never tells the iPhone about a Watch-side ending, so something durable must carry endings; and because every automatic rule for conflicting endings (dropping late Sets, splitting them into a new Workout, earliest or latest ending wins, greying out Discard) either lost training or hid a surprising choice from the user.

## Considered Options

- **Mirroring channel only:** rejected; it doesn't queue across a disconnect (Apple header) and ends with the Watch session.
- **Guaranteed queue for every change:** rejected; Apple may delay its deliveries to save power, so the other screen would lag during a Workout.
- **Heartbeat / checksum:** rejected once acks and the outbox covered drift.
- **Reconciling only at the end with no acks:** rejected; the two screens would drift apart mid-Workout.
- **Automatic rules for conflicting endings** (drop Sets after the Finish; split them into a separate "Continuation" Workout with derived IDs; Discard only while empty or only while Connected; earliest or latest ending wins; Undo): rejected as hacky or lossy; the user reviews instead.
- **A shared "Workout time" clock:** dropped; with conflicts decided by review, timestamps only order Sets for display, and system clocks are good enough.
- **The Watch saving to Health only after reconciliation:** rejected; the live workout builder must be finished promptly, and holding the session open for hours risks losing the recording.

## Consequences

- **Health:** the Watch saves the Workout to Health when it ends, if it was in the Workout (otherwise the iPhone saves a plain record, ADR-0003). Nothing changes that Health record afterwards except (a) deleting it when a Discard becomes final, and (b) linking the effort rating. Heart rate and energy are never edited.
- **Finished Workouts' start and end times can't be edited**, in our app or Health. Sets, Exercises and notes still can (on the iPhone).
- **Unverified, pending "Device tests before building":** whether the iPhone can delete, and link effort to, a Workout the Watch saved (Apple only allows the saving app; whether our Watch app counts as the same app is undocumented). If not, the Watch does it on the iPhone's request, or effort is rated on the Watch.
- **Tidy-up:** the Watch deletes its stored copy of a Finished Workout once the iPhone confirms it has it (ADR-0001); this is storage only, never a permission the Watch waits for.
- Relies on both devices' clocks being set automatically (the default); a manually set clock could make "latest edit wins" pick oddly.
- Module and interface design for all of this is left to `/to-spec`.
