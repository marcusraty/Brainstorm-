# The Watch syncs only through the iPhone, never with iCloud directly

The Watch app does not talk to iCloud. Everything it needs arrives from the iPhone, and everything it produces (Finished Workouts) goes to the iPhone, which alone syncs with iCloud. We chose this because the Watch holds only a working set (ADR-0001), and because the research says Watch-side CloudKit sync is slow and poorly documented, so keeping one sync path is simpler. The cost is that if the iPhone is away or dead for days, Finished Workouts wait on the Watch until it returns, rather than reaching iCloud.
