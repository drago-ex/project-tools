# Agent Guide

Use this file as package-specific guidance when working in this repository. Keep changes aligned with the existing public API, project conventions, and supported PHP range.

## Package Overview
- **Package**: `drago-ex/project-tools`
- **Type**: library
- **Purpose**: Installation and cleanup tools for Drago projects.
- **PHP**: >=8.3 <9
- **Runtime dependencies**: composer-runtime-api (^2.0)
- **License**: MIT

## Repository Layout
- `bin/`: command-line entry points.
- `.github/`: CI workflows; keep their PHP versions and commands consistent with Composer and the package scripts.
- `composer.json`: package metadata, dependency constraints, autoloading, and available Composer scripts.

## Key Parts
- Design intent: resource packages are minimal starting points; copied files become project code that each project customizes. `composer update` must never overwrite them (`copy` skips existing files, `replace-once` replaces only once). Do not propose moving that code into libraries or overwriting it on update; see "The Idea Behind Drago Packages" in the README.
- `bin/drago-install` copies and replaces package resources into the project (with `--dev` into `.dev-resource/` of the package being developed); `bin/drago-clean` then deletes `resources/` only from packages whose resources are installed (type `drago-tools-resource`, or libraries with install rules when `allow-library-install` is on).
- After a successful `replace-once` package, `drago-install` sets `skip-replace` in the root composer.json. The file is decoded as objects (empty `{}` must not become `[]`) and written with its own indentation (tabs stay tabs), like Composer does.
- `drago-install --dev` also checks the root package's own install rules without copying them (`planOwnResources()`): own files are reported as `planned` with their project path, a missing source or unsafe path as `error`, an own `copy` file that a dependency installs too as `conflict` (dependency files skipped as "provided by this package" are remembered in `$providedByPackage` for that). Destinations in the report are relative to the project (`.dev-resource/` in dev mode).
- `drago-clean` exits without deleting anything when the root package is a `drago-tools-resource`: there the dependencies' `resources/` are the source for the next `drago-install --dev`, which recreates `.dev-resource/`.
- There is no setup runner (`drago-setup` was removed): migrations (`migration db:migrate --packages`, order by `extra.drago-tools.migration-priority`) and permission providers (`create-permission --packages`, declared in `extra.drago-permission`) have their own consoles that do not depend on this package.
- `bin/drago-report` shows the reports in `.drago-project/` (`install.json` written by `drago-install`, `migration.json` by drago-ex/migration, `permission.json` by drago-ex/permission; same header `tool`, `command`, `date`, `result`, `stats`, `entries`) and falls back to the legacy `drago-install.json`.
- `.drago-project/installed.json` (not in dev mode) keeps, across runs, which package version installed each project file: package, version, reference (the commit; tells apart two installs of a branch version such as dev-main; older records without it compare by version only), source path in the package and the hash of the installed content (line endings ignored). Copied and replaced files are recorded; an existing unrecorded file is adopted only when it equals the package file.
- `bin/drago-diff` compares the record with the installed package versions: upstream change = the file in the current version differs from the recorded hash; local change = the project file differs from it. Package versions come from vendor/ or `composer archive` run in a temporary directory (inside the project Composer did not find other versions of resource packages); `--accept` moves the record to the current version and commit. `composer archive` cannot fetch an older commit of a branch, so the diff of one file then compares the project file with the current one (exact when the file was not edited, a warning otherwise). It never writes project files. It also lists files of `replace-once` packages (skip-replace set in the root composer.json) that exist in the current package version but not in the project and are not recorded; packages skipped on purpose (`skip`, or skip-replace without replace-once) are not checked.
- The scripts change files of the consuming project; test them on a copy, never on a real project.

## Dependencies

Composer constraints describe the package contract. Before changing integration behavior, inspect the source and the actual call sites in this package; do not infer runtime usage solely from dependency declarations.

## Essential Commands

```bash
# No test or static-analysis Composer script is defined in this package.
```

## Tests

This repository does not currently define a test script in `composer.json` or contain a `tests/` directory.
Do not claim behavior is covered by tests unless you find and run a relevant suite. When adding a test suite, follow the package’s existing Nette Tester setup if one is present and add an explicit Composer script.

## Conventions & Rules

- Preserve backward compatibility unless the task explicitly permits a breaking change. Check callers and public signatures before changing behavior.
- Follow the existing PHP style: strict types, explicit parameter and return types, tabs for indentation, and single-quoted strings where practical. Match nearby code when a convention is unclear.
- Keep package-specific configuration, resources, and migrations in their established directories. Do not edit generated or consuming-project files without confirming they are part of this repository’s intended output.
- Add or update focused tests for behavioral changes when this package has a runnable test setup. Do not invent test commands; use only the scripts and configuration present here.
- Update README examples and Composer metadata when a public API or supported requirement changes.
