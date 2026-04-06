# Developer Notes — BIOS Preservation Tool

Technical reference for contributors and anyone extending the tool. Assumes familiarity with the user-facing README.

---

## Table of Contents

1. [Database Schema](#database-schema)
2. [File Identity Model](#file-identity-model)
3. [Manifest JSON Structure](#manifest-json-structure)
4. [Source YAML Structure](#source-yaml-structure)
5. [CSV Column Reference](#csv-column-reference)
6. [Implementation Notes](#implementation-notes)

---

## Database Schema

The database is a standard SQLite file using the [sqlar](https://sqlite.org/sqlar.html) schema for blob storage, extended with additional tables for file tracking.

### `sqlar` — blob storage

```sql
CREATE TABLE sqlar (
    name  TEXT PRIMARY KEY,   -- storage key: "{md5}.{ext}", e.g. "a5c85cf57b56a98e.bin"
    mode  INT,                -- Unix file mode (always 0)
    mtime INT,                -- modification time (always 0)
    sz    INT,                -- uncompressed size in bytes
    data  BLOB                -- raw file content (uncompressed)
);
```

### `files` — canonical file registry

```sql
CREATE TABLE files (
    sqlar_name      TEXT PRIMARY KEY,  -- references sqlar.name
    canonical_name  TEXT NOT NULL,     -- lowercase canonical identifier
    status          TEXT NOT NULL,     -- "verified" | "unverifiable" | "mismatch_accepted"
    sha1            TEXT,
    md5             TEXT,
    sha256          TEXT,
    crc32           TEXT,
    size            INTEGER
);
```

A canonical may have multiple rows in `files` (one per verified regional variant). The `sqlar_name` primary key is always an MD5-based filename — two variants with different MD5s coexist naturally.

### `missing_files` — canonicals not yet found

```sql
CREATE TABLE missing_files (
    canonical_name TEXT PRIMARY KEY
);
```

Populated at the end of each build run for canonicals present in the manifest but absent from `files`. Cleared and repopulated on every build.

### `canonical_aliases` — alias resolution

```sql
CREATE TABLE canonical_aliases (
    canonical_name  TEXT NOT NULL,
    sqlar_name      TEXT NOT NULL,
    PRIMARY KEY (canonical_name, sqlar_name)
);
```

Populated when `_store()` detects that the same blob (same MD5) is already stored under a different `canonical_name`. The incoming canonical is recorded here so report lookups can resolve it without creating a duplicate `files` row.

### `meta` — build metadata

```sql
CREATE TABLE meta (
    key   TEXT PRIMARY KEY,
    value TEXT
);
```

Stores `generated_at` (ISO 8601 timestamp of the last manifest generation) and `unrar_tool` (path to the UnRAR binary, if saved). The `generated_at` value is used to detect manifest regeneration between build runs and trigger an orphan purge.

---

## File Identity Model

**Canonical** — the lowercase filename used as the logical identity of a BIOS file across all platforms. Derived from the first `name:` field encountered for a given hash identity in the manifest. A canonical may map to many different filenames across platforms (e.g. `scph5500.bin` and `sony-playstation:8d8cb7fe...`).

**Blob** — a row in `files` + the corresponding row in `sqlar`. Keyed by `{md5}.{ext}`. Multiple blobs can exist for one canonical when platforms declare different accepted MD5s (regional variants). All are stored; staging picks the best match for each platform.

**Alias canonical** — a canonical whose bytes are already stored under a different `canonical_name`. Recorded in `canonical_aliases`. This happens when two manifest entries (different names, same MD5) resolve to the same physical file. The report step uses `canonical_aliases` as a 4th fallback lookup.

### `_get_file_rows()` lookup chain (bios_report.py)

1. Direct `canonical_name` lookup in `files`
2. Hash-based fallback — query `files` for any declared hash from the platform manifest
3. `database_filename` fallback — query `files.sqlar_name` using the manifest's stored filename field
4. `canonical_aliases` fallback — resolve via the alias table

### Status hierarchy

```
verified  >  unverifiable  >  mismatch_accepted
```

Status only ever upgrades. When a higher-status blob supersedes a lower-status one, the lower blob is deleted from `sqlar` and `files`. Two `verified` blobs for the same canonical (different MD5s) coexist — neither supersedes the other.

---

## Manifest JSON Structure

`combined_platform_manifest.json` (Update output) and `combined_platform_build.json` (Build output) share the same structure. Build adds real `database_filename` values; Update uses zero-padded integer placeholders.

```json
{
  "generated_at": "2026-03-25T17:22:00Z",
  "files": {
    "scph5500.bin": {
      "database_filename": "8dd7d5296a650fac7319bce665a6a53c.bin",
      "size": 524288,
      "sha1": "9c0421858e217805f4abe18698afea8d5aa36ff7",
      "md5": "8dd7d5296a650fac7319bce665a6a53c",
      "sha256": null,
      "crc32": "37157331",
      "platforms": {
        "batocera": {
          "known_file": true,
          "staging_paths": ["scph5500.bin"],
          "expected_hashes": {
            "md5":    ["8dd7d5296a650fac7319bce665a6a53c"],
            "sha1":   ["9c0421858e217805f4abe18698afea8d5aa36ff7"],
            "sha256": [],
            "crc32":  ["37157331"]
          },
          "expected_size": 524288
        },
        "retrodeck": {
          "known_file": true,
          "staging_paths": ["scph5500.bin"],
          "expected_hashes": { ... }
        }
      }
    }
  },
  "platform_metadata": {
    "batocera": {
      "base_destination": "bios"
    }
  }
}
```

Key points:
- Top-level keys are lowercase canonical names
- `database_filename` is `{md5}.{ext}` after Build, or a zero-padded placeholder before
- `platforms` only contains entries for platforms that declare this file (`known_file: true`)
- `expected_hashes` is always a dict of lists, even for single values
- `platform_metadata.base_destination` is used by Stage to construct the full output path

---

## Source YAML Structure

Platform YAML files from Abdess/retrobios follow this schema:

```yaml
platform:          "Batocera"
base_destination:  "bios"        # root staging path for this platform
hash_type:         "md5"
verification_mode: "md5"

systems:
  sony-playstation:
    files:
      - name:        scph5500.bin      # filename the platform uses (case-preserved)
        destination: scph5500.bin      # path relative to base_destination
        required:    true
        md5:         8dd7d5296a650fac7319bce665a6a53c
        sha1:        9c0421858e217805f4abe18698afea8d5aa36ff7
        crc32:       37157331
        size:        524288             # bytes, optional
```

Key schema details:
- `name` is what the platform calls the file (case-sensitive, always preserved in output)
- `destination` is the path relative to `base_destination` (may include subdirectories, e.g. `np2kai/bios.rom`)
- Hash fields (`md5`, `sha1`, `sha256`, `crc32`) are optional. Multiple accepted values are comma-separated in a single string
- `size` in bytes is optional
- `inherits: parent_platform` — child declarations take precedence over parent's per filename
- `includes: [group_name]` — expands shared file groups from `_shared.yml`

### Shared groups (`_shared.yml`)

Defines file groups used across multiple platforms and systems. Groups use a bare list (no `files:` wrapper):

```yaml
shared_groups:
  np2kai:
    - name: bios.rom
      destination: np2kai/bios.rom
      required: true
      md5: cd237e16e7e77c06bb58540e9e9fca68
```

The subdirectory a BIOS goes into is determined by the libretro core, not the platform. Shared groups encode the correct destination once so platforms cannot drift. Examples: `np2kai`, `keropi`, `quasi88`, `fuse`, `kronos`, `ep128emu`, `mt32`, `jiffydos`.

---

## CSV Column Reference

### `combined_platform_manifest.csv` / `combined_platform_build.csv`

One row per canonical. Columns:

| Column | Description |
|--------|-------------|
| `database_filename` | `{md5}.{ext}` (build) or zero-padded placeholder (update) |
| `size` | Declared size in bytes, or `unknown` |
| `sha1` | Best known SHA1, or `unknown` |
| `md5` | Best known MD5, or `unknown` |
| `sha256` | Best known SHA256, or `unknown` |
| `crc32` | Best known CRC32, or `unknown` |
| `{platform}_known_file` | `Yes` or `not present` |
| `{platform}_aliases` | Comma-separated staging-path basenames |
| `{platform}_staging_path` | Comma-separated full staging paths |
| `{platform}_expected_hashes` | `ht:value,ht:value,...` or `unverifiable` |

Repeated 4× for each platform (40 platform columns for 10 platforms, plus 6 database columns = 46 total).

### `report/<platform>_report.csv`

One row per staging path, expanded per declared MD5 variant when mismatched. Three comment header lines precede the data:

```
# PHYSICAL FILES (matches build): total=X present=Y verified=A unverifiable=B hash_mismatch=C missing=D
# MANIFEST ENTRIES: total=X present=Y missing=Z hash_mismatch=W
# STAGING PATHS: total=X present=Y missing=Z (counts reflect unique staging paths; actual CSV rows may be higher ...)
```

Data columns: `filename`, `present`, `staging_path`, `actual size`, `expected size`, `actual sha1`, `expected sha1`, `actual md5`, `expected md5`, `actual sha256`, `expected sha256`, `actual crc32`, `expected crc32`.

### `report/global_shopping_list.csv`

One row per distinct declared MD5 that is not yet stored as verified.

| Column | Description |
|--------|-------------|
| `Known Aliases` | Comma-sorted staging-path basenames; canonical added if not already present (case-insensitive) |
| `Expected MD5` | The MD5 to search for, or `unknown` if no MD5 declared |
| `Status` | `missing` / `hash_mismatch` / `unverifiable` |
| `Platforms` | Comma-sorted platform display names |
| `Actual MD5` | MD5 of what is currently stored, or `not present` |

---

## Implementation Notes

### 1. Filename normalisation

All filename comparisons are performed on lowercase-normalised strings internally. All output (CSV, JSON, staged filenames) preserves original case exactly as declared in the YAML source.

### 2. Storage key convention

The `{md5}.{ext}` storage convention in the sqlar eliminates case-sensitivity ambiguity and duplicate-filename collisions at the storage layer. The extension is taken from the canonical name; `.bin` is used as a fallback when the canonical has no extension.

### 3. Inheritance resolution

When a platform YAML uses `inherits:`, the parent is fully resolved before the child is processed. The child's declarations take precedence over the parent's for the same filename (matched case-insensitively). Circular inheritance is detected and stopped with a warning.

### 4. `_data_dirs.yml` is out of scope

Asset data directories (Dolphin/PPSSPP/blueMSX game data packs) are defined in `_data_dirs.yml` in the upstream project. This tool processes only individual BIOS files, not data directories.

### 5. Status upgrade path

Status rank: `verified (1) > unverifiable (2) > mismatch_accepted (3)`. A lower rank number is better. Status only ever moves to a lower number. A `mismatch_accepted` blob is deleted when any `verified` blob for the same canonical is stored.

### 6. Multi-variant coexistence

Two `verified` blobs for the same canonical (different MD5s) coexist without one superseding the other. `_cleanup_superseded()` only removes `mismatch_accepted` blobs when a `verified` copy exists — it never removes a `verified` blob to make room for another `verified` blob.

Non-verified blobs (`unverifiable`, `mismatch_accepted`) are limited to one per canonical. If a second non-verified blob arrives for the same canonical, `_should_store()` rejects it unless it is a strict status upgrade (e.g. `mismatch_accepted` → `unverifiable`). This constraint prevents duplicate blobs with the same canonical name from colliding in the dump zip and from causing false inflation in collection counts.

### 7. Pre-existing snapshot for upgrade counting

`Scanner.__init__` takes a `pre_existing` snapshot (`frozenset(found)`) before scanning begins. The `total_upgraded` counter is only incremented when the canonical was in `pre_existing`. This prevents pass 1 → pass 2 within-run promotions from being counted as cross-run upgrades.

### 8. `_should_store()` logic

Three conditions must all pass for a blob to be stored:
1. This exact MD5 is not already stored with the same status
2. The incoming status is not lower than the best existing status for this canonical (no downgrading a verified canonical with unverifiable/mismatch data)
3. For non-verified blobs: no blob of equal or better non-verified status already exists for this canonical. Only `verified` blobs may coexist (regional variants with distinct MD5s). A strict status upgrade (e.g. `mismatch_accepted` → `unverifiable`) is allowed; a same-status or lower-status duplicate is rejected. This prevents duplicate `unverifiable` or `mismatch_accepted` blobs from accumulating across multiple source scans, which would cause sqlar bloat and path collisions during the dump stage.

### 9. Source management and persistence

Source paths are managed interactively at the start of each Build run and persisted to `bios_preservation_user.conf` automatically as `source_1`, `source_2`, … `source_N`. They can also be set manually in the conf file.

### 10. Manifest regeneration detection

The build step reads `generated_at` from the `meta` table and compares it to the timestamp in the incoming manifest. If they differ, the manifest has been regenerated and `_purge_orphans()` is run before scanning — removing blobs whose `canonical_name` is no longer present in the current manifest.

### 11. Backup and dump naming

Backup files: `DD_mon_YYYY_backup.sqlar`. Dump files: `DD_mon_YYYY_dump.zip`. Both are cross-platform safe. If a file with today's date already exists, a counter suffix is appended (`(2)`, `(3)`, …). No existing file is ever overwritten.

### 12. 7z extraction and temp directory

py7zr 1.x removed the `read()` API. The tool uses `extractall()` to a subdirectory inside `temp/` for all `.7z` scanning. This avoids the RAM-backed `/tmp` filesystem on Linux (often capped at 50% of RAM). The temp directory is always cleaned up in a `finally` block. On out-of-space errors the error message explicitly names `temp_dir` as the setting to change.

### 13. YAML cache directory naming

The cache directory is named `yaml_cache/` rather than `yaml/`. Python treats any folder on `sys.path` as a potential package — a folder named `yaml/` in the project root would shadow the PyYAML library, causing `AttributeError: module 'yaml' has no attribute 'safe_load'`. Any existing installations with a `yaml/` folder should rename it and update `yaml_local_dir` and `yaml_cache_dir` in `bios_preservation.conf`.

### 14. Alias canonical lookup

When `_store()` finds that the incoming blob's MD5 already exists in `sqlar` under a different `canonical_name`, it records the new name in `canonical_aliases` (rather than overwriting the existing entry). `_get_file_rows()` in `bios_report.py` uses `canonical_aliases` as its 4th fallback lookup, after direct canonical name, hash-based, and `database_filename` lookups all fail. `write_build_manifest()` also joins against `canonical_aliases` to fill `database_filename` for alias canonicals.

The `canonical_aliases` table is populated only during active scanning. On a fresh database it will be empty until the first build run. A full rebuild (`incremental = false`) guarantees complete population.

### 15. Shopping list status determination

`_sl_status_for_platform()` re-evaluates status for each (canonical, platform) pair using only that platform's declared hashes — not the global DB status stored at scan time. This ensures that a file verified by Platform A but undeclared by Platform B correctly appears as `unverifiable` from Platform B's perspective.

The shopping list uses a two-bucket accumulation per canonical across all enabled platforms:

- `per_md5` — one entry per distinct declared MD5 that didn't match. Emitted regardless of `any_verified`. Already-verified MD5s (present in `verified_md5s`) are skipped.
- `no_md5` — collects platforms that declare no MD5. Emitted only when `any_verified = False` **and** `per_md5` is empty.

When a platform declares non-MD5 hashes only (e.g. SHA1 but no MD5), and those hashes don't match, the status is converted from `hash_mismatch` to `unverifiable` before entering the `no_md5` bucket — there is no MD5 to express what to search for.

A final sanity pass before CSV write corrects any `unverifiable + known expected MD5` combination to `hash_mismatch` (the two states are mutually exclusive by definition).

### 16. Report row expansion for multi-variant mismatches

When a file is present but has multiple declared MD5 variants and none match what is stored, the per-platform report emits one row per declared MD5. This makes each acceptable regional version a distinct, searchable row. The `# STAGING PATHS` header comment notes that actual CSV row count may exceed unique staging path count for this reason.
