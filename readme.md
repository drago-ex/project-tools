# Drago Project Tools

Tools for automatic installation and cleanup of project resources from Composer packages.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/drago-ex/project-tools/blob/main/license)
[![PHP version](https://badge.fury.io/ph/drago-ex%2Fproject-tools.svg)](https://badge.fury.io/ph/drago-ex%2Fproject-tools)
[![Coding Style](https://github.com/drago-ex/project-tools/actions/workflows/coding-style.yml/badge.svg)](https://github.com/drago-ex/project-tools/actions/workflows/coding-style.yml)

## The Idea Behind Drago Packages

Drago resource packages (`project-auth`, `project-front`, `project-backend` and others) are not libraries you call
from your code. They are starting points: each package contains only the essential, basic parts of one feature, and
`drago-install` copies them into your project, where they become your own code.

Every project is different, so the copied files are meant to be adapted to its needs. That is why each package is
developed separately and kept minimal, and why a later `composer update` does not copy or overwrite files that already
exist in the project: it must never destroy changes made for that project.

- **`copy`** creates files that do not exist yet and never touches existing ones.
- **`replace`** with **`replace-once`** lets a preset compose the project once, without overwriting it on later updates.
- **`replace`** without `replace-once` always overwrites; use it only for files that are not meant to be edited in the project.

When a package releases a fix for files that are already in your project, the update does not apply it for you.
Compare the package files with your copies and take over the changes you want (`drago-report` shows which files came
from which package).

`create-netis` builds on this: it creates a complete project from all packages at once. A project with only some
features can be put together by hand, following the README of each package.

## Requirements

- PHP >= 8.3
- Composer 2

## Features

- **`drago-install`**: Automatically copies or replaces files/directories from vendor packages to your project root.
- **`drago-clean`**: Cleans up redundant resource folders in configured vendor folders to prevent class duplication or namespace collisions.
- **`drago-report`**: Shows the reports in `.drago-project/`: what `drago-install` copied, replaced or skipped,
  and which migrations and permission providers were set up.

## Installation

Add the package to your project:
```bash
composer require drago-ex/project-tools
```
For component development, add it to `require-dev`:
```bash
composer require --dev drago-ex/project-tools
```

## Configuration

In your package's `composer.json`, define what should be installed:

```json
"type": "drago-tools-resource",
"extra": {
    "drago-tools": {
        "install": {
            "copy": {
                "resources/Permission": "app/Core/Permission",
                "resources/assets": "assets/naja"
            },
            "replace-once": true,
            "replace-priority": 100,
            "replace": {
                "resources/config/overrides.neon": "app/config/overrides.neon"
            }
        }
    }
}
```

### Global Options

Configure trusted Composer vendors in your **root** `composer.json`:

```json
"extra": {
    "drago-tools": {
        "vendor-names": ["drago-ex", "netis-cms"],
        "allow-library-install": true
    }
}
```

`vendor-names` limits which packages are scanned by `drago-install` and `drago-clean`.
Default value is `["drago-ex"]`.

If a configured vendor folder does not exist in `vendor/`, the tools print a warning:

```text
WARN Vendor name not found: vendor-name
```

*Note: For security, only packages of type `drago-tools-resource` are allowed to install resources by default. Use `allow-library-install` to enable mirroring for standard libraries from configured vendors.*

### Sections:

- **`copy`**: Copies files only if they do not already exist in the destination. Safe for initial setup.
- **`replace`**: Always overwrites the destination files. Useful for core updates or shared assets.

Source and destination paths must be relative paths inside the package and project. Absolute paths and paths containing `..` are rejected.

### Skipping Packages

To skip a specific package during installation, add it to the `packages` map in your **root** `composer.json`:

```json
"extra": {
    "drago-tools": {
        "packages": {
            "vendor/package-name": {
                "skip": true
            }
        }
    }
}
```

When `skip` is set to `true`, the package is ignored entirely - neither `copy` nor `replace` will run for it.

### Skipping Only Copy or Replace

You can also skip only one install section for a package:

```json
"extra": {
    "drago-tools": {
        "packages": {
            "vendor/package-name": {
                "skip-copy": true,
                "skip-replace": true
            }
        }
    }
}
```

- **`skip-copy`**: Skips only the package `copy` section.
- **`skip-replace`**: Skips only the package `replace` section.
- **`skip`**: Skips the whole package and has priority over section skips.

This is useful for preset packages where files should be overwritten once during initial project composition, but not overwritten again on later Composer updates.

### Install Once

Packages that contain `replace` rules can mark themselves as replace-once:

```json
"extra": {
    "drago-tools": {
        "install": {
            "replace-once": true,
            "replace": {
                "resources/app": "app",
                "resources/assets": "assets"
            }
        }
    }
}
```

After a successful `replace` run, `drago-install` writes this to the root project `composer.json`:

```json
"extra": {
    "drago-tools": {
        "packages": {
            "vendor/package-name": {
                "skip-replace": true
            }
        }
    }
}
```

The next run keeps the package installed, but skips its `replace` section. The safer `copy` section can still run unless `skip-copy` or `skip` is also enabled.

`replace-once` is not applied in `--dev` mode.

### Replace Priority

The installer runs in two phases:

1. All package `copy` sections are processed first.
2. Package `replace` sections are processed afterwards and sorted by priority.

Priority is configured in the package `composer.json`:

```json
"extra": {
    "drago-tools": {
        "install": {
            "replace-priority": 200
        }
    }
}
```

Lower priority runs earlier, higher priority runs later. This means higher-priority overlays win when multiple packages replace the same file.

Default priority is `100`.

Example:

```text
drago-ex/project-backend       priority 100
drago-ex/project-backend-ui    priority 200
netis-cms/project-preset       priority 300
```

In this example, `project-backend-ui` can override backend files, and `netis-cms/project-preset` can override both.

## Usage

### Automation

Add these scripts to your `composer.json` to automate the process:

```json
"scripts": {
    "post-install-cmd": [
        "drago-install",
        "drago-clean"
    ],
    "post-update-cmd": [
        "drago-install",
        "drago-clean"
    ]
}
```

### Manual Execution

You can also run the commands manually from your terminal:

```bash
vendor/bin/drago-install
vendor/bin/drago-clean
vendor/bin/drago-report
```

### Install Report

`drago-install` writes a report of every processed file to `.drago-project/install.json` in the project.
The file is overwritten on each run. `.drago-project/` collects the reports of the tools that set up a project,
so a programmer or an agent sees in one place what happened:

| Report | Written by |
|---|---|
| `install.json` | `drago-install` (this package) |
| `migration.json` | `migration db:migrate --packages` ([drago-ex/migration](https://github.com/drago-ex/migration)) |
| `permission.json` | `create-permission --packages` ([drago-ex/permission](https://github.com/drago-ex/permission)) |

All reports share the header `tool`, `command`, `date`, `result` (`ok` or `error`), `stats` and `entries`.
Each install entry contains the package and its version, section (`copy` or `replace`), action, source and destination:

- `copied`: new file created.
- `replaced`: existing file overwritten (`replace` section).
- `skipped`: file already exists (`copy` section), or the package's `resources/` were already installed and
  removed by `drago-clean`.
- `error`: the file could not be installed, with a `message` (e.g. source not found).

Show the reports with:

```bash
vendor/bin/drago-report
vendor/bin/drago-report --tool=install
vendor/bin/drago-report --package=translator
vendor/bin/drago-report --action=skipped
```

```text
== install: drago-install, 2026-10-18T14:05:12+02:00 ok
drago-ex/translator (v4.1.0)
  copied    app/Core/Translator/Panel.php
  replaced  app/config/translator.neon [replace]
2 copied, 0 replaced, 0 skipped, 0 error
```

Commit `install.json` and `permission.json`: they record where the project's files came from. Add
`.drago-project/migration.json` to `.gitignore`, because it depends on the database of each environment.

`drago-install` used to write `drago-install.json` in the project root; it is no longer written and can be deleted.
`drago-report` still shows it while `.drago-project/install.json` does not exist.

### Cleanup

`drago-clean` removes `resources/` directories from packages in configured vendor folders after their files have been copied or replaced into the project. Only packages whose resources are installed are cleaned: packages of type `drago-tools-resource`, and libraries with install rules when `allow-library-install` is enabled. Other libraries keep their `resources/` directory.

This is useful when resource folders contain PHP classes or other files that are mirrored into the project. Leaving the same classes both in `vendor/` and in the project can cause duplicate class or namespace conflicts.

Only package resource folders are removed. Installed Composer packages remain installed.

Packages installed from local Composer `path` repositories may be symlinked into `vendor/`. In that case, `drago-clean` leaves their resource folders in place so it does not delete files from the source package. This also applies to directory junctions used for symlinks on Windows. Resource folders in regular package copies under `vendor/` are still removed.

To test a local package as it behaves after a regular installation, configure the path repository to mirror packages instead of symlinking them:

```json
"repositories": [
    {
        "type": "path",
        "url": "../packages/*",
        "options": {
            "symlink": false
        }
    }
]
```

With `symlink: false`, Composer copies packages into `vendor/`, so `drago-clean` can remove only the copied resource folders while preserving the source repositories.

### Options

- `--verbose` or `-v`: Show detailed file-by-file progress during installation.
- `--dev`: For developing a resource package. Installs the resources of its dependencies into `.dev-resource/`
  (instead of the project root), so you can check that they install and that all their classes are available,
  e.g. for PHPStan and tests. The folder is recreated on every run, files the package has itself in `resources/`
  are not copied (its own version wins), and the report is written to `.dev-resource/.drago-project/install.json`.
  Add `/.dev-resource/` to `.gitignore`, `scanDirectories: [.dev-resource]` to `phpstan.neon`, and load classes
  from `.dev-resource/app` after `resources/app` in `tests/bootstrap.php`.

  The package's own `copy` and `replace` rules are checked as well, without copying its files (they stay in
  `resources/`, a copy would declare their classes twice). The report lists each own file as `planned` with the place
  it gets in a project, next to what the dependencies installed, so a wrong path in `composer.json` shows up before
  the package reaches a project:

  ```text
  drago-ex/project-auth (v2.6.0)
    copied    app/Model/User/UserEntity.php
    skipped   app/Model/User/UserRepository.php Provided by this package.
  drago-ex/project-account (dev-main)
    planned   app/Presentation/Account/AccountPresenter.php
    planned   app/Model/User/UserRepository.php [replace] Replaces the file of a dependency.
    error     resources/missing Source not found.
    conflict  app/Presentation/Sign/SignPresenter.php A dependency installs the same file; ...
  ```

  `error` is a missing source or an unsafe path; `conflict` is an own file in a `copy` section that a dependency
  installs too, where only the one installed first would be kept (use `replace` to override a dependency's file).
  Either sets `result` to `error`.

  In a resource package (type `drago-tools-resource`), `drago-clean` deletes nothing: the resources of its
  dependencies are not autoloaded there and every `drago-install --dev` run copies them again.

## Project Setup

After the resources are installed, the remaining setup is done by the packages that own it, each with its own
console. Neither needs this package, so they also work in projects assembled without Project Tools:

```bash
# migrations of all installed packages in their order, then the project's migrations/ (drago-ex/migration)
php vendor/bin/migration db:migrate --packages

# permission providers declared by installed packages (drago-ex/permission)
php vendor/bin/create-permission --packages
```

Inside Docker, run them as the web user:

```bash
docker compose exec -u www-data server php vendor/bin/migration db:migrate --packages
docker compose exec -u www-data server php vendor/bin/create-permission --packages
```

Packages take part by declaration only: SQL files in `migrations/` with `migration-priority` (below), and
permission rules in a `permission.php` referenced from `extra.drago-permission`. Both commands write their report
into `.drago-project/`, where `drago-report` shows it.

### Migration Priority

The order of the package migrations is set separately by `migration-priority` (lower first, default `100`):

```json
"extra": {
    "drago-tools": {
        "migration-priority": 10
    }
}
```

It is read by `drago-ex/migration`, so the migrations run in the same order through
`migration db:migrate --packages` and the web installer of `drago-ex/project-install`. A package that uses tables
of another package needs a higher `migration-priority` than that package. Packages released before
`migration-priority` existed set the order by `commands-priority`, which is still used as a fallback.

## How it works

1. `drago-install` scans configured vendors for the `extra.drago-tools.install` configuration.
2. It resolves relative paths against the package root and the current working directory.
3. It skips packages listed under `extra.drago-tools.packages` in the root `composer.json` with `"skip": true`.
4. It runs all `copy` sections first.
5. It runs `replace` sections afterwards, sorted by `install.replace-priority`.
6. It can skip only `copy` or `replace` sections with `skip-copy` and `skip-replace`.
7. It respects the `allow-library-install` flag in your root `composer.json` (defaults to `false` for libraries).
8. Packages marked with `install.replace-once` automatically add `skip-replace: true` to the root `composer.json` after a successful replace run.
9. `drago-clean` targets configured `vendor-names` and removes `resources` directories of installed resource packages to keep your vendor clean after files have been mirrored to your project.
10. `drago-install` records every file action in `.drago-project/install.json`; `drago-report` prints it together with the reports of migrations and permissions.