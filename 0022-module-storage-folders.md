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
| Public data | `files/modules/<module>/` | Served as static files, no script execution | Cache clear, upgrade, uninstall | Owned by the module |

`<module>` is always the technical name of the module, so two modules can never collide and an operator can tell at a glance which module owns a file.

### 2. Rules per location

**`modules/<module>/` is code.** A module MUST NOT create, modify or delete files in its own folder, or in any other module's folder, at runtime. Images, CSS, JS and templates distributed with the module stay there. Only the module manager writes there, when installing, upgrading or removing the module.

**Cache** is for anything the module can rebuild and that is never served directly: compiled templates, API responses, temporary files of an import. The core may delete it at any time; the module must not rely on it being present.

**Logs** go to `var/logs/`, through a logger provided by the core. `PrestaShop\PrestaShop\Core\Module\ModuleLoggerFactoryInterface::create(string $moduleName, ?string $channel = null)` returns a PSR-3 logger writing to `<module>_<env>.log`, or `<module>_<channel>_<env>.log` for a module with several channels, so that `<module>_*` always matches all its files. The files go next to the core log (`PS_LOG_OUTPUT`) with the same rotation (`PS_LOG_MAX_FILES`), and the logger works the same in the back office, the front office and the CLI. A module MUST NOT write logs anywhere else, and does not declare its own Monolog handler: `%kernel.logs_dir%` is not defined in the legacy front-office container, where most modules log, and a handler of its own would ignore `PS_LOG_OUTPUT` and `PS_LOG_MAX_FILES`. Example:

```yaml
services:
  my_module.logger:
    class: Psr\Log\LoggerInterface
    factory: ['@PrestaShop\PrestaShop\Core\Module\ModuleLoggerFactoryInterface', 'create']
    arguments: ['my_module']
```

**Private data** is everything that must persist but must never be served directly: configuration files, keys, exports, invoices, files uploaded by customers. When such a file has to be downloaded, the module exposes it through a controller that checks the permissions of the requester and streams the file. The web server never serves `var/` directly.

**Public data** is every file produced after installation that is meant to be fetched as-is by anyone holding its URL: images uploaded by the merchant (logos, banners, contact pictures), generated thumbnails, product feeds (Google Merchant Center XML), public PDF documents, CSS or JS generated from the module settings. It goes to `files/modules/<module>/`, a new top-level directory served directly by the web server, without going through PHP.

Since this directory is writable by PHP and served by the web server, the core enforces two rules on it:

- **Static types only.** The core ships a `files/.htaccess` built like `img/.htaccess`: everything is denied except a list of static types (the media allowed in `img/`, plus `xml`, `csv`, `json`, `txt`, `pdf`, `xlsx`, `css`, `js`, `woff`, `woff2`). An allow list cannot miss an executable extension the way a deny list does, and it also refuses an uploaded `.html` or `.htm`. Every response carries `X-Content-Type-Options: nosniff` and `Content-Security-Policy: sandbox`, so that an SVG opened directly cannot run its scripts. The documented nginx configuration gets the equivalent rules. A module needing another type asks for it in the core; it does not ship its own `.htaccess`.
- **Public means public.** A file is placed there only if it can be read by anyone who knows or guesses its URL. Files holding personal or commercial data (order or customer exports, invoices, private feeds) are private data, whatever their format, and are served through a controller that checks permissions.

**Configuration** is not one location but three, depending on who writes it and when:

| Kind of configuration | Location | Reason |
|---|---|---|
| Shipped with the module (`config/services.yml`, `config/routes.yml`, ...) | `modules/<module>/config/` | It is code: versioned, read-only, loaded by the kernel |
| Settings edited by the merchant (options, API keys entered in the back office) | `ps_configuration` (`Configuration::updateValue()`) or the module's own tables | Already the standard, and scoped per shop in multistore; no file involved |
| Files written at runtime (generated JSON/XML, certificates, private keys) | `var/modules/<module>/` | Private data that must persist and never be served |

`upload/` and `download/` remain reserved to the core (product customizations and virtual products); modules MUST NOT write there.

### 3. The core provides the storage, not the paths

Modules must not build these paths themselves from `_PS_ROOT_DIR_`, nor write their private and public data with `file_put_contents()` and the other PHP file functions: the day a shop keeps these files elsewhere (a shared disk, a bucket, through ADR 0026), every module would have to be rewritten.

- **Private and public data are written through the core.** The core exposes a service, `PrestaShop\PrestaShop\Core\Module\ModuleStorageProviderInterface`, returning for a module name its private storage and its public storage. Each storage offers `write()`, `writeStream()`, `read()`, `readStream()`, `exists()`, `delete()` and `listContents()`, on paths relative to the module's location; the public storage adds `publicUrl()`. They are implemented on the local disk (`var/modules/<module>/`, `files/modules/<module>/`) and are backed by the named storages of 0026 once it is accepted, with no change in the modules.
- **`localPath()` for libraries that need a real file.** Some private files can only be handed to a library as a path, such as client certificates and keys for cURL or OpenSSL. The private storage offers `localPath()`, returning the file itself on the local disk, or a copy in the module's cache directory when the storage is remote. It is the only method that does not carry over to another storage, and is documented as such: a module uses it only when the library leaves no other choice.
- **The cache keeps a path.** Smarty, GD or Imagick need a real directory, and the cache is local to each node by design. The provider returns the module's cache directory, `var/cache/<env>/modules/<module>/`, created on first access; the container parameter `prestashop.module_cache_dir` is available for modules declaring their own services, with the module name appended by the module.
- **Shop scope.** Each method of the provider takes an optional shop ID. Without it, it returns the storage of the module's root (`<module>/`), for data shared by all shops; with it, a subfolder `<module>/<id_shop>/`, for data specific to one shop (a feed per shop, a logo per shop). The same code works on a single-shop installation, which simply has one shop ID.
- The provider also exposes `removeModuleData(string $moduleName)`, deleting the module's private and public data.
- Legacy modules reach the same service through `Module::get()` / the container; no new method is added on the `Module` class, since a module already declaring a method with the same name would break (BC promise, ADR 0017).

### 4. Uninstall and removal

- On uninstall, the core wipes the module's cache directory. Private and public data are kept, so that reinstalling the module, or installing a newer version, does not lose the merchant's data, consistently with database tables a module may choose to keep.
- A module that wants to delete its data on uninstall does it explicitly in `uninstall()`, by calling `removeModuleData()`. Keeping is the default because a forgotten deletion can be done later, while a deleted logo or feed history cannot be restored.

### 5. Adoption

- The rules apply as soon as this ADR is accepted, as a recommendation for all module developers and as a requirement for native modules, which are migrated one by one.
- A module moving existing data out of its own folder does it in an upgrade script, so that merchants do not lose files.
- The devdocs good-practices page is rewritten from this ADR, and the PrestaShop Addons validator can flag runtime writes into `modules/`.
- From the next major version, the project officially supports a read-only `modules/` directory; writing there becomes unsupported.

## Options considered and rejected

- **A single `var/modules/<module>/` with `public/`, `private/` and `cache/` subfolders** (suggested on PR #34). One location per module is easier to reason about, but `var/` is denied to the web server in every supported configuration (Apache `.htaccess`, the documented nginx setup). Serving `var/modules/*/public/` means punching a hole in that rule for every host, and a misconfigured host would expose private data next to it. A separate top-level `files/` keeps the boundary "denied/served" at the top-level directory, where operators already handle it.
- **`img/modules/<module>/`** (first version of this proposal). `img/` is already served, persisted and restricted to safe types, but only to media: XML feeds, CSV and generated CSS/JS cannot be served from there, and widening `img/.htaccess` would weaken it for the core images too. Keeping media in `img/` and everything else elsewhere would bring back two public locations per module.
- **A non-guessable public path** (random segment per shop, suggested on PR #34 to avoid revealing installed modules). The module name is already visible in the front office through its assets (`modules/<module>/views/...`) and hook output; obfuscating only the data folder adds a path to store and migrate without hiding anything.
- **`public/modules/<module>/`** (suggested during the review). Explicit, but `public/` is the web root of a Symfony application: if PrestaShop moves to a dedicated web root one day, the two meanings would collide. `files/` carries no such meaning.
- **A locator returning the paths of the private and public directories** (previous version of this proposal). It keeps modules on the PHP file functions, so moving these files to another storage would mean rewriting every module again. Only the cache, which needs a real directory, keeps a path.
- **A Monolog recipe copied in each module** (previous version of this proposal). It relies on `%kernel.logs_dir%`, missing in the legacy front-office container, and ignores `PS_LOG_OUTPUT` and `PS_LOG_MAX_FILES`.
- **A deny list of executable extensions in `files/.htaccess`** (previous version of this proposal). It always ends up missing one, and lets an uploaded `.html` through.
- **A subfolder per module in `var/logs/`** (`var/logs/<module>/<module>_<env>.log`). It keeps a crowded `var/logs/` tidy, but hosting cleanup jobs, `logrotate` configurations and log collectors (Kubernetes sidecars, Filebeat, Promtail) commonly target `var/logs/*.log` without recursing, so logs in a subfolder risk never being cleaned or collected. The module prefix already groups a module's files, and the rotating handler caps their number.
- **A flag declaring whether the module's files are deleted on uninstall** (suggested during the review). It would duplicate `uninstall()`, where the module already decides what happens to its tables and configuration; a single place for that decision is easier to follow.
- **A folder per shop group.** Group membership can change, which would leave a shop's files in the folder of its former group.
- **Leaving it to the devdocs.** The current paragraph has existed for years without being followed, precisely because it is not a decision and comes with no API.

## Consequences

- Module developers get one answer to "where do I write this file?", and agencies stop debating it per project.
- An operator knows which directories to persist and share between nodes: `img/`, `upload/`, `download/`, `files/`, `var/modules/`. `var/cache/` stays local to each node, `modules/` can ship read-only in the image.
- Backups and environment synchronisation can exclude cache and logs as a whole.
- Logs of every module are found and rotated in one place.
- Upgrading or removing a module no longer destroys merchant data.
- Existing modules writing into their own folder keep working but are non-compliant; native modules have to be migrated, each with an upgrade script.
- Modules write their data through the storage service instead of the PHP file functions. Once 0026 lands, moving private or public data to another storage needs no change in the modules, except where they use `localPath()`.
- The core itself still writes into `modules/<module>/`: `Module::_generateConfigXml()` creates the module's `config.xml` when the back office lists the modules. It checks `is_writable()` first, so a read-only `modules/` breaks nothing, but this generated cache has to move to `var/cache/` before a read-only `modules/` is officially supported.
- A new top-level directory, `files/`, has to be created by the installer and by the upgrade tool, shipped with its `.htaccess`, added to the documented nginx configuration, and mounted as a persistent volume by hosting setups.
- Feeds, exports and generated assets that modules serve today through a PHP script can be served as static files, which is faster and cacheable by a CDN.

## Open questions

1. **Name of the services and of the storage interface**, to align with ADR 0026 if it is accepted first: its storage names (`modules_cache`, `modules_private`, `modules_public`), and its storage interface, which the module storages could expose directly instead of a dedicated one.
