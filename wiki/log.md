# Change Log

> Newest first. One `## YYYY-MM-DD` heading per day (ISO 8601). Maintained by
> the agent — an entry is appended on every ingest/lint change.

## 2026-07-09

- **Lint**: Health-check (no broken links, no orphaned knowledge pages, `index`/`log` in sync). Low-risk fixes: reworded the stale "future logging pages" note in [Log line format](/logging/format.md); fixed `SPEC.md` typos (`specifific`→`specific`, malformed `python` table row) and added `typescript` + `rust` type rows; linked `SPEC.md` from the root [index](/index.md). Resolved a soft contradiction on `CRITICAL` handling by making [Log output streams](/logging/streams.md) the single authority (own-code vs dependency distinction) and trimming the duplicate clause from the format page. Adopted convention `timestamp` = last-modified (documented in `SPEC.md`); bumped `format.md`/`streams.md` timestamps.
- **Ingest**: Added [Log output streams](/logging/streams.md) (`type: practice`) — convention that DEBUG/INFO/WARNING → stdout and ERROR → stderr (stderr only), with `ERROR` as the separately-alertable channel. Dropped `CRITICAL` from the canonical level set (now four levels). Filed the Python implementation [Logging in Python](/python/logging.md) (`type: python`) — two-handler + `_MaxLevelFilter` setup, UTC formatter via `gmtime`, and the `basicConfig` limitation. Revised [Log line format](/logging/format.md) to remove `CRITICAL` from the `level` schema row (correcting a now-superseded claim) and added a `# See also` cross-link. Refreshed root [index](/index.md) (Logging group gains the streams page; Python group now populated).
- **Creation**: Added [Log line format](/logging/format.md) (`type: practice`) — ingested the convention that all logging uses `<timestamp_utc> <level> <message>`. Refreshed root [index](/index.md) (declared `okf_version: "0.1"`; switched to topic-based grouping).
- Initialised the wiki: scaffolded `SPEC.md`, `index.md`, and this `log.md`.
