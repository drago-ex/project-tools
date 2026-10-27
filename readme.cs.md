# Drago Project Tools

Nástroje pro automatickou instalaci a úklid zdrojů projektu z balíčků Composeru.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/drago-ex/project-tools/blob/main/license)
[![PHP version](https://badge.fury.io/ph/drago-ex%2Fproject-tools.svg)](https://badge.fury.io/ph/drago-ex%2Fproject-tools)
[![Coding Style](https://github.com/drago-ex/project-tools/actions/workflows/coding-style.yml/badge.svg)](https://github.com/drago-ex/project-tools/actions/workflows/coding-style.yml)

## Požadavky

- PHP >= 8.3
- Composer 2

## Instalace

Přidejte balíček do projektu:
```bash
composer require drago-ex/project-tools
```
Pro vývoj komponent ho přidejte do `require-dev`:
```bash
composer require --dev drago-ex/project-tools
```

## Myšlenka balíčků Drago

Resource balíčky Drago (`project-auth`, `project-front`, `project-backend` a další) nejsou knihovny, které voláte
ze svého kódu. Každý obsahuje jen základní části jedné funkce a `drago-install` je zkopíruje do projektu,
kde se z nich stane váš vlastní kód k úpravám. Pozdější `composer update` nikdy nepřepíše soubory, které už v projektu
existují, takže opravu vydanou v balíčku je potřeba převzít ručně; `drago-diff` ukáže, kterých souborů se týká.

## Funkce

- **`drago-install`**: Automaticky kopíruje nebo nahrazuje soubory a adresáře z balíčků ve vendor do kořene projektu.
- **`drago-clean`**: Uklízí nadbytečné adresáře se zdroji v nastavených složkách vendor, aby se třídy neduplikovaly a nekolidovaly jmenné prostory.
- **`drago-commands`**: Spustí skripty, které nainstalované balíčky uvádějí v `extra.drago-tools.commands`.
- **`drago-report`**: Zobrazí reporty v `.drago-project/`: co `drago-install` zkopíroval, nahradil nebo přeskočil,
  které migrace a které příkazy balíčků proběhly.
- **`drago-diff`**: Zobrazí soubory projektu, které jejich balíček od instalace změnil.

## Konfigurace

V `composer.json` svého balíčku určete, co se má instalovat:

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

### Globální volby

Důvěryhodné vendory Composeru nastavte v **kořenovém** `composer.json`:

```json
"extra": {
    "drago-tools": {
        "vendor-names": ["drago-ex", "acme"],
        "allow-library-install": true
    }
}
```

`vendor-names` omezuje, které balíčky `drago-install` a `drago-clean` prohledávají.
Výchozí hodnota je `["drago-ex"]`.

Pokud nastavená složka vendoru ve `vendor/` neexistuje, nástroje vypíšou varování:

```text
WARN Vendor name not found: vendor-name
```

*Poznámka: Z bezpečnostních důvodů mohou ve výchozím stavu instalovat zdroje jen balíčky typu `drago-tools-resource`. Pro zrcadlení běžných knihoven z nastavených vendorů použijte `allow-library-install`.*

### Sekce:

- **`copy`**: Kopíruje soubory, jen pokud v cíli ještě neexistují. Bezpečné pro první nastavení.
- **`replace`**: Cílové soubory vždy přepíše. Hodí se pro aktualizace jádra nebo sdílené assety.

Zdrojové a cílové cesty musí být relativní cesty uvnitř balíčku a projektu. Absolutní cesty a cesty obsahující `..` se odmítnou.

### Přeskočení balíčků

Pro přeskočení konkrétního balíčku při instalaci ho přidejte do mapy `packages` v **kořenovém** `composer.json`:

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

S `skip` nastaveným na `true` se balíček úplně ignoruje - neproběhne pro něj `copy` ani `replace`.

### Přeskočení jen copy nebo replace

Můžete přeskočit i jen jednu sekci instalace balíčku:

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

- **`skip-copy`**: Přeskočí jen sekci `copy` balíčku.
- **`skip-replace`**: Přeskočí jen sekci `replace` balíčku.
- **`skip`**: Přeskočí celý balíček a má přednost před přeskočením sekcí.

Hodí se to pro presety, jejichž soubory se mají přepsat jednou při prvním sestavení projektu, ale při dalších aktualizacích Composeru už ne.

### Instalace jen jednou

Balíčky s pravidly `replace` se mohou označit jako replace-once:

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

Po úspěšném `replace` zapíše `drago-install` do kořenového `composer.json` projektu:

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

Další běh balíček ponechá nainstalovaný, ale jeho sekci `replace` přeskočí. Bezpečnější sekce `copy` může dál běžet, pokud není zapnuté i `skip-copy` nebo `skip`.

`replace-once` se v režimu `--dev` neuplatňuje.

### Priorita nahrazování

Instalace probíhá ve dvou fázích:

1. Nejdřív se zpracují sekce `copy` všech balíčků.
2. Potom se zpracují sekce `replace` balíčků seřazené podle priority.

Priorita se nastavuje v `composer.json` balíčku:

```json
"extra": {
    "drago-tools": {
        "install": {
            "replace-priority": 200
        }
    }
}
```

Nižší priorita běží dřív, vyšší později. Když tedy stejný soubor nahrazuje více balíčků, vyhraje vyšší priorita.

Výchozí priorita je `100`.

Příklad:

```text
acme/backend       priority 100
acme/backend-ui    priority 200
acme/overlay       priority 300
```

V tomto příkladu může `acme/backend-ui` přepsat soubory `acme/backend` a `acme/overlay` oba předchozí.

## Použití

### Automatizace

Pro automatizaci přidejte do `composer.json` tyto skripty:

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

### Ruční spuštění

Příkazy můžete spustit i ručně v terminálu:

```bash
vendor/bin/drago-install
vendor/bin/drago-clean
vendor/bin/drago-report
vendor/bin/drago-diff
vendor/bin/drago-commands
```

### Volby

- `--verbose` nebo `-v`: Zobrazí podrobný průběh instalace po jednotlivých souborech.
- `--dev`: Pro vývoj resource balíčku, viz níže.

### Report

`drago-install` zapíše, co zkopíroval, nahradil nebo přeskočil, do `.drago-project/install.json`. Migrace
(`drago-ex/migration`) a `drago-commands` zapisují své reporty do stejné složky.
Zobrazíte je takto:

```bash
vendor/bin/drago-report
vendor/bin/drago-report --tool=install
vendor/bin/drago-report --package=translator
vendor/bin/drago-report --action=skipped
```

Složku `.drago-project/` commitněte, kromě `migration.json`, který se liší podle databáze.

### Aktualizace balíčků

`drago-install` zaznamenává do `.drago-project/installed.json`, která verze balíčku nainstalovala který soubor. Po
`composer update` vypíše `drago-diff` soubory, které jejich balíček změnil, a zda jste je upravili:

```bash
vendor/bin/drago-diff                          # files changed in their packages
vendor/bin/drago-diff app/Presentation/Sign/SignPresenter.php            # what the package changed
vendor/bin/drago-diff app/Presentation/Sign/SignPresenter.php --project  # your file against the new version
vendor/bin/drago-diff --accept app/Presentation/Sign/SignPresenter.php   # after taking the changes over
```

Vypíše také nové soubory balíčků s `replace-once`, které `drago-install` po první instalaci už nekopíruje.
Nic se nepřepisuje; změny převezměte ručně. Verze balíčků se stahují přes `composer archive`.
Větev (`dev-main`) se zaznamená i s commitem, takže se vypíše i novější commit; jeho starší commit nejde stáhnout,
proto pak `drago-diff <soubor>` porovná váš soubor s novou verzí.

### Úklid

`drago-clean` odstraní z `vendor/` adresáře `resources/` nainstalovaných resource balíčků, aby stejné třídy
nebyly zároveň ve `vendor/` i v projektu. Balíčky připojené symlinkem (repozitáře Composeru typu `path`) zůstanou beze změny.

## Vývoj resource balíčku

V resource balíčku spusťte `drago-install --dev`. Zkopíruje zdroje jeho závislostí do `.dev-resource/`,
aby PHPStan a testy mohly používat jejich třídy, a zkontroluje vlastní pravidla `copy` a `replace` balíčku: report
v `.dev-resource/.drago-project/install.json` ukáže, kam se každý vlastní soubor v projektu dostane, chybějící zdroj
a soubor, který instaluje i některá závislost.

Přidejte `/.dev-resource/` do `.gitignore`, `scanDirectories: [.dev-resource]` do `phpstan.neon` a v `tests/bootstrap.php`
načítejte třídy z `.dev-resource/app` až po `resources/app`. `drago-clean` v resource balíčku nic nemaže.

## Nastavení projektu

Zbytek nastavení obstarávají balíčky, kterým patří:

```bash
php vendor/bin/migration db:migrate --packages
php vendor/bin/drago-commands
```

### Příkazy balíčků

Balíček uvádí skripty, které `drago-commands` spustí z kořene projektu, například generátor jeho
providerů oprávnění. Cesty jsou relativní k balíčku, názvy jsou volitelné:

```json
"extra": {
    "drago-tools": {
        "commands": ["bin/create-front-permission"]
    }
}
```

`drago-commands` spustí skripty všech nainstalovaných balíčků v pořadí podle názvu balíčku a zapíše
`.drago-project/commands.json`.

### Priorita migrací

Pořadí migrací balíčků (nižší dřív, výchozí `100`), které čte `drago-ex/migration`:

```json
"extra": {
    "drago-tools": {
        "migration-priority": 10
    }
}
```
