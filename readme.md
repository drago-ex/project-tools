# Drago Project Tools

Tools for automatic installation and cleanup of project resources from Composer packages.

## Features
- **`drago-install`**: Automatically copies or replaces files/directories from vendor packages to your project root.
- **`drago-clean`**: Cleans up redundant resource folders in `vendor/drago-ex` to prevent class duplication or namespace collisions.

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
"extra": {
    "drago-project": {
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
To allow installation from packages with the default type `library`, enable this flag in your **root** `composer.json`:

```json
"extra": {
    "drago-project": {
        "allow-library-install": true
    }
}
```
*Note: For security, only packages of type `drago-project-resource` are allowed to install resources by default. Use this flag to enable mirroring for standard libraries.*

### Sections:
- **`copy`**: Copies files only if they do not already exist in the destination. Safe for initial setup.
- **`replace`**: Always overwrites the destination files. Useful for core updates or shared assets.

### Skipping Packages
To skip a specific package during installation, add it to the `packages` map in your **root** `composer.json`:

```json
"extra": {
    "drago-project": {
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
    "drago-project": {
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
    "drago-project": {
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
    "drago-project": {
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
    "drago-project": {
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
```

### Options
- `--verbose` or `-v`: Show detailed file-by-file progress during installation.
- `--dev`: Switches the destination directory to `resources/` in the current working directory. Useful for testing installation logic during package development.

## How it works
1. `drago-install` scans all installed packages for the `extra.drago-project.install` configuration.
2. It resolves relative paths against the package root and the current working directory.
3. It skips packages listed under `extra.drago-project.packages` in the root `composer.json` with `"skip": true`.
4. It runs all `copy` sections first.
5. It runs `replace` sections afterwards, sorted by `install.replace-priority`.
6. It can skip only `copy` or `replace` sections with `skip-copy` and `skip-replace`.
7. It respects the `allow-library-install` flag in your root `composer.json` (defaults to `false` for libraries).
8. Packages marked with `install.replace-once` automatically add `skip-replace: true` to the root `composer.json` after a successful replace run.
9. `drago-clean` specifically targets `vendor/drago-ex` and removes `resources` directories to keep your vendor clean after files have been mirrored to your project.
