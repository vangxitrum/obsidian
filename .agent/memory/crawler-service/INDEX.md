# crawler-service — memory index


- [[agm-crawler-test-coverage]] — how to test a fetcher (private feed iface + struct literal), the gwis_nasa nil-deref that killed the process, the committed Mongo credential, and 3 pre-existing failures.
- [[earthquake-fire-app-review-2026-09-07]] — review of internal/app + internal/earthquake + internal/fire: shakemap's StoreIfAbsent `inserted` bool discarded (always republishes), gdacs earthquake's day-granularity cursor causes duplicate full-day republishes every ~10min, gwis_nasa's copy-pasted WMS base_url breaks the fetcher entirely, and usgs/shakemap detail-fetch all-or-nothing failure semantics. Also documents the fetcher goroutine/errgroup architecture in app.go.
