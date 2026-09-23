# Strong and Hevy export file formats

Research for issue #4. Researched 2026-09-23. Vocabulary follows `CONTEXT.md` (Workout, Exercise, Set, Routine, Custom Exercise).

## Answer

- **Neither app publishes a spec.** Strong's help centre only says the export is a "spreadsheet friendly CSV" that can't be imported back into Strong [S1]. Hevy's help centre describes the menu path, not the columns [H1][H2]. Everything below comes from real export files published on GitHub and from open-source importers. Each claim is marked with the source it came from.
- **Both apps export one flat CSV with one row per Set.** Workout-level fields repeat on every row. Neither exports Routines.
- **Strong has at least three header variants**, and an importer must accept all of them:
  1. *Legacy* (about 2020–2021): comma-delimited, `Duration` like `2h 38m`, unitless `Weight`/`Distance`.
  2. *Unit-column*: semicolon-delimited, `Weight Unit` / `Distance Unit` columns and `Workout Duration` at the end.
  3. *Current* (seen in a 2026 export): semicolon-delimited, every field quoted, `Workout #`, `Duration (sec)`, `Weight (kg)`, `Distance (meters)`.

  The delimiter differs by platform and version, so sniff it from the header line. `Set Order` mixes set numbers with `W` (warm-up), `D` (drop set), `F` (failure), `Rest Timer` and `Note` rows. The export has no supersets, no timezone and no flag for Custom Exercises.
- **Hevy has one stable 14-column snake_case header** (`title,start_time,end_time,description,exercise_title,superset_id,exercise_notes,set_index,set_type,<weight col>,reps,<distance col>,duration_seconds,rpe`). The unit is carried in the column name (`weight_kg` or `weight_lbs`, `distance_km` or `distance_miles`). Times are naive local text such as `"15 Sep 2025, 07:48"`. `set_type` is one of `normal|warmup|failure|dropset`. `superset_id` is a small per-Workout integer, empty when the Exercise is not in a superset. `rpe` takes half-steps from 6 to 10. `exercise_notes` holds the Exercise's note and `description` holds the Workout's note.
- **What an importer must handle:**
  - Detect the delimiter and header variant.
  - Map unit-bearing column names. The legacy Strong format has no unit, so ask the user.
  - Treat every time as local wall-clock time with no zone, so ask for or assume the device timezone.
  - Group Sets into Workouts: Strong by `Workout #`, or by `Date` + `Workout Name` when that column is missing; Hevy by `start_time` + `title`.
  - Group Sets into Exercises by name. Neither app exports an exercise ID, so Custom Exercises have to be matched or created by name.
  - Parse RPE as a decimal.
  - Skip Strong's `Rest Timer` and `Note` pseudo-rows. Turn `Note` rows into Exercise notes.

## Strong

### How to export

Profile → Settings → Export data (or "Export Workouts"). The app produces one CSV through the share sheet or email. Measurements export as separate files [S1][C9]. The export can't be imported back into Strong [S1].

### Header variants (all from community sample files)

| Variant | Header line (verbatim) | Delimiter / quoting | Source |
|---|---|---|---|
| A. Legacy | `Date,Workout Name,Duration,Exercise Name,Set Order,Weight,Reps,Distance,Seconds,Notes,Workout Notes,RPE` | `,`; text fields quoted | real export 2020–21 [C1] |
| A′. Legacy, no notes | `Date,Workout Name,Duration,Exercise Name,Set Order,Weight,Reps,Distance,Seconds,RPE` | `,` | sample in [C2] (provenance unclear) |
| B. Unit columns | `Date;Workout Name;Exercise Name;Set Order;Weight;Weight Unit;Reps;RPE;Distance;Distance Unit;Seconds;Notes;Workout Notes;Workout Duration` | `;` | 2023 export shown in [C3] |
| C. Current | `"Workout #";"Date";"Workout Name";"Duration (sec)";"Exercise Name";"Set Order";"Weight (kg)";"Reps";"RPE";"Distance (meters)";"Seconds";"Notes";"Workout Notes"` | `;`; every field quoted, empty = `""` | real export covering 2020–2026, downloaded May 2026 or later [C4]; same header documented in [C5] |

Notes on the variants:
- The 2026 file in [C4] uses variant C for workouts dating back to 2020. That suggests Strong rewrites the whole history in its current format on every export, so the variant depends on the app version at export time, not on when the Workout was logged. This is an inference from one file and is unverified.
- Ryot's importer says "Delimiter is `;` on android and `,` on iOS" [C6]. Variant C files are semicolon-delimited, but their source platform is unknown. Treat the platform/delimiter link as unverified and sniff the delimiter from the header line instead.
- Ryot also accepts `Duration (sec)`, `Weight (kg)` and `Distance (m)` as aliases [C6]. `Distance (m)` doesn't match `Distance (meters)` in [C4], so match headers by prefix (for example `Distance (`).
- No `Weight (lbs)` header turned up in public files; a GitHub code search returned 0 hits. Presumably an lb-set account gets `Weight (lbs)` or similar. This is unverified. Parse the unit from the parentheses rather than hard-coding kg.

### Column meanings

| Column | Meaning / format | Maps to | Source |
|---|---|---|---|
| `Workout #` (C only) | Integer Workout ID, 1…N | Workout identity | [C4][C5] |
| `Date` | `YYYY-MM-DD HH:MM:SS`, 24-hour, **no timezone**, the Workout's start | Workout start | [C1][C4][C7] |
| `Workout Name` | Free text, may be localised (for example "Mittags-Workout") or have trailing spaces | Workout name (maybe the Routine name, but that isn't reliable) | [C4][C5] |
| `Duration` (A) / `Workout Duration` (B) | Human text: `35m`, `2h 38m` | Workout length | [C1][C3][C6] |
| `Duration (sec)` (C) | Integer seconds, for example `4195` | Workout length | [C4] |
| `Exercise Name` | Display name only, for example `Incline Bench Press (Barbell)`; no ID, no custom flag | Exercise | [C4][C5] |
| `Set Order` | `1`…`N` working Set; `W` warm-up; `D` drop set; `F` failure; `Rest Timer`; `Note` | Set order and type | [C4][C5][C6] |
| `Weight` / `Weight (kg)` | Decimal (`60.0`). In A it has no unit and follows the app setting. In B the unit is in `Weight Unit` (`kg`/`lbs`). Bodyweight Sets have `0`/`0.0` (A) or empty (C) | Set weight | [C1][C3][C4][C9] |
| `Reps` | Integer, sometimes written as a decimal (`13.0` in [C2]) | Set reps | [C1][C2] |
| `RPE` | Decimal or empty; rarely filled | Set RPE | [C4][C8] |
| `Distance` / `Distance (meters)` | Decimal. In A the unit follows the app setting (a swim shows `1.0`, likely km or miles; unverified). In B the unit is in `Distance Unit` | Set distance | [C1][C4] |
| `Seconds` | Seconds for timed Sets; on `Rest Timer` rows it's the rest length (`120.0`) | Set duration / rest | [C1][C4][C5] |
| `Notes` | Exercise note. In C it appears on a separate row with `Set Order = Note` | Exercise note | [C4][C5] |
| `Workout Notes` | Workout note, repeated on every row | Workout note | [C4][C5] |

### Sample rows

Variant A [C1]:
```
Date,Workout Name,Duration,Exercise Name,Set Order,Weight,Reps,Distance,Seconds,Notes,Workout Notes,RPE
2020-12-30 18:51:52,"Evening Workout",2h 38m,"Snatch (Barbell)",1,40.0,3,0,0,"","",
2021-05-13 12:00:00,"Evening Workout",5m,"Swimming",1,0,0,1.0,30,"","",
```
Variant B [C3]:
```
2023-07-30 16:49:28;"Arms";"Bench Press (Dumbbell)";1;14;kg;8;;;;0;"";"";44m
```
Variant C [C4]:
```
"1";"2020-05-28 12:58:47";"Day 3";"4195";"Incline Bench Press (Barbell)";"W";"40.0";"15";"";"";"";"";""
"1";"2020-05-28 12:58:47";"Day 3";"4195";"Incline Bench Press (Barbell)";"1";"60.0";"7";"";"";"";"";""
"1";"2020-05-28 12:58:47";"Day 3";"4195";"Lying Leg Curl (Machine)";"Note";"";"";"";"";"";"Beine einzeln";""
"64";"2020-09-12 12:12:59";"Leg 2";"3872";"Hammer Curl (Dumbbell)";"D";"10.0";"10";"";"";"";"";""
"692";"2025-02-08 13:48:22";"Mittags-Workout";"4725";"Romanian Deadlift (Barbell)";"Rest Timer";"";"";"";"";"120.0";"";""
```
In [C4], about 3,661 of roughly 23,000 rows are `Rest Timer` rows. They must be filtered out.

### Strong quirks

- **Supersets:** the export has no column for them and none of the sample files show them, so superset grouping is lost [C4][C5]. That's based on the absence of a column. Strong's help centre doesn't confirm it (unverified).
- **Set types:** there's a single `Set Order` column, so a Set can't be both numbered and typed. `W`, `D` and `F` rows carry no ordinal. `F` is handled by Ryot [C6] but doesn't appear in the sample files (unverified).
- **Distance/duration Sets:** Weight and Reps are `0` in A and empty in C. The Exercise type must be inferred from which fields are filled; Ryot infers it from the first non-zero fields [C6].
- **Exercise names** combine the name with the equipment in parentheses, for example `Squat (Barbell)`. The same names are used for Custom Exercises, with no marker [C4].
- **Duplicate rows** occur. One analysis found 2,695 exact duplicates in a raw file, most of them `Rest Timer` rows [C8].
- A Workout without `Workout #` has to be keyed on `Date` + `Workout Name`. Ryot keys on `Date` alone [C6].

## Hevy

### How to export

Profile → Settings → Export & Import Data → Export Data → Export Workouts. The file is `workout_data.csv`. Measurements go to a separate `measurement_data.csv` [H1][C10]. The export can't be imported back into Hevy, but Hevy can import a Strong CSV [H1]. The help pages are behind Cloudflare and couldn't be fetched directly; their content here comes from search snippets.

### Header and columns

Header, verbatim from a real kg export [C2]:
```
"title","start_time","end_time","description","exercise_title","superset_id","exercise_notes","set_index","set_type","weight_kg","reps","distance_km","duration_seconds","rpe"
```
A real lb export has `weight_lbs` and `distance_miles` in the same positions [C11].

| Column | Meaning / format | Maps to | Source |
|---|---|---|---|
| `title` | Workout name | Workout name | [C2][C11] |
| `start_time`, `end_time` | `"D Mon YYYY, HH:MM"`, for example `"15 Sep 2025, 07:48"`: 24-hour, no seconds, **no timezone**, English month abbreviation, comma inside quotes | Workout start/end | [C2][C11][C12] |
| `description` | Workout note | Workout note | [C12]; empty in the samples |
| `exercise_title` | Display name, for example `Squat (Barbell)`; no template ID, no custom flag | Exercise | [C2] |
| `superset_id` | Empty = not in a superset. An integer (`0`, `1`, …) is shared by the Exercises in the same superset within that Workout | Superset grouping | [C2][C13] |
| `exercise_notes` | Exercise note, repeated on every Set row of that Exercise | Exercise note | [C2] |
| `set_index` | 0-based Set order within the Exercise | Set order | [C2] |
| `set_type` | `normal`, `warmup`, `failure`, `dropset` | Set type | [C2][H3] |
| `weight_kg` / `weight_lbs` | Decimal or empty (empty for bodyweight or timed Sets); the unit is in the column name | Set weight | [C2][C11][C14] |
| `reps` | Integer; `0` for timed Sets | Set reps | [C2] |
| `distance_km` / `distance_miles` | Decimal or empty | Set distance | [C2][C11] |
| `duration_seconds` | Integer; `0` (not empty) for weight/reps Sets | Set duration | [C2] |
| `rpe` | Empty or one of 6, 7, 7.5, 8, 8.5, 9, 9.5, 10 | Set RPE | [C2][H3] |

### Sample rows

kg account [C2]:
```
"Lower // Ramp // Strength // 1","15 Sep 2025, 07:48","15 Sep 2025, 09:13","","Seated Leg Curl (Machine)",,"",0,"warmup",39,10,,0,6
"Lower // Ramp // Strength // 1","15 Sep 2025, 07:48","15 Sep 2025, 09:13","","Seated Leg Curl (Machine)",,"",3,"failure",78,10,,0,9.5
"Legs // Ramp // Hypertrophy // 1","11 Sep 2025, 07:49","...","","Hip Adduction (Machine)",0,"",1,"normal",...
"Arms 1 + Weakpoint","3 May 2025, 09:31","3 May 2025, 10:20","","Warm Up",,"",0,"normal",,0,,300,
```
lb account [C11]:
```
"Thursday- Upper Reps","28 Mar 2025, 17:29","28 Mar 2025, 18:52","","Band Pullaparts",,"",0,"normal",,20,,0,
```

### Hevy's own data model (primary source, the API rather than the CSV)

Hevy's public API spec [H3] matches the CSV semantics:
- set `type` is one of `warmup|normal|failure|dropset`
- `rpe` is one of `6,7,7.5,8,8.5,9,9.5,10`
- `superset_id` is a nullable integer
- weights are `weight_kg`
- API times are ISO 8601 UTC (`2024-08-14T12:00:00Z`), but the CSV uses naive local text
- exercise templates have `is_custom` and a `type` of `weight_reps|reps_only|bodyweight_reps|bodyweight_assisted_reps|duration|weight_duration|distance_duration|short_distance_weight`
- sets can carry a `custom_metric` (floors or steps)

The CSV drops the template ID, `is_custom`, the exercise type and `custom_metric`. The API is a possible higher-fidelity import path, but it needs the user's Hevy Pro API key (unverified).

### Hevy quirks

- **The unit column differs by account.** Community guides disagree: one says weights are "always in the weight_lbs column" [C12], while a real Sep 2025 file has `weight_kg` [C2]. Accept either and read the unit from the header [C13][C14].
- **Timestamps have a comma inside quotes**, so the file needs a real CSV parser [C12]. Some converters also accept ISO timestamps, which suggests a format change at some point [C13] (unverified).
- **RPE is fractional.** Ryot parses it as `u8`, which would reject `9.5` [C6]. Use a decimal type.
- **Superset IDs are only meaningful within one Workout.** Several Exercises share one ID, and `0` is a valid ID [C2].
- **Workout identity** has to be inferred from `start_time` (+ `end_time`/`title`), because there's no Workout ID [C6][C14]. Minute resolution makes collisions possible if two Workouts start in the same minute.
- **Hevy's Strong import is lossy.** One user lost 5–10 Workouts [C11]. It doesn't affect the Hevy export format, but it means a user's Hevy history may already contain imported Strong data with duplicates.

## Implications for our importer (inference, not sourced)

- Model a "source row" layer: delimiter sniffing → header variant detection → unit resolution (from the column name, a unit column, or a user prompt) → group into Workouts → group into Exercises → Sets.
- Set types need at least normal, warm-up, drop set and failure to import both apps without loss. Also keep Exercise-level and Workout-level notes and an optional superset group.
- Store imported times as local date-time plus a timezone chosen at import (default: the device timezone).
- Custom Exercise matching is name-based for both sources. Use a mapping step for names that don't match a Library Exercise and create Custom Exercises for the rest.

## Sources

Primary (vendor):
- [S1] Strong Help Center, "Can I export my workout data?" https://help.strongapp.io/article/235-export-workout-data
- [H1] Hevy Help Centre, "How to Import Strong App CSV Files and Export Your Data in Hevy" https://help.hevyapp.com/hc/en-us/articles/38001424401943-How-to-Import-Strong-App-CSV-Files-and-Export-Your-Data-in-Hevy (Cloudflare-blocked; content from search snippets)
- [H2] Hevy Help Centre, "Tutorial: Log Previous Workouts and Import CSV" https://help.hevyapp.com/hc/en-us/articles/35687878672663-Tutorial-Log-Previous-Workouts-and-Import-CSV (not fetched)
- [H3] Hevy Public API OpenAPI spec, https://api.hevyapp.com/docs/ (read from `swagger-ui-init.js`)

Community (sample files and open-source importers):
- [C1] Real Strong export, variant A: https://github.com/AlexandrosKyriakakis/StrongAppAnalytics/blob/main/Data/strong.csv
- [C2] Sample Strong (A′) and real Hevy (kg) files: https://github.com/DaKheera47/strong-statistics/tree/60adc118f2eeb16b332a9327df090e3a9ba21759/data_sample
- [C3] Strong variant B header and rows: https://aebel-shajan.github.io/notes/projects/gym-data-analysis/
- [C4] Real Strong export, variant C, 2020–2026: https://github.com/PaulH322/FitnessData/blob/HEAD/data/raw/strong_userdata.csv
- [C5] Strong variant C schema write-up: https://github.com/LordRaydenMK/Bybon/blob/HEAD/docs/strong-import.md
- [C6] Ryot importers (Strong and Hevy): https://github.com/IgnisDa/ryot/tree/8d5fe9840806ae695d1bceb94d689df5ecdaef0d/crates/services/importer (`strong-app/src/lib.rs`, `hevy/src/lib.rs`)
- [C7] strong-progress parser: https://github.com/treble-snake/strong-progress/blob/HEAD/src/engine/parsing/strong-app.ts
- [C8] FitnessData README (duplicates, row kinds): https://github.com/PaulH322/FitnessData
- [C9] Taper, Strong export guide: https://thetaperapp.com/articles/how-to-export-strong-data/
- [C10] Gript, Hevy export guide: https://griptapp.com/help/export-workouts-from-hevy
- [C11] AJ's Blog, real Hevy lb export: https://blog.ayjc.net/posts/migrate-strong-hevy-app/
- [C12] Taper, Hevy export guide: https://thetaperapp.com/articles/how-to-export-hevy-data/
- [C13] openweight, Hevy migration: https://openweight.dev/migrate/hevy.html
- [C14] heart-api issue #72, Hevy CSV import: https://github.com/kit-g/heart-api/issues/72
