# Bettergist — Schema Reference

**Purpose:** A PHP package analysis platform that mirrors Packagist metadata and runs automated quality/code analysis on open-source PHP packages.

**DB:** PostgreSQL 18.3 · Owner: `bettergist` · Schema: `public`

---

## Core Concept

Every table revolves around a single universal key: `package` (varchar), formatted as `"vendor/project"` (e.g. `laravel/framework`). Nearly all tables use this as their PK and FK back to `packages`.

---

## Tables

### `packages` ← **Central Entity**
| Column | Type | Notes |
|---|---|---|
| `package` | varchar | PK — `"vendor/project"` |
| `vendor` | varchar(255) | |
| `project` | varchar(255) | |
| `last_fetched` | timestamptz | Last Packagist fetch |
| `is_abandoned` | boolean | Default false |
| `created_at`, `updated_at` | timestamptz | Auto-managed |

Unique constraint on `(vendor, project)`. All other tables cascade delete/update from here.

---

### `packagist_packages`
Crawl queue / existence tracker for Packagist. Just `(package, updated_at)` — used to track which packages have been seen on Packagist.

### `packagist_stats`
Packagist-sourced popularity metadata per package.
`project_type`, `language`, `installs`, `dependents`, `github_stars`, `deps_core`, `deps_dev`

### `package_stats`
`disk_space` (integer, KB), `git_host` (varchar) — filesystem footprint per package.

---

### `code_stats` — Static Analysis Metrics
| Column | Notes |
|---|---|
| `loc`, `loc_comment`, `loc_active` | Lines of code |
| `classes`, `methods`, `functions`, `anonymous` | Structure counts |
| `avg_class_loc`, `avg_method_loc`, `avg_function_loc` | Averages |
| `cyclomatic_class`, `cyclomatic_method` | Complexity scores (float) |

### `code_quality` — CI/Quality Results
| Column | Notes |
|---|---|
| `composer_failed` | Error message if `composer install` failed |
| `has_tests` | Boolean |
| `phpunit_passes` | Boolean |
| `phpstan_level` | Integer (0–9) |

> `code_quality_v1` is a legacy archive copy without constraints.

---

### `dependencies`
Composer `require` / `require-dev` entries.
PK: `(package, dependency, is_dev)` · `constraints` = raw composer constraint string (e.g. `^2.0`)

### `dependency_constraints`
Parsed version constraint bounds derived from `dependencies`.
PK: `(package, dependency, is_dev)` · Fields: `min_version`, `min_inclusive`, `max_version`, `max_inclusive`, `constraints`

---

### `supported_php_versions`
PK: `(package, version)` — `version` is `numeric(4,1)` (e.g. `8.1`). `installable` boolean.

### `required_extensions`
PK: `(package, extension)` — PHP extensions declared in `composer.json` (e.g. `ext-json`).

### `licenses`
PK: `(package, license)` — SPDX identifiers (e.g. `MIT`). A package can have multiple.

### `vcs_repos`
`(package, type, repo_url)` — VCS source (type = `git`, `svn`, etc.).

---

### `disqualified_packages`
Packages excluded from analysis. `reason` (varchar 255) + `details` (text). FKs cascade from `packages`.

### `last_run_log`
PK: `package` — timestamp of the most recent analysis pipeline run.

---

### `users`
Application users. `(id serial PK, name, username, email, password, role_id int, is_active int)`. No FK to a roles table in this schema.

### `migrations`
Laravel-style migration log. `(id serial PK, migration varchar, batch int)`.

---

## Views (Reporting)

| View | Description |
|---|---|
| `disk_space` | Total MB grouped by first letter of package name |
| `disk_space_995_percentile` | 99.5th percentile disk usage across all packages |
| `lost_packages` | Disqualified packages that still have `code_stats` rows |
| `report_downloads_by_date` | Count of packages fetched per calendar date |
| `report_huge_vendors` | Vendors with ≥250 MB total size, ranked by `badness_score` = `total_mb / (total_stars + 1)` |
| `report_packages_by_stars_range` | Package + disk space counts bucketed by GitHub stars (0, 1-9, 10-99 … 10k+) |
| `report_popularity` | All packages ordered by `installs DESC` |
| `report_top_vendors` | Pivoted vendor project counts for snapshots: 2020-05, 2021-12, 2023-03, 2024-02 |
| `report_notable_vendors` | Source table for `report_top_vendors` — `(month char(7), vendor, project_count)` |
| `report_unanalyzed_packages` | Packages with no `code_stats` or stale stats (>3 days), not disqualified |
| `report_vendors_most_projects` | All vendors ranked by project count |
| `report_version_ranges` | PHP version range combinations (min→max) with package counts |
| `vendors_most_projects` | Top 10 vendors by project count |

---

## Infrastructure

**Trigger:** `trigger_set_timestamp()` — BEFORE UPDATE trigger that sets `updated_at = NOW()`. Applied to: `code_quality`, `code_stats`, `dependencies`, `dependency_constraints`, `disqualified_packages`, `last_run_log`, `package_stats`, `packages`, `packagist_packages`, `packagist_stats`.

**ENUM:** `php_version` — `'5.6', '7.0', '7.1', '7.2', '7.3', '7.4', '8.0', '8.1'` (defined but not currently used as a column type in active tables).

**Extension:** `tablefunc` (crosstab support).

---

## Legacy / Archive Tables
These have no constraints or FKs and are historical snapshots:
- `code_quality_v1` — older quality run results
- `raw_code_stats_v1` — uses `package_id char(22)` instead of `package varchar`
- `raw_packagist_stats_v1` — same old ID scheme
- `packagist_stats_old` — all-nullable copy of `packagist_stats`
- `disk_space_over_time` — `(package_id char(22), dec_2020, dec_2021)`
- `migrations_backup` — all-nullable copy of `migrations`

---

## Key Relationships (FK Graph)

```
packages (PK: package)
  ├── packagist_stats       CASCADE UPDATE/DELETE
  ├── package_stats         CASCADE UPDATE/DELETE
  ├── code_stats            CASCADE UPDATE/DELETE
  ├── code_quality          CASCADE UPDATE/DELETE
  ├── dependencies          CASCADE DELETE
  ├── dependency_constraints CASCADE UPDATE/DELETE
  ├── supported_php_versions CASCADE UPDATE/DELETE
  ├── required_extensions   CASCADE UPDATE/DELETE
  ├── licenses              CASCADE UPDATE/DELETE
  ├── vcs_repos             CASCADE DELETE
  ├── disqualified_packages CASCADE UPDATE/DELETE
  └── last_run_log          CASCADE DELETE
```

---

## Common Query Patterns

```sql
-- Full package profile
SELECT p.*, ps.installs, ps.github_stars, pst.disk_space,
       cq.phpstan_level, cq.phpunit_passes, cs.loc, cs.cyclomatic_method
FROM packages p
JOIN packagist_stats ps ON ps.package = p.package
JOIN package_stats pst  ON pst.package = p.package
LEFT JOIN code_quality cq ON cq.package = p.package
LEFT JOIN code_stats cs   ON cs.package = p.package
WHERE p.package = 'vendor/project';

-- Packages pending analysis
SELECT * FROM report_unanalyzed_packages LIMIT 100;

-- Top packages by quality signal
SELECT cq.package, cq.phpstan_level, ps.installs, ps.github_stars
FROM code_quality cq
JOIN packagist_stats ps ON ps.package = cq.package
WHERE cq.phpstan_level >= 5
ORDER BY ps.installs DESC;

-- All dependencies of a package (prod only)
SELECT dependency, constraints
FROM dependencies
WHERE package = 'vendor/project' AND is_dev = false;
```
