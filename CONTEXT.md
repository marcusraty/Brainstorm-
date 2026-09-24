# Workout Tracker

A strength-training tracker for iPhone and Apple Watch, where the Watch can run a Workout on its own as well as alongside the iPhone.

## Language

### Training

**Routine**:
A reusable, named plan made of ordered Exercise Templates that a Workout can be started from.
_Avoid_: Program, plan, workout template

**Workout**:
One performed training session, from start to finish, optionally started from a Routine.
_Avoid_: Session, log, training

**Exercise**:
A named movement, such as Bench Press, that Sets are recorded against.
_Avoid_: Movement, lift

**Set**:
One recorded effort within a Workout Exercise, holding only the fields its Exercise's Measurement uses, in the unit the user entered.
_Avoid_: Rep, entry

**Exercise Template**:
One Exercise as planned inside a Routine: its position, notes and Target Sets.
_Avoid_: Routine exercise, template (alone)

**Target Set**:
A planned Set inside an Exercise Template, which becomes an empty Set when a Workout starts.
_Avoid_: Planned set, goal

**Workout Exercise**:
One Exercise as performed inside a Workout: its position, notes, optional superset group and Sets.
_Avoid_: Workout entry, exercise log

**Measurement**:
What an Exercise records per Set: weight × reps, reps only, weighted or assisted bodyweight, time, distance, or distance × time.
_Avoid_: Tracking type, metric

**Set Type**:
Whether a Set is normal, warm-up, drop or failure.
_Avoid_: Set kind, set tag

**Library Exercise**:
An Exercise bundled from the open-source exercise library; read-only to users.
_Avoid_: Built-in exercise, default exercise

**Custom Exercise**:
An Exercise the user created or added from the Community, private to that user and stored in their data.
_Avoid_: User exercise, personal exercise

**Community Exercise**:
A separate, public copy of a Custom Exercise that its creator has Published, which other users can find, add, Rate and Report.
_Avoid_: Shared exercise, public exercise

### Workout lifecycle

**Live**:
The state of a Workout from Start until it is Finished or Discarded; the same Live Workout is present on both iPhone and Watch.
_Avoid_: Active, running, in progress, started

**Finished**:
The final state of a Workout that was ended on either device; it is part of history and never becomes Live again.
_Avoid_: Ended, stopped, completed, saved

**Discard**:
Throwing away a Live Workout from either device, so that nothing of it is kept; one made while the devices were apart can be overturned in review if the other device ended the Workout differently.
_Avoid_: Cancel, delete, abandon

**Needs review**:
The mark on a Workout whose two devices ended it differently while apart, until the user chooses between the two versions on the iPhone.
_Avoid_: Conflict, merge, sync error

**Workout Connection**:
Whether the iPhone and Watch can currently exchange a Live Workout's changes: **Connected** or **Disconnected**.
_Avoid_: Link, sync, mirroring, reachable, paired

### Community

**Publish**:
The one-way act of copying a Custom Exercise into a Community Exercise.
_Avoid_: Share, upload

**Rating**:
One user's score of a Community Exercise.
_Avoid_: Review, vote, like

**Report**:
One user's flag that a Community Exercise is wrong or inappropriate, for moderation.
_Avoid_: Flag, complaint, issue

### Import

**Import**:
One run of bringing a Strong or Hevy export into the app; imported Workouts link to it and keep their original rows as extras.
_Avoid_: Migration, sync

### Appearance

**Style**:
The app's overall visual language, either **Classic** (pre-iOS 26 native look) or **Glass** (Liquid Glass look).
_Avoid_: Theme, skin, mode
