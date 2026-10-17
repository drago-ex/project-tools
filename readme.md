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
- **`drago-setup`**: Collects and runs setup commands provided by installed Drago packages.
- **`drago-report`**: Shows what the last `drago-install` run copied, replaced or skipped.

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

`vendor-names` limits which packages are scanned by `drago-install`, `drago-clean` and `drago-setup`.
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
vendor/bin/drago-setup
vendor/bin/drago-report
```

### Install Report

`drago-install` writes a log of every processed file to `drago-install.json` in the project root. The file is overwritten on each run.

Each entry contains the package, section (`copy` or `replace`), action, source and destination:

- `copied`: new file created.
- `replaced`: existing file overwritten (`replace` section).
- `skipped`: file already exists (`copy` section), or the package's `resources/` were already installed and
  removed by `drago-clean`.
- `error`: source not found.

Show it with:

```bash
vendor/bin/drago-report
vendor/bin/drago-report --package=translator
vendor/bin/drago-report --action=skipped
```

```text
drago-ex/translator
  copied    app/Core/Translator/Panel.php
  replaced  app/config/translator.neon [replace]
```

Add `drago-install.json` to `.gitignore` if you do not want it committed.

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
- `--dev`: Switches the destination directory to `resources/` in the current working directory. Useful for testing installation logic during package development.

## Setup Commands

Packages can expose setup commands in `extra.drago-tools.commands`.
These commands can be used for database migrations, generated permission classes or any other post-installation task.

```json
"extra": {
    "drago-tools": {
        "commands-priority": 10,
        "commands": {
            "db:migrate-auth": "php vendor/bin/migration db:migrate vendor/drago-ex/project-auth/migrations",
            "create:auth-permission": "php vendor/bin/create-auth-permission"
        }
    }
}
```

Run the setup tool from the project root:

```bash
vendor/bin/drago-setup
```

Inside Docker, run it as the web user:

```bash
docker compose exec -u www-data server php vendor/bin/drago-setup
```

### Command Priority

The package-level `commands-priority` value controls the order of setup commands from that package.
Lower priority runs earlier. Default priority is `100`.

Commands are sorted in two groups:

1. Commands with labels starting with `db:` run first.
   Use this prefix for database migrations and other database setup tasks.
2. All other commands run afterwards, for example `create:*`, `generate:*` or `get:*`.

Inside each group, commands are sorted by `commands-priority` first and by command label alphabetically second.
This keeps migrations before application setup while still making the console order predictable.

Command values can be plain strings. Object values are also supported for readability.
Supported object keys are `run`, `command` and `bin`; `run` is the preferred form.

### Setup Interaction

- Select specific tasks by number, for example `1,3`.
- Enter `a` to run all tasks sequentially.
- Enter `q` to quit.

`drago-setup` does not inspect the database or Nette container.
It only discovers package commands and executes the selected shell command.
Database-specific behavior, such as creating the `migrations` table, belongs to the migration command itself.

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
10. `drago-install` logs every file action to `drago-install.json`; `drago-report` prints it.
11. `drago-setup` scans configured vendors for `extra.drago-tools.commands` and runs selected setup commands in priority order.
