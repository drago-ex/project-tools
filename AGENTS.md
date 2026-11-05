# Agent Guide

`drago-ex/project-tools` (library, PHP >=8.3 <9, only `composer-runtime-api`): the commands that copy the files
of resource packages into projects and keep track of them. The workspace rules are in `../AGENTS.md`.

## Layout
- `bin/drago-install`, `drago-clean`, `drago-commands`, `drago-report`, `drago-diff`: self-contained PHP scripts,
  no `src/`, no tests.

## Design
- Resource packages are minimal starting points; copied files become project code that each project customizes.
  `composer update` must never overwrite them (`copy` skips existing files, `replace-once` replaces only once).
  Do not propose moving that code into libraries or overwriting it on update.
- The scripts change files of the consuming project; test them on a copy, never on a real project.

## Commands
- `drago-install` copies and replaces package resources into the project (with `--dev` into `.dev-resource/` of the
  package being developed). After a successful `replace-once` package it sets `skip-replace` in the root
  composer.json, decoded as objects (an empty `{}` must not become `[]`) and written with its own indentation.
  `--dev` also checks the root package's own rules without copying (`planOwnResources()`): own files are
  `planned` with their project path, a missing source or unsafe path is `error`, an own `copy` file that a
  dependency installs too is `conflict` (tracked by `$providedByPackage`). Report destinations are relative to the
  project.
- `drago-clean` deletes `resources/` only from packages whose resources are installed (type
  `drago-tools-resource`, or libraries with install rules when `allow-library-install` is on), skips symlinked
  packages, and deletes nothing when the root package is a `drago-tools-resource` (there the dependencies'
  `resources/` are the source of the next `drago-install --dev`).
- `drago-commands` runs the PHP scripts that installed packages list in `extra.drago-tools.commands` (a list, or an
  object whose keys are names for the output), from the project root, packages in name order, with `PHP_BINARY`
  through `proc_open` (no shell). No menu and no selection: it runs everything it finds. Same vendor filter and
  `packages.<name>.skip` as `drago-install`; unsafe paths and missing scripts are errors; exit code 1 when a script
  fails. Writes `.drago-project/commands.json`. No drago-ex package uses it at the moment.
- `drago-report` shows the reports in `.drago-project/` (`install.json`, `migration.json` of drago-ex/migration,
  `commands.json`; header `tool`, `command`, `date`, `result`, `stats`, `entries`) and still reads the older
  `drago-install.json`.
- `.drago-project/installed.json` (not in dev mode) keeps which package version installed each project file:
  package, version, reference (commit, tells apart installs of a branch version), source path and the hash of the
  content (line endings ignored). An existing unrecorded file is adopted only when it equals the package file.
- `drago-diff`: upstream change = the file of the current package version differs from the recorded hash; local
  change = the project file differs from it. Package versions come from vendor/ or `composer archive` in a
  temporary directory; `--accept` moves the record to the current version. An older commit of a branch cannot be
  fetched, so the diff of one file then compares the project file with the current one. It never writes project
  files. It also lists files of `replace-once` packages that are new in the current version and not in the project.
- Migrations are not here: drago-ex/migration runs them (`db:migrate --packages`) and reads
  `extra.drago-tools.migration-priority`.
