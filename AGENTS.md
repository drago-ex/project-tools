# Agent Guide

Use this file as package-specific guidance when working in this repository. Keep changes aligned with the existing public API, project conventions, and supported PHP range.

## Package Overview
- **Package**: `drago-ex/project-tools`
- **Type**: library
- **Purpose**: Installation and cleanup tools for Drago projects.
- **PHP**: >=8.3
- **Runtime dependencies**: composer-runtime-api (^2.0)
- **License**: MIT

## Repository Layout
- `bin/`: command-line entry points.
- `.github/`: CI workflows; keep their PHP versions and commands consistent with Composer and the package scripts.
- `composer.json`: package metadata, dependency constraints, autoloading, and available Composer scripts.

## Key Parts
- Design intent: resource packages are minimal starting points; copied files become project code that each project customizes. `composer update` must never overwrite them (`copy` skips existing files, `replace-once` replaces only once). Do not propose moving that code into libraries or overwriting it on update; see "The Idea Behind Drago Packages" in the README.
- `bin/drago-install` copies and replaces package resources into the project; `bin/drago-clean` then deletes `resources/` only from packages whose resources are installed (type `drago-tools-resource`, or libraries with install rules when `allow-library-install` is on).
- `bin/drago-setup` runs package commands from `extra.drago-tools.commands`; `bin/drago-report` shows the last install log (`drago-install.json`).
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
