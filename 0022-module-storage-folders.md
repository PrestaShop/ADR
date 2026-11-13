# 22. Module storage folders

Date: 2026-11-13

## Status

In discussion

Supersedes the first draft of this ADR ([PR #34](https://github.com/PrestaShop/ADR/pull/34), 2023-05-30) and takes over the review comments left on it.

Related to [0026 - Pluggable file storage](https://github.com/PrestaShop/ADR/pull/46): this ADR defines *where* a module's files live and who owns each location; 0026 defines *through which abstraction* they are read and written. Each location below is meant to become a named storage of 0026 once it is accepted, without changing the rules of this ADR.

## Context

The core gives modules no sanctioned place to write files. The only guidance is a paragraph of the [module good practices](https://devdocs.prestashop-project.org/9/modules/creation/good-practices/) recommending `var/cache/<env>/modules/<module>/` and `var/modules/<module>/`, which is not backed by any ADR, any constant or any API. As a result, every agency and module vendor follows its own convention, as a community discussion on Friends of Presta showed again in August 2026:

- images uploaded by the merchant stored in `modules/<module>/views/img/`, in a subfolder of `img/`, or in `upload/`;
- logs written next to the code, in `var/modules/<module>/`, or in `var/logs/`;
- cache and generated files mixed with the source code.

What exists today:

| Location | Web access | Owner |
|---|---|---|
| `modules/<module>/` | Allowed, except `.php`, `.tpl`, `.twig`, `.yml`, `.json`, `.log`, `.txt`, `config*.xml` (`modules/.htaccess`) | Module code |
| `var/` | Denied (`var/.htaccess`) | Core |
| `var/modules/` | Denied (inherited) | Created in 2021 (`f6b3abd`), used by nothing, no constant |
| `var/cache/<env>/` | Denied | Core, wiped on cache clear |
| `var/logs/` | Denied | Core (`%kernel.logs_dir%`) |
| `img/` | Media extensions only (`img/.htaccess`) | Core |
| `upload/`, `download/` | Denied / hashed names only | Core (customizations, virtual products) |

Writing at runtime into `modules/<module>/` causes concrete problems:

- **Scaling and containers.** With several web nodes or pods, `modules/` ships inside the image or the release and should be read-only. Files written there by one node are missing on the others, and are lost at the next deployment. Operators have to mount persistent volumes on locations they cannot predict.
- **Backups and environment synchronisation.** Data, cache and logs mixed with code cannot be backed up, synchronised or excluded as a whole.
- **Upgrades and uninstall.** Replacing a module's folder on upgrade, or deleting it, silently destroys the merchant's data.
- **Security.** A writable code directory is the first thing an attacker uploading a file looks for.
- **Logs never cleaned.** Hosting cron jobs and log rotation target `var/logs/`; logs written elsewhere grow forever and are hard to find.

## Decision

### 1. Five locations, one purpose each

| Purpose | Location | Web access | Survives | Lifecycle |
|---|---|---|---|---|
| Code and assets shipped with the module | `modules/<module>/` | As today | Until the module is removed or upgraded | **Read-only at runtime** |
| Cache and temporary files | `var/cache/<env>/modules/<module>/` | Denied | Nothing: wiped on every cache clear | Must be rebuildable at any time |
| Logs | `var/logs/` | Denied | Rotation | One file per module and environment |
| Private data | `var/modules/<module>/` | Denied | Cache clear, upgrade, uninstall | Owned by the module |
| Public data | `img/modules/<module>/` | Media extensions only | Cache clear, upgrade, uninstall | Owned by the module |

`<module>` is always the technical name of the module, so two modules can never collide and an operator can tell at a glance which module owns a file.

### 2. Rules per location

**`modules/<module>/` is code.** A module MUST NOT create, modify or delete files in its own folder, or in any other module's folder, at runtime. Images, CSS, JS and templates distributed with the module stay there. Only the module manager writes there, when installing, upgrading or removing the module.

**Cache** is for anything the module can rebuild: compiled templates, generated CSS, API responses, temporary files of an import. The core may delete it at any time; the module must not rely on it being present.

**Logs** go to `var/logs/`, through a Monolog logger declared in the module's services, named `<module>_<env>.log` (rotating handler, `%kernel.logs_dir%`). A module MUST NOT write logs anywhere else. Example:

```yaml
services:
  my_module.logger:
    class: Monolog\Logger
    arguments: ['my_module']
    calls:
      - pushHandler: ['@my_module.logger.handler']

  my_module.logger.handler:
    class: Monolog\Handler\RotatingFileHandler
    arguments:
      - '%kernel.logs_dir%/my_module_%kernel.environment%.log'
      - 10
```

**Private data** is everything that must persist but must never be served directly: configuration files, keys, exports, invoices, files uploaded by customers. When such a file has to be downloaded, the module exposes it through a controller that checks the permissions of the requester and streams the file. The web server never serves `var/` directly.

**Public data** is media produced after installation that the front office displays as-is: images uploaded by the merchant (logos, banners, contact pictures), generated thumbnails. It goes to `img/modules/<module>/`, because `img/` is already served by the web server, already restricted to media types, and already part of what every hosting setup persists and shares between nodes.

**Configuration** is not one location but three, depending on who writes it and when:

| Kind of configuration | Location | Reason |
|---|---|---|
| Shipped with the module (`config/services.yml`, `config/routes.yml`, ...) | `modules/<module>/config/` | It is code: versioned, read-only, loaded by the kernel |
| Settings edited by the merchant (options, API keys entered in the back office) | `ps_configuration` (`Configuration::updateValue()`) or the module's own tables | Already the standard, and scoped per shop in multistore; no file involved |
| Files written at runtime (generated JSON/XML, certificates, private keys) | `var/modules/<module>/` | Private data that must persist and never be served |

Secrets that depend on the environment (production and staging credentials) are better read from environment variables than stored in the database, which ends up in dumps and backups.

`upload/` and `download/` remain reserved to the core (product customizations and virtual products); modules MUST NOT write there.

### 3. The core provides the paths, not the module

Modules must not build these paths themselves from `_PS_ROOT_DIR_`, so that the locations can later move or be backed by another storage (0026) without touching the modules.

- The core exposes a service, `PrestaShop\PrestaShop\Core\Module\ModuleStorageLocatorInterface`, returning for a module name its cache directory, private data directory, public data directory and the public URL of the latter. Directories are created on first access.
- The equivalent container parameters (`prestashop.module_cache_dir`, `prestashop.module_data_dir`, `prestashop.module_public_dir`) are available for modules declaring their own services, with the module name appended by the module.
- Legacy modules reach the same service through `Module::get()` / the container; no new method is added on the `Module` class, since a module already declaring a method with the same name would break (BC promise, ADR 0017).

### 4. Uninstall and removal

- On uninstall, the core wipes the module's cache directory. Private and public data are kept, so that reinstalling the module, or installing a newer version, does not lose the merchant's data, consistently with database tables a module may choose to keep.
- A module that wants to delete its data on uninstall does it explicitly in `uninstall()`, through the same service.

### 5. Adoption

- The rules apply as soon as this ADR is accepted, as a recommendation for all module developers and as a requirement for native modules, which are migrated one by one.
- A module moving existing data out of its own folder does it in an upgrade script, so that merchants do not lose files.
- The devdocs good-practices page is rewritten from this ADR, and the PrestaShop Addons validator can flag runtime writes into `modules/`.
- From the next major version, the project officially supports a read-only `modules/` directory; writing there becomes unsupported.

## Options considered and rejected

- **A single `var/modules/<module>/` with `public/`, `private/` and `cache/` subfolders** (suggested on PR #34). One location per module is easier to reason about, but `var/` is denied to the web server in every supported configuration (Apache `.htaccess`, the documented nginx setup). Serving `var/modules/*/public/` means punching a hole in that rule for every host, and a misconfigured host would expose private data next to it. Keeping public files in `img/` keeps the boundary "denied/served" at the top-level directory, where operators already handle it.
- **A non-guessable public path** (random segment per shop, suggested on PR #34 to avoid revealing installed modules). The module name is already visible in the front office through its assets (`modules/<module>/views/...`) and hook output; obfuscating only the data folder adds a path to store and migrate without hiding anything.
- **`img/module/<module>/`** (singular). `img/module/` already exists and holds core files (default module logo); a separate `img/modules/` avoids mixing core files and module data.
- **Leaving it to the devdocs.** The current paragraph has existed for years without being followed, precisely because it is not a decision and comes with no API.

## Consequences

- Module developers get one answer to "where do I write this file?", and agencies stop debating it per project.
- An operator knows which directories to persist and share between nodes: `img/`, `upload/`, `download/`, `var/modules/`. `var/cache/` stays local to each node, `modules/` can ship read-only in the image.
- Backups and environment synchronisation can exclude cache and logs as a whole.
- Logs of every module are found and rotated in one place.
- Upgrading or removing a module no longer destroys merchant data.
- Existing modules writing into their own folder keep working but are non-compliant; native modules have to be migrated, each with an upgrade script.
- Generated CSS/JS that must be public and depends on configuration has no public location in this ADR (see open questions).

## Open questions

1. **Public CSS/JS generated at runtime** (for example a stylesheet built from module settings). `img/` only serves media. Options: serve it through a module front controller with cache headers, let the core asset manager (CCC) publish it from the module's cache directory, or allow `css`/`js` in `img/modules/` only.
2. **Shop scope.** Should the locator offer a per-shop subfolder (`<module>/<id_shop>/`) for multistore setups, or is it left to the module?
3. **Name of the service and parameters**, to align with the storage names of ADR 0026 (`modules_cache`, `modules_private`, `modules_public`) if it is accepted first.
