# Workout Tracker

A strength-training tracker for iPhone and Apple Watch, where the Watch can run a Workout on its own as well as alongside the iPhone.

## Language

### Training

**Routine**:
A reusable, named plan of Exercises (and their target Sets) that a Workout can be started from.
_Avoid_: Template, program, plan

**Workout**:
One performed training session, from start to finish, optionally started from a Routine.
_Avoid_: Session, log, training

**Exercise**:
A named movement, such as Bench Press, that Sets are recorded against.
_Avoid_: Movement, lift

**Set**:
One recorded effort within an Exercise in a Workout: weight × reps, or time or distance.
_Avoid_: Rep, entry

**Library Exercise**:
An Exercise bundled from the open-source exercise library; read-only to users.
_Avoid_: Built-in exercise, default exercise

**Custom Exercise**:
An Exercise a user created themselves, private to that user.
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
Throwing away a Live Workout from either device, so that nothing of it is kept.
_Avoid_: Cancel, delete, abandon

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

### Appearance

**Style**:
The app's overall visual language, either **Classic** (pre-iOS 26 native look) or **Glass** (Liquid Glass look).
_Avoid_: Theme, skin, mode
