# Convin Backend Intern Assignment

This project fixes a webhook ingestion service that was duplicating call events, overcounting account stats, and silently dropping async recording work.

I fixed the root cause by making event deduplication atomic in Postgres using a unique `event_id`, separated recording processing from the HTTP request context, and locked the in-memory stats cache to prevent concurrent race conditions.

I also added a regression test to ensure duplicate retries do not double-count anything.
