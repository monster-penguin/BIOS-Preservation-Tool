


# ROADMAP — BIOS Preservation Tool

Items here are agreed improvements not yet implemented. Each entry notes the
motivation and the rough approach discussed during design.

---

## 1. Stronger verification for `unverifiable` canonicals

**Status:** Partially addressed — confidence annotation implemented; size/CRC32 cross-matching pending
**Context:** The reconciliation pass (see Implementation Notes §17 in
DEVELOPER_NOTES.md, once written) corrects `unverifiable` blobs that have a
verified counterpart stored under the same plain text filename. However, the
linking signal is bare filename only — a weak heuristic. Several stronger
secondary checks should be added in a follow-up pass.

**Implemented (verify pass — Implementation Note §20):**

A third build pass, `run_verify_pass()`, now runs after `reconcile_aliases` on every build (controlled by `verify_pass = true/false` in config). It annotates every unverifiable blob with a confidence level:

- **High confidence** — 2+ platforms corroborate the file's presence, or a non-MD5 hash matches a verified blob under a different canonical name (true alias).
- **Low confidence** — exactly 1 platform declares the file.

Confidence is stored in `files.confidence` for direct blobs and `canonical_aliases.confidence` for alias canonicals. The build summary and shopping list `Confidence` column both reflect these values.

**Still to address:**

- **Size matching** — the manifest sometimes declares `size` even when no
  hashes are known. A filename + size agreement is a meaningfully stronger
  signal than filename alone and costs nothing to check.

- **CRC32 as a lightweight secondary hash** — some platforms declare CRC32
  but not MD5. Not cryptographically strong, but a CRC32 + filename match is
  substantially better than filename alone and is already present in the
  manifest data model.

- **Cross-platform corroboration against manifest declared hashes** — if the same
  filename appears across multiple platforms all declaring the same MD5, and an
  unverifiable canonical matches that filename and size, the probability it
  represents the same file is high.

- **User-confirmable flagging** — for cases where none of the above signals
  reach a high-confidence threshold, surface the candidate in the report as
  \"probable duplicate — recommend review\" rather than auto-correcting. A human
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

## 3. Emulator-level preservation tool (new parallel tool)

**Status:** Design complete — ready to implement

**Context:** The platform tool is top-down: it answers "what does Batocera/RetroDECK/etc. expect?" The emulator tool is bottom-up: it answers "what does each emulator actually load at runtime, verified from source code?" These are complementary axes and intentionally kept as separate tools to avoid entangling their different logic, status models, and lifecycle concerns.

Source data: the `emulators/` directory in Abdess/retrobios (~330 YAML files, one per emulator). These are source-verified profiles — every file entry traces to a specific line of emulator source code.

---

### Agreed design decisions

**Architecture — separate tool, separate database**
A completely independent tool (`emulator_preservation/`) with its own sqlar database. The platform tool's purge logic, alias reconciliation, verify pass, and status hierarchy are not shared. No runtime dependency between the two tools. Cross-referencing the platform database is a future option, not a v1 requirement.

**Update step — fetch all emulator YAMLs at once**
All ~330 YAMLs are fetched from `emulators/` on every update run (no selective fetching). Files are tiny; the whole directory is well under 1 MB. Cached locally in `emulator_yaml_cache/`. Re-download gated by `yaml_refresh = true` config flag, same pattern as the platform tool.

Emulators with `files: []` (HLE-only, no BIOS needed) are **silently skipped at update time** — they produce no manifest entries, appear in no reports, and contribute to no counts. The YAML is fetched and cached but otherwise ignored entirely.

The `profiled_date` field is preserved per emulator in the manifest as a staleness indicator.

**Manifest schema — emulator-keyed**
The combined manifest is keyed by emulator name (YAML filename stem). Each entry carries the emulator-level metadata (`display_name`, `type`, `core_classification`, `systems`, `cores`, `profiled_date`, `source`, `upstream`) and a list of file entries. File entries preserve all declared fields from the YAML: `name`, `system`, `size`, `md5`, `sha1`, `sha256`, `crc32`, `known_hash_adler32`, `required`, `hle_fallback`, `aliases`, `mode`, `region`, `description`, `note`, `source_ref`, `validation`, `category`, `agnostic`, `contents`, `path`, `destination`.

The `validation` field is stored as informational metadata only — it does not control what the tool checks. Like the platform tool, verification is driven entirely by the presence of declared hash/size values in the file entry fields.

**Storage key — `{md5}.{ext}`**
Files are hashed on ingest regardless of what checks are declared. MD5 is always computed and used as the storage key, identical to the platform tool convention.

**Verification model — consistent with platform tool**
Check all declared values present in the file entry: `md5`, `sha1`, `sha256`, `crc32`, `size`, `known_hash_adler32`. All declared checks must pass for a file to be `verified`. Same principle as the platform tool's `determine_status()` — the YAML fields are the authority, not the `validation` list.

**Status vocabulary (four states)**
- `verified` — file present, all declared checks pass
- `unvalidated` — file present, at least one declared check fails (covers size mismatch, hash mismatch, or both — future-proof when additional check types are added)
- `unverifiable` — file present, no hash or size declared — nothing to check against
- `missing` — file not found in any scanned source

**Report unit — per emulator**
One CSV report per emulator. Each row is a file that emulator declares. Report columns include the rich context the platform tool lacks: `required`, `hle_fallback`, `description`, `source_ref`, `note`, `validation`, `region`, `mode`.

**System metadata**
The `system` field from each file entry is carried through into the manifest and database. Used for filtering and reporting; not part of the storage key or file identity.

**Selection at build/report time**
With ~330 emulators, a flat toggle list is impractical. Filter by `systems` (Nintendo, Sega, Sony, etc.) or by `core_classification` (`official_port`, `community_fork`, `frozen_snapshot`, etc.) rather than by individual emulator name.

---

### Still to decide before implementation

- **Manifest JSON structure** — exact schema for the combined emulator manifest. Key open question: how to normalise the `validation` field's two possible shapes (list vs dict with `core`/`upstream` keys) at parse time into a consistent internal form.

- **`bios_zip` / `contents` field** — some BIOS entries are ZIPs with declared internal file structure (each member with its own CRC32). V1 treats the ZIP as a single blob and skips inner-file validation. Flagged for a future pass.

- **`agnostic: true` files** — the emulator accepts any file within a size range under the system path. V1 behaviour: if a file is present and within declared size constraints, treat as `verified`. If nothing present, `missing`.

- **`known_hash_adler32`** — Dolphin IPL files use Adler-32. Python's `zlib.adler32()` handles this natively. Confirm before implementation.

- **Shopping list** — equivalent to the platform tool's global shopping list but simpler: no MD5 variant expansion, no per-platform aggregation. One row per missing or unvalidated file entry, with emulator name, filename, required flag, and declared hashes.

- **Staging** — explicitly deferred. No stage step in v1.

- **Cross-reference with platform database** — deferred. When implemented: a one-way read at report time checking the platform sqlar for a matching blob to surface its verification status alongside the emulator tool's own. No shared write path.

---
