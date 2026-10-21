# SharePoint Online integration notes — Ticket Time Tracker

This document accompanies the SharePoint Online migration design in the Ticket Portal repository:
https://github.com/CTMJUAN/ticket-metrics-dashboard/blob/sharepoint-online-integration/docs/sharepoint-online-migration.md

## Current tracker storage contract

The current tracker is a single-file HTML app with a restrictive Content Security Policy that includes `connect-src 'none'`. It uses browser IndexedDB for per-browser ticket/event state and the File System Access API to write `ticket-work-log-{agent}-{month}.csv` files to a user-selected folder. The current CSV field order is:

`Timestamp, Agent ID, Event, Ticket ID, Customer, Complaint, Elapsed, Competency / Training Readiness, Category`

These are not a shared multi-user persistence layer. Local IndexedDB is isolated by browser profile/device; a folder selected with the File System Access API is local filesystem access, not direct SharePoint REST.

## Required integration seam

Do not add SharePoint REST calls to this HTML file and assume it will work when uploaded as a page. The current CSP blocks all connections. Further, SharePoint's supported HTML page sandbox documents that arbitrary `fetch()` requests/outbound API calls are unsupported and browser `localStorage` is unavailable. The tracker needs a supported app host before a shared SharePoint data adapter can be integrated.

Preferred path: port the UI and lifecycle logic into an approved SharePoint Framework (SPFx) web part, bundling runtime code with the package and using the signed-in user's existing SharePoint permissions. No external runtime dependencies, OneDrive sync, secrets in client code, or permission workarounds. SPFx package deployment may require an authorized app-catalog owner or administrator; if so, request approval rather than bypassing it.

## Migration rules for the implementation

- Preserve ticket lifecycle transitions, timer semantics, elapsed-time units, and event names exactly after mapping every transition from the source.
- Keep TicketEvents append-only during normal operation. Use stable event idempotency keys so retries don't duplicate work-log rows.
- Use stable TicketKey values across tracker and portal; do not rely on row position or display text as identity.
- Use concurrency checks for updates and explicitly handle partial failure between ticket state and event writes.
- Keep the CSV import/export path as a controlled migration and backup path, not the live synchronization mechanism.
- Do not remove existing local mode until the shared backend is validated and a rollback path is tested.
- Validate behavior with multiple authorized accounts and with read-only users; SharePoint permissions must remain the security boundary.

## Validation status

No lifecycle code has been changed in this branch yet. The next implementation step depends on confirmation that an SPFx package can be deployed to the target site, or that the organization has an already-approved application host that supports authenticated same-site API calls.
