# Drago Project Tools

Tools for automatic installation and cleanup of project resources from Composer packages.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/drago-ex/project-tools/blob/main/license)
[![PHP version](https://badge.fury.io/ph/drago-ex%2Fproject-tools.svg)](https://badge.fury.io/ph/drago-ex%2Fproject-tools)
[![Coding Style](https://github.com/drago-ex/project-tools/actions/workflows/coding-style.yml/badge.svg)](https://github.com/drago-ex/project-tools/actions/workflows/coding-style.yml)

## The Idea Behind Drago Packages

Drago resource packages (`project-auth`, `project-front`, `project-backend` and others) are not libraries you call
from your code. Each contains only the basic parts of one feature, and `drago-install` copies them into your project,
where they become your own code to adapt. A later `composer update` never overwrites files that already exist in the
project, so a fix released in a package has to be taken over by hand; `drago-diff` shows which files it concerns.

## Requirements

- PHP >= 8.3
- Composer 2

## Features

- **`drago-install`**: Automatically copies or replaces files/directories from vendor packages to your project root.
- **`drago-clean`**: Cleans up redundant resource folders in configured vendor folders to prevent class duplication or namespace collisions.
- **`drago-report`**: Shows the reports in `.drago-project/`: what `drago-install` copied, replaced or skipped,
  and which migrations and permission providers were set up.
- **`drago-diff`**: Shows the project files whose package changed them since they were installed.

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
vendor/bin/drago-diff
```

### Options

- `--verbose` or `-v`: Show detailed file-by-file progress during installation.
- `--dev`: For developing a resource package, see below.

### Report

`drago-install` writes what it copied, replaced or skipped to `.drago-project/install.json`. The migrations
(`drago-ex/migration`) and permissions (`drago-ex/permission`) write their reports to the same folder.
Show them with:

```bash
vendor/bin/drago-report
vendor/bin/drago-report --tool=install
vendor/bin/drago-report --package=translator
vendor/bin/drago-report --action=skipped
```

Commit `.drago-project/`, except `migration.json`, which differs per database.

### Package Updates

`drago-install` records in `.drago-project/installed.json` which package version installed each file. After
`composer update`, `drago-diff` lists the files whose package changed them, and whether you edited them:

```bash
vendor/bin/drago-diff                          # files changed in their packages
vendor/bin/drago-diff app/Presentation/Sign/SignPresenter.php            # what the package changed
vendor/bin/drago-diff app/Presentation/Sign/SignPresenter.php --project  # your file against the new version
vendor/bin/drago-diff --accept app/Presentation/Sign/SignPresenter.php   # after taking the changes over
```

Nothing is overwritten; take the changes over by hand. The package versions are fetched by `composer archive`.

### Cleanup

`drago-clean` removes `resources/` directories of installed resource packages from `vendor/`, so the same classes
are not both in `vendor/` and in the project. Symlinked packages (Composer `path` repositories) are left untouched.

## Developing a Resource Package

In a resource package, run `drago-install --dev`. It copies the resources of its dependencies into `.dev-resource/`,
so PHPStan and tests can use their classes, and checks the package's own `copy` and `replace` rules: the report
in `.dev-resource/.drago-project/install.json` shows where each own file goes in a project, a missing source
and a file that a dependency installs too.

Add `/.dev-resource/` to `.gitignore`, `scanDirectories: [.dev-resource]` to `phpstan.neon`, and load classes
from `.dev-resource/app` after `resources/app` in `tests/bootstrap.php`. `drago-clean` deletes nothing in a resource package.

## Project Setup

The rest of the setup is done by the packages that own it:

```bash
php vendor/bin/migration db:migrate --packages
php vendor/bin/create-permission --packages
```

### Migration Priority

Order of the package migrations (lower first, default `100`), read by `drago-ex/migration`:

```json
"extra": {
    "drago-tools": {
        "migration-priority": 10
    }
}
```
