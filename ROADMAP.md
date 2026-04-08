# ROADMAP — BIOS Preservation Tool

Items here are agreed improvements not yet implemented. Each entry notes the
motivation and the rough approach discussed during design.

---

## 1. Stronger verification for `unverifiable` canonicals

**Status:** Planned
**Context:** The reconciliation pass (see Implementation Notes §17 in
DEVELOPER_NOTES.md, once written) corrects `unverifiable` blobs that have a
verified counterpart stored under the same plain text filename. However, the
linking signal is bare filename only — a weak heuristic. Several stronger
secondary checks should be added in a follow-up pass:

- **Size matching** — the manifest sometimes declares `size` even when no
  hashes are known. A filename + size agreement is a meaningfully stronger
  signal than filename alone and costs nothing to check.

- **CRC32 as a lightweight secondary hash** — some platforms declare CRC32
  but not MD5. Not cryptographically strong, but a CRC32 + filename match is
  substantially better than filename alone and is already present in the
  manifest data model.

- **Cross-platform corroboration** — if the same filename appears across
  multiple platforms all declaring the same MD5, and an unverifiable canonical
  matches that filename, the probability it represents the same file is high.
  The manifest already contains all the data needed to compute a
  cross-platform agreement score.

- **User-confirmable flagging** — for cases where none of the above signals
  reach a high-confidence threshold, surface the candidate in the report as
  "probable duplicate — recommend review" rather than auto-correcting. A human
  confirmation path prevents silent mis-aliasing of genuinely distinct files
  that happen to share a name.

---

## 2. Wire `reconcile_aliases` into restore, retire `.aliases.json` sidecar

**Status:** Planned — `.aliases.json` retained as safety net in the interim
**Context:** `bios_restore.py` currently restores alias relationships from the
`.aliases.json` sidecar written by `bios_backup.py`. With `reconcile_aliases`
now running at the end of every build, most aliases can be re-derived from
scratch after a restore rather than being read from the sidecar.

**To complete this:**
- Call `reconcile_aliases(conn, canonical_map)` at the end of `bios_restore.py`
  after blobs are re-ingested and `populate_missing_files` has run.
- Once that is in place, `.aliases.json` is redundant for all MD5-resolvable
  aliases (Cases 1–3 of the reconciliation pass).
- The one case `.aliases.json` uniquely handles is an unverifiable-to-unverifiable
  alias — a canonical with no declared hashes whose bytes are stored as an alias
  of another canonical that also has no declared hashes. No MD5 comparison is
  possible, so reconciliation cannot find it. This edge case is covered by
  Roadmap item 1 (stronger verification for unverifiable canonicals). Until
  that item is addressed, dropping `.aliases.json` would silently lose these
  relationships on restore.
- `.blob_map.json` is not affected and must be retained regardless — it is the
  only mechanism for identifying non-verified blobs whose canonical has since
  left the manifest (orphans), and reconciliation cannot substitute for it.

**Decision:** Keep `.aliases.json` until Roadmap item 1 provides a reliable
signal for unverifiable alias relationships.

---
