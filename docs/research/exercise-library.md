# Research: which open-source exercise library to bundle

Ticket: [#2](https://github.com/marcusraty/Brainstorm-/issues/2). Researched 2026-09-23 against the dataset repos, their LICENSE files and the live wger API. Numbers were measured from shallow clones and API pulls on that date.

## Answer

**Facts**

- **free-exercise-db** (yuhonas) is released under the Unlicense (public domain) and has about 876 Exercises, each with 2 JPG photos (~94 MB of images). It has the cleanest schema for Library Exercises: primary and secondary muscles, equipment, category, level, force, mechanic, and step-by-step instructions. **But** its images come from wrkout/exercises.json, whose author wrote that the images "have been scrapped off the internet, therefore l do not own the copy right for these images and would advise against using them in comercial projects". Three issues on free-exercise-db asking about image licensing have had no answer from the maintainer.
- **wger** has about 910 Exercises. The exercise data is CC-BY-SA 4.0 (757), CC-BY-SA 3.0 (132) or CC0 (21), and each entry names its licence and author. Only 273 Exercises have an image (374 images, all CC-BY-SA, 41 flagged as AI-generated), and 46 have a video. It is actively maintained, with exercise edits as recent as 2026-09-21. For commercial use, CC-BY-SA requires you to credit each author and to share any adapted database under the same licence. It also has a "no Effective Technological Measures" clause, and whether App Store DRM triggers it is an open question.
- **exercemus/exercises** (MIT code, 872 Exercises) mostly merges exercises.json and wger. It adds no clean images and has been essentially dormant since 2022.
- **longhaul-fitness/exercises** (MIT) contains text only: 349 strength Exercises with muscles, steps and notes. It has no equipment or category field and no images.
- **ExerciseDB** is a paid, commercial API, not an open dataset you could bundle. Its repo code is AGPL.

**Recommendation (the decision is yours)**

Seed Library Exercises from **wger**. Show the author and licence on each Library Exercise, and publish the bundled library JSON under CC-BY-SA. Use wger images only where they exist, and plan to commission or make your own images for the rest. **Do not bundle free-exercise-db's images.** If you want free-exercise-db's richer fields, you can use its Unlicense text, but its origin traces back to wrkout's scraped dataset, and I could not verify where the text came from. Have a lawyer check the CC-BY-SA question (App Store DRM and share-alike) before launch.

## Candidates in detail

### 1. free-exercise-db: github.com/yuhonas/free-exercise-db

| Aspect | Finding | Source |
|---|---|---|
| Licence | Unlicense (public domain dedication) for the repo | [LICENSE.md](https://github.com/yuhonas/free-exercise-db/blob/main/LICENSE.md) |
| Origin | Restructured from wrkout/exercises.json | [README "Why?"](https://github.com/yuhonas/free-exercise-db#why) |
| Size | 876 Exercises; `dist/exercises.json` is 1.0 MB; 1,746 JPGs (~94 MB, ~55 KB each, e.g. 850×567) | measured at commit `a859101`; [dist/exercises.json](https://github.com/yuhonas/free-exercise-db/blob/main/dist/exercises.json) |
| Fields | id, name, force, level, mechanic, equipment (13 values), primaryMuscles / secondaryMuscles (17 values), instructions[], category (7 values), images[] | [schema.json](https://github.com/yuhonas/free-exercise-db/blob/main/schema.json) |
| Category mix | strength 584, stretching 123, plyometrics 61, powerlifting 38, olympic weightlifting 35, strongman 21, cardio 14 | measured |
| Images | 2 still photos per Exercise (start and end positions); 3 Exercises have none; no animations | measured |
| Image licence | **Not established.** The upstream author said they were scraped and advised against commercial use. Maintainer has not answered issues [#2](https://github.com/yuhonas/free-exercise-db/issues/2), [#12](https://github.com/yuhonas/free-exercise-db/issues/12) (closed without a visible reply) or [#13](https://github.com/yuhonas/free-exercise-db/issues/13) | [wrkout CONTRIBUTING.md, "Exercise Images"](https://github.com/wrkout/exercises.json/blob/master/CONTRIBUTING.md) |
| Data quality | README says force, mechanic and equipment are null in some files, and that there are duplicate images. Measured nulls: equipment 77, mechanic 87, force 30. 5 Exercises have no instructions. Schema is linted in CI | [README TODO](https://github.com/yuhonas/free-exercise-db#todo); measured |
| Maintenance | Low but alive: 40 commits in 2023, 2 in 2024, 4 in 2025, 4 in 2026. Latest was 2026-08-30 and added kettlebell Exercises (#28) | `git log`; [commits](https://github.com/yuhonas/free-exercise-db/commits/main) |

Text provenance is **unverified**. The wrkout note is about images only. The instruction wording (for example "Tip: … This will be your starting position.") reads like a commercial exercise guide, but I found no primary source that confirms or rules this out. The Unlicense can only give away rights the author actually holds.

### 2. wger: github.com/wger-project/wger and wger.de API

| Aspect | Finding | Source |
|---|---|---|
| Licence | Code AGPL-3.0-or-later. Exercise data "Creative Commons (see individual entries)". Documentation CC-BY-SA-4.0 | [README "License"](https://github.com/wger-project/wger#license); [LICENSE.txt](https://github.com/wger-project/wger/blob/master/LICENSE.txt) |
| Per-entry licences | Available licences: CC-BY-SA 3, CC-BY-SA 4, CC-BY 4, CC0, ODbL. Exercises: CC-BY-SA 4 757, CC-BY-SA 3 132, CC0 21 | [/api/v2/license/](https://wger.de/api/v2/license/); [/api/v2/exerciseinfo/](https://wger.de/api/v2/exerciseinfo/) |
| Size | 910 Exercises (full `exerciseinfo` JSON with all translations ≈ 5.6 MB). All have English; many are also in German (644), language 1 (629) and language 12 (582) | measured from API |
| Fields | category (Abs, Arms, Back, Calves, Cardio, Chest, Legs, Shoulders), muscles and muscles_secondary (15 muscles, Latin plus English names, front/back flag), equipment (12 values), translations (name, HTML description, aliases, notes), variation_group, images, videos, license, license_author, author_history | [exerciseinfo sample](https://wger.de/api/v2/exerciseinfo/?limit=1); [/api/v2/muscle/](https://wger.de/api/v2/muscle/); [/api/v2/equipment/](https://wger.de/api/v2/equipment/) |
| Images | 374 images covering 273 Exercises. Licence: CC-BY-SA 4 (286), CC-BY-SA 3 (88). Style: photo 266 (the model's default), line art 96, other 12. 41 are flagged `is_ai_generated` | [/api/v2/exerciseimage/](https://wger.de/api/v2/exerciseimage/); [image.py](https://github.com/wger-project/wger/blob/master/wger/exercises/models/image.py) |
| Video | 78 videos on 46 Exercises, all CC-BY-SA 4 | [/api/v2/video/](https://wger.de/api/v2/video/) |
| Muscle diagrams | SVGs per muscle (`image_url_main` / `_secondary`). Their licence was **not verified** | exerciseinfo sample |
| Data quality | Crowd-sourced. 157 Exercises have no primary muscle, 217 have no equipment, and 25 have an English description shorter than 20 characters. Instructions are one HTML block, not steps. There is no level, force or mechanic field | measured |
| Maintenance | Very active: repo commits on 2026-09-13, exercise edits on 2026-09-21 | `git log`; `last_update_global` in API |

**Obligations for commercial, ad-supported use.** CC-BY-SA allows commercial use. It requires:

- **Attribution:** credit the creator, give a licence reference and URI, and say what you changed (§3(a)). wger supplies `license_author` and `author_history` for each entry.
- **ShareAlike:** adaptations must be released under the same or a compatible licence (§3(b)). Where database rights apply, a database that includes the contents counts as adapted material (§4(b)). In practice, publish your modified Library Exercise data under CC-BY-SA. Your app code is not affected.
- **No downstream restrictions:** you "may not … apply any Effective Technological Measures to the Licensed Material if doing so restricts exercise of the Licensed Rights" (§2(a)(5)(c)).

Whether App Store FairPlay DRM on the app binary counts as such a measure is a **legal question I have not resolved**. A common mitigation is to also publish the same data openly, for example in a public repo. That is also **not verified** as sufficient. Source: [CC BY-SA 4.0 legal code](https://creativecommons.org/licenses/by-sa/4.0/legalcode.en).

CC0 entries (21) and any CC-BY 4 entries carry no share-alike obligation.

### 3. exercemus/exercises: github.com/exercemus/exercises

- The code is MIT. The README says every exercise keeps its own licence, which users must follow ([LICENSE](https://github.com/exercemus/exercises/blob/main/LICENSE), [README "Licensing Notes"](https://github.com/exercemus/exercises#licensing-notes)).
- It was curated from exercemus, wger and exercises.json (same README).
- It has 872 Exercises with category, instructions[], tips, equipment[], primary and secondary muscles, aliases, `variation_on` and YouTube video URLs ([README "Format"](https://github.com/exercemus/exercises#format)).
- Measured: only 1 entry carries a licence (CC-BY-SA 3), and no entry has images. Most entries come from exercises.json, so the same provenance doubt applies.
- Maintenance: 34 commits in 2022 and 1 in 2025 (`git log`).

### 4. longhaul-fitness/exercises: github.com/longhaul-fitness/exercises

- MIT licence ([LICENSE](https://github.com/longhaul-fitness/exercises/blob/main/LICENSE)).
- Contains 349 strength Exercises (340 KB), plus small cardio and flexibility files.
- Fields: pk, name, slug, primaryMuscles, secondaryMuscles, steps[], notes. Equipment is only encoded in the name ("Shrug – Barbell"), and there are no images ([README](https://github.com/longhaul-fitness/exercises#readme)).
- Muscle names are more detailed, for example "shoulder - back" and "rotator cuff - back" (measured).
- Maintenance: 20 commits in 2025, last on 2025-11-24.
- Content is original to the project as far as the README says. Provenance is not otherwise verified.

### 5. ExerciseDB: github.com/ExerciseDB/exercisedb-api

- The repo code is AGPL-3.0 ([LICENSE](https://github.com/ExerciseDB/exercisedb-api/blob/main/LICENSE)).
- The data (11,000+ Exercises, GIFs, videos) is sold through an API with pricing plans and terms of use. Its playground endpoints are "not recommended for production integration" ([README](https://github.com/ExerciseDB/exercisedb-api#readme)).
- It is not an open dataset that can be bundled. I did not review its commercial terms.

### Not a candidate: wrkout/exercises.json

This is the upstream source of free-exercise-db. It is Unlicense, and its last commit was 2025-02-16. Its README points commercial users to a paid dataset at wrkout.xyz ([README](https://github.com/wrkout/exercises.json#commerical-projects)), and its CONTRIBUTING file carries the scraped-images warning quoted above.

## Could not verify

- Where free-exercise-db / exercises.json instruction text originally came from.
- The licence of wger's muscle SVG diagrams.
- Whether App Store DRM conflicts with CC-BY-SA 4.0 §2(a)(5)(c), and whether publishing the data openly alongside the app resolves it.
- Whether wger's AI-generated images carry any extra caveats beyond their stated CC-BY-SA licence.
