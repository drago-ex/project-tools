# Drago Project Installer

Tools for automatic installation and cleanup of project resources from Composer packages.

## Features
- **`drago-install`**: Automatically copies or replaces files/directories from vendor packages to your project root.
- **`drago-clean`**: Cleans up redundant resource folders in `vendor/drago-ex` to prevent class duplication or namespace collisions.

## Installation
Add the package to your project:
```bash
composer require drago-ex/project-installer
```
For component development, add it to `require-dev`:
```bash
composer require --dev drago-ex/project-installer
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
3. It respects the `allow-library-install` flag in your root `composer.json` (defaults to `false` for libraries).
4. `drago-clean` specifically targets `vendor/drago-ex` and removes `resources` directories to keep your vendor clean after files have been mirrored to your project.
