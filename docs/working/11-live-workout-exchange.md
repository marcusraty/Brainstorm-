# Working capture: how iPhone and Watch exchange a Live Workout (#11)

Status: **in discussion, not yet decided.** Working notes for [Define how iPhone and Watch exchange Live Workout changes, including Finish](https://github.com/marcusraty/Brainstorm-/issues/11). When the ticket closes, the decision moves into an ADR and this file is deleted or kept as history.

Research behind it: `docs/research/watch-workout-and-sync.md`, `docs/research/workout-split-and-mirroring-channel.md`, `docs/research/watch-app-version-gate.md`.

## The problem in one paragraph

ADR-0003 gives a Live Workout two copies (iPhone and Watch) that can each change while the devices can't talk. Adding Sets never conflicts (each Set has its own ID, so both sets of Sets combine). The hard cases are the same thing changed on both sides while apart, and **endings** (Finish/Discard) made by a device that can't see the other. HealthKit only adds consequences (the Health record must match the final Workout).

## Current proposal (Q25 + Q26, awaiting the user)

1. **Live:** each change (Set added/edited/deleted, Exercise added/moved) is sent over the HealthKit live channel (`sendToRemoteWorkoutSession`) with its own ID and **acknowledged** by the receiver. Un-acknowledged changes wait in a **persistent outbox** (saved with the Workout, survives restarts) and are resent on a ~10 s timeout or on reconnect (iPhone: `workoutSessionMirroringStartHandler` delivers a new mirrored session; Watch: first message that gets through). The receiver ignores IDs it has already applied and acks them again. Result: no drift beyond the disconnect itself.
2. **Disconnect banners:** iPhone via Apple's `didDisconnectFromRemoteDeviceWithError` ("Last heard from your Watch at 18:05"); Watch when a send fails ("iPhone not connected").
3. **Endings** (Finish, Discard) go through the **persistent queue** (WatchConnectivity `transferUserInfo`) carrying **the whole Workout** as that device has it, as a final safety net (needed when the Watch's session ended while apart, so no reconnect callback ever comes).
4. **Final reconciliation** when an ending arrives, record by record, using each device's own "changed since last synced with the other" marks (no clock comparisons): changed on one side only → taken silently; changed on both sides → a **Conflict**.
5. **Conflicts** (same thing changed on both while apart; delete vs edit; different endings, including Finish vs Discard and two different Finishes): during the Workout each device shows its own version, flagged; after the ending the Workout is marked **Needs review** and the iPhone shows a side-by-side comparison (pick per item, or "Keep everything"). **Until reviewed, everything is kept** (all Sets from both, the later ending).
6. **Undo** when a Discard arrives from the other device while Live (for the accidental Discard while Connected).
7. **No heartbeat.**
8. **Workout time (Q26):** when the devices first see each other in a Workout, they fix **time zero** (the Start on the starting device) and measure the offset between their clocks. Every time inside the Workout (Sets, endings, Health record) is stored as Workout time, so both devices order things the same way.
9. Carried over and accepted: Sets ordered by time logged (now Workout time); Workout Exercises carry a position; the Watch deletes a Finished Workout only after the iPhone confirms it has it (Q9).

## Accepted along the way and still standing

- Q5: Sets ordered by time logged; Workout Exercises have a position either device can change.
- Q9: iPhone confirmation before the Watch deletes.
- Q18/Q19/Q20 as folded into items 5 and 6 above (user proposed the review screen; accepted in spirit, final yes pending with Q25).

## Superseded (kept so nothing is lost)

| Earlier answer | Replaced by | Why |
|---|---|---|
| Q1 numbered changes per device | Per-change IDs + acks + outbox | User wanted persistence without extra machinery |
| Q2/Q10 "workout data channel only" | Channel + outbox + ending queue | Research: channel drops while Disconnected; Watch ending while apart never reaches iPhone |
| Q3/Q4/Q11 "latest edit wins", "delete wins", "by what each had seen" | Both-changed → Conflict → user review | User: no hidden rules; clocks unreliable |
| Q6 drop Sets after Finish | Keep everything; review | User: never lose Sets |
| Q7 earliest Finish wins | Different endings → Conflict → review | Same |
| Q8/Q12 Discard variants (only when empty, "what it had seen", split into Continuation) | Review + Undo | Felt hacky; user wants to choose |
| Q14-Q17 Continuation, derived IDs, greyed-out Discard, "latest ending wins" | Withdrawn | Same |
| Q21 queue for every change; Q22 heartbeat; Q23/Q24 | Q25 | User: acks on the live channel, queue for endings only |

## Contradictions with existing decisions (to amend on close)

- **ADR-0003** "Finish is final wherever it is tapped… the other device applies it when it learns of it": an ending is final **unless the other device also changed things while apart**, in which case it goes to review. Also its "queued when Disconnected" becomes "held in our outbox and resent on reconnect; endings also via the persistent queue".
- **ADR-0003 / ADR-0006 Health:** when a review or a Discard changes a Workout the Watch already saved to Health, the change must be carried out **on the Watch** (research: the iPhone app probably can't delete or replace what the Watch app wrote; unverified). ADR-0006 "Discard deletes our samples" becomes "when a Discard becomes final (no conflict, or confirmed in review), the Watch deletes the samples its session wrote".
- **CONTEXT.md Discard** "so that nothing of it is kept": still true once a Discard is final; add that a Discard made while apart can be overturned in review. New terms: **Conflict**, **Needs review**, **Workout time**.
- **Lifecycle resolution (#19)** "earliest Finish?" question: answered by review, not by a rule.
- **ADR-0001** (Watch deletes after iPhone confirms): unchanged; a Workout that Needs review stays on the Watch until reconciled.
- **ADR-0005** (sync version in every message header): applies to live-channel changes, acks, and ending messages.

## Device checks to run before building

1. Does a queued `transferUserInfo` wake the iPhone app in the background, or only deliver on next launch?
2. Does the Watch get any callback when the iPhone reconnects mid-Workout?
3. Can the iPhone app delete or replace Health data the Watch app wrote (separate sources)?
4. Exercise minutes the system awards: can they be removed by deleting a Workout?
5. Live channel: rate (100 KB / 10 s), hang behaviour (~5% reports), error codes 100 / 554.
6. Clock offset stability between iPhone and Watch during a Workout.
