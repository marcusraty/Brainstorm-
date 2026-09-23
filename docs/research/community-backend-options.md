# Community Exercises backend: CloudKit public database vs own droplet

Research for [issue #3](https://github.com/marcusraty/Brainstorm-/issues/3). Researched 2026-09-23. Vocabulary follows `CONTEXT.md`: **Community Exercise**, **Publish**, **Rating**, **Report**.

## Answer

**Facts**

- **Reading works without iCloud; writing does not.** The public database "is available regardless of whether the user's device has an iCloud account" and is "readable by all users of the app" ([publicCloudDatabase](https://developer.apple.com/documentation/cloudkit/ckcontainer/publicclouddatabase)). An Apple DTS engineer confirmed in Feb 2026: "only authenticated iCloud users can create and write data to a CloudKit public database" ([forum 816523](https://developer.apple.com/forums/thread/816523)). So browsing Community Exercises works for everyone, but Publish, Rating and Report need the user to be signed in to iCloud.
- **Permissions fit the model.** The built-in roles are World (read-only), Authenticated and Creator, and permissions (read/write/create) are set per record type ([Designing for CloudKit](https://developer.apple.com/icloud/cloudkit/designing/)). "World-readable, creator-writable" is the default shape.
- **CloudKit cannot compute aggregates on the server.** `CKQuery` offers predicates and sorting, with no SUM/COUNT/AVG ([CKQuery](https://developer.apple.com/documentation/cloudkit/ckquery)), and there are no server-side triggers. You would compute an aggregate Rating on the client, or run a small job that uses a CloudKit Web Services server-to-server key, which "access[es] the public database of a container as the developer who created the key" ([CloudKit Web Services](https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/CloudKitWebServicesReference/SettingUpWebServices.html)).
- **Cost: CloudKit is free up to some quota, and Apple no longer publishes the quota.** Apple's current pages say only "up to 1PB of free storage for each app" ([What's included](https://developer.apple.com/programs/whats-included/)). The request/storage/transfer numbers in circulation (for example 40 req/s, rising to 400 req/s) come from older Apple pages quoted on forums. **I could not verify them against a current Apple page.** Going over a limit produces throttling errors (`requestRateLimited` with a retry-after), not surprise bills ([CKError.requestRateLimited](https://developer.apple.com/documentation/cloudkit/ckerror/code/requestratelimited); forum reports: [6430](https://developer.apple.com/forums/thread/6430), [113318](https://developer.apple.com/forums/thread/113318)).
- **A droplet costs about $6–$30 a month plus ongoing ops, and you must build identity and anti-abuse yourself.** Basic droplets start at $4/$6/$12 a month ([DO pricing](https://www.digitalocean.com/pricing/droplets)). Managed Postgres starts at $15.15 a month ([DO DB pricing](https://www.digitalocean.com/pricing/managed-databases)). Without iCloud identity, App Attest is the strongest account-free option: it proves requests come from a genuine copy of the app on a real device. It does not prove a unique human ([DeviceCheck](https://developer.apple.com/documentation/devicecheck)).

**Recommendation (the decision is yours):** use the **CloudKit public database** for Community Exercises. It matches "no accounts, iCloud is identity", costs nothing at this app's likely scale, and has no server to patch. Accept that users must be signed in to iCloud to Publish, Rate or Report. For aggregate Ratings and Report triage, add a small scheduled job that uses a server-to-server key; it can run anywhere, including a $4–6 droplet or CI. Revisit a full droplet backend only if you need writes from users who are not signed in to iCloud, or server-side logic on every write.

## Detail: CloudKit public database

### Quotas, rate limits, scaling (unverified numbers)

| Claim | Source | Status |
|---|---|---|
| "up to 1PB of free storage for each app" | [Apple Developer Program, What's included](https://developer.apple.com/programs/whats-included/); [CloudKit page](https://developer.apple.com/icloud/cloudkit/) says "up to 1PB of storage for your app's public data" | Verified (current Apple page) |
| Limits scale with *active users* = users with container activity "within the last 16 months"; "overage charges may apply"; extra containers give no extra allowance | Quoted as Apple's text in [forum 660009](https://developer.apple.com/forums/thread/660009) (Sept 2020) | **Unverified**: the original Apple page is no longer online. I could not fetch the Wayback Machine. |
| Base free tier 10 GB assets / 100 MB DB / 2 GB transfer / **40 req/s**, scaling with users | Community reply, [forum 6430](https://developer.apple.com/forums/thread/6430) (2015) | **Unverified**, and may be stale |
| Public DB ceiling of 1 PB assets / 10 TB DB / 200 TB transfer / **400 req/s** | Community reply, [forum 113318](https://developer.apple.com/forums/thread/113318) (2020) | **Unverified** |
| Overage of $100 per extra 10 req/s, $0.10/GB transfer, $3/GB DB, $0.03/GB assets | Quoted in [forum 660009](https://developer.apple.com/forums/thread/660009) | **Unverified**. No current Apple page lists prices. |
| Asset downloads count as requests; failed requests count | Community replies, [forum 114154](https://developer.apple.com/forums/thread/114154) | **Unverified** |

Verified behaviour:

- Throttling returns `CKError.requestRateLimited`. Apps should read `CKErrorRetryAfterKey` and wait that many seconds ([docs](https://developer.apple.com/documentation/cloudkit/ckerror/code/requestratelimited)).
- Per-operation limits are "400 items (records or shares) per operation" and "2 MB per request (not counting asset sizes)". "The server can change its limits at any time" ([CKError.limitExceeded](https://developer.apple.com/documentation/cloudkit/ckerror/code/limitexceeded)).
- Running out of public storage returns `quotaExceeded`, and "you can go to the CloudKit Dashboard to view and manage your container's storage" ([docs](https://developer.apple.com/documentation/cloudkit/ckerror/code/quotaexceeded)).
- CloudKit Console shows usage (active users, storage) and telemetry (requests, errors, latency) ([WWDC24 10122](https://developer.apple.com/videos/play/wwdc2024/10122/)). Treat Console as the source of truth for your container's real limits once it exists.
- Throttling has fired unexpectedly on Web Services before: a 503 THROTTLED at about 0.3 req/s in May–June 2021, with sporadic reports into 2024 ([forum 683677](https://developer.apple.com/forums/thread/683677)).

For scale: a text-only Community Exercise plus Rating and Report records are tiny. Even the lowest unverified numbers (100 MB DB, 40 req/s) are unlikely to bind early on, provided clients cache and don't poll. That is an inference, not a measurement.

### Cost

No verified current cost. The Developer Program membership ($99/yr, which you already need for the App Store) includes CloudKit. The only Apple-stated figure today is the 1 PB storage ceiling ([What's included](https://developer.apple.com/programs/whats-included/)). Community reports say hitting a limit causes throttling or errors, not automatic billing ([6430](https://developer.apple.com/forums/thread/6430), [113318](https://developer.apple.com/forums/thread/113318)). **Unverified.**

### Security roles and permissions

- Roles: "World role includes all iCloud users, whether or not they are authenticated. The Authenticated role includes all iCloud users who are signed in… The Creator role includes only the authenticated user who has created a record." Permissions are read / write / create and are additive, and "you set permission levels for a role, then assign the role to a given record type" ([Designing for CloudKit](https://developer.apple.com/icloud/cloudkit/designing/)).
- "users have write access to the records… they create. The public database's contents are visible in the developer portal, where you can assign roles to users and restrict access" ([publicCloudDatabase](https://developer.apple.com/documentation/cloudkit/ckcontainer/publicclouddatabase)).
- Roles on record types can cause `permissionFailure`, which is nonrecoverable ([docs](https://developer.apple.com/documentation/cloudkit/ckerror/code/permissionfailure)).
- A record type can be made readable only by its creator, for example for Reports. Developers discussing this in 2026 confirmed it ([Michael Tsai summary](https://mjtsai.com/blog/2026/06/05/permissions-in-the-cloudkit-public-database/), [Rambo](https://www.rambo.codes/posts/2021-12-06-using-cloudkit-for-content-hosting-and-feature-flags)). These are secondary sources.

A suggested mapping (inference):

| Record type | World | Authenticated | Creator |
|---|---|---|---|
| CommunityExercise (Publish) | read | create | write |
| Rating | read | create | write |
| Report | none | create | read/write |

### Computing aggregate Ratings

CloudKit has no aggregation queries or server functions ([CKQuery](https://developer.apple.com/documentation/cloudkit/ckquery)). The options (my inference) are:

1. **Client-side.** Fetch the Rating records for one Community Exercise and average them on the device. This is simple, but it costs requests that grow with the number of Ratings. It is fine for detail views and poor for sorting a list by rating.
2. **Denormalised fields on the Community Exercise.** This does not work with default roles, because only the Creator can write the record. Granting write to Authenticated would let anyone overwrite it.
3. **Scheduled aggregator using a server-to-server key.** The job reads Ratings through CloudKit Web Services and writes an aggregate record or field as the developer. Keys are ECDSA P-256 keys registered in Console, and "signed requests are valid for only 10 minutes" ([Web Services reference](https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/CloudKitWebServicesReference/SettingUpWebServices.html)). **Unverified:** whether a server-to-server key bypasses record-type roles when writing records other users created. Test this, or have the job write to a separate `RatingSummary` type that only the developer can create.

To keep one Rating per user, give each Rating a deterministic record name (Community Exercise ID plus user). `fetchUserRecordID` returns a stable per-container user ID for signed-in users ([docs](https://developer.apple.com/documentation/cloudkit/ckcontainer/fetchuserrecordid(completionhandler:))). The aggregator should also dedupe by the record's creator rather than trust the name. This is inference.

### Moderation

- **CloudKit Console (Database app)** can inspect and delete public records. Apple warns: "Don't use the CloudKit Database app as a general data editor… the intent of this functionality is to help you debug your schema" ([docs](https://developer.apple.com/documentation/cloudkit/managing-icloud-containers-with-cloudkit-database-app)). It works for occasional manual takedowns but is a poor moderation queue.
- **Server-to-server key / CloudKit Web Services** give scripted access to the public database as the developer ([reference](https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/CloudKitWebServicesReference/SettingUpWebServices.html); [Apple news, 2016](https://developer.apple.com/news/?id=02042016a)). A small admin script or page could list Reports, then hide or delete Community Exercises.
- Apple's App Review Guideline 1.2 requires user-generated content to have filtering, reporting and blocking. Report covers the reporting part. **Not researched here.** Check it separately.

### When the user isn't signed in to iCloud

- The public database stays readable ([publicCloudDatabase](https://developer.apple.com/documentation/cloudkit/ckcontainer/publicclouddatabase)).
- Writes (Publish, Rating, Report) require sign-in ([DTS answer, forum 816523](https://developer.apple.com/forums/thread/816523)). `fetchUserRecordID` returns `notAuthenticated` when there is no account, iCloud is disabled, or access is restricted ([docs](https://developer.apple.com/documentation/cloudkit/ckcontainer/fetchuserrecordid(completionhandler:))).
- Check with `accountStatus` and listen for `CKAccountChanged` ([docs](https://developer.apple.com/documentation/cloudkit/ckcontainer/accountstatus(completionhandler:))). The UI should turn the Publish, Rate and Report controls into a "Sign in to iCloud to …" prompt.

## Detail: self-hosted DigitalOcean droplet

### Cost

| Item | Price | Source |
|---|---|---|
| Basic droplet 512 MiB / 1 vCPU / 10 GiB / 500 GiB transfer | $4/mo | [DO droplet pricing](https://www.digitalocean.com/pricing/droplets) |
| 1 GiB / 1 vCPU / 25 GiB / 1,000 GiB | $6/mo | same |
| 2 GiB / 1 vCPU / 50 GiB / 2,000 GiB | $12/mo | same |
| Backups | 20% (weekly) or 30% (daily) of droplet price | same |
| Bandwidth over allowance | $0.01/GiB, pooled across the team | [DO bandwidth docs](https://docs.digitalocean.com/platform/billing/bandwidth/) |
| Managed PostgreSQL (optional) | from $15.15/mo, 1 GiB RAM, 10–30 GiB | [DO DB pricing](https://www.digitalocean.com/pricing/managed-databases) |

A realistic setup is a $6–12 droplet with SQLite or Postgres on the box plus backups: about $7–16 a month. With managed Postgres it is about $22–30 a month. These totals are my arithmetic from the prices above.

**Ops burden (inference):** OS and security patching, TLS, backups and restore drills, uptime monitoring, and schema migrations. You also build the API, the rate limiting and the moderation tooling yourself. For a solo, ad-supported app, this recurring work is the main cost, more than the dollars.

### Account-free identity options

| Option | What it proves | Abuse / rate-limit implications | Source |
|---|---|---|---|
| **App Attest** (`DCAppAttestService`) | A Secure Enclave key attested by Apple as belonging to a genuine instance of your app. Later requests carry signed assertions with a counter. | Stops scripted clients and modified apps. Does **not** stop one person using many devices or reinstalling; a reinstall creates a new key. Apple can report an approximate count of attestations per device, which helps flag farms. Not available on Mac or in most extensions. watchOS extensions are supported on watchOS 9+. Apple suggests keeping `attestKey` under about 100 req/s app-wide. | [Establishing integrity](https://developer.apple.com/documentation/devicecheck/establishing-your-app-s-integrity), [Validating](https://developer.apple.com/documentation/devicecheck/validating-apps-that-connect-to-your-server), [Fraud risk](https://developer.apple.com/documentation/devicecheck/assessing-fraud-risk), [isSupported](https://developer.apple.com/documentation/devicecheck/dcappattestservice/issupported), [Preparing](https://developer.apple.com/documentation/devicecheck/preparing-to-use-the-app-attest-service) |
| **DeviceCheck** (`DCDevice`) | Your server can "set and query two binary digits of data per device", and the bits survive reinstalls. | Useful for a persistent "banned device" bit that a reinstall does not clear. Too little state for anything else. | [DeviceCheck](https://developer.apple.com/documentation/devicecheck), [Per-device data](https://developer.apple.com/documentation/devicecheck/accessing-and-modifying-per-device-data) |
| **Per-install keypair** (for example a CryptoKit key in Keychain, public key registered on first launch) | Continuity for one install only. | Anyone can mint unlimited identities with a script, so rate limits per key are meaningless. It has to be combined with App Attest, or with IP limits that CGNAT and IPv6 weaken. (Inference.) | n/a (design pattern) |
| **Better Auth anonymous plugin** | Creates a user and session "without requiring… email address, password, OAuth provider, or any other PII". A second `signIn.anonymous()` on the same client errors. Can later link a real account. | Its docs say nothing about abuse. The built-in rate limiter defaults to 100 requests per 60 s per IP (IPv6 grouped by /64), enabled in production. Unlimited anonymous users can still be created from many IPs, so pair it with App Attest. | [Anonymous plugin](https://www.better-auth.com/docs/plugins/anonymous), [Rate limit](https://www.better-auth.com/docs/concepts/rate-limit) |

By comparison, CloudKit's identity is the user's iCloud account. Creating many iCloud accounts is costly, so each Rating or Report in CloudKit is tied to a real Apple ID at no cost to you. This is the strongest anti-sock-puppet property on the table (inference).

## Open questions / not verified

- Current CloudKit public-database quota numbers and overage pricing. Check CloudKit Console → Usage once the container exists.
- Whether a server-to-server key can modify or delete records created by other users when roles restrict write. Test this in the development environment.
- App Store Review Guideline 1.2 obligations for user-generated content (filter, block, contact info).
