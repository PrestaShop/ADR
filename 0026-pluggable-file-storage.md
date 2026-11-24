# 26. Pluggable file storage

Date: 2026-11-24

## Status

In discussion

Replaces [PR #46](https://github.com/PrestaShop/ADR/pull/46) by @adrianllopis (2026-10-21), no longer followed up, itself built on the proof of concept of @matthieu-rolland ([discussion #41141](https://github.com/PrestaShop/PrestaShop/discussions/41141), [PrestaShop#41140](https://github.com/PrestaShop/PrestaShop/pull/41140)). This version is reworked after the acceptance of [0027 - Module storage folders](0027-module-storage-folders.md).

0027 defines *where* each kind of file lives and who owns it; this ADR defines *through which abstraction* the core and modules read and write those files, so that the location is no longer necessarily a local disk. It also settles the open question left by 0027 on the module storages.

## Context

PrestaShop accesses files through native PHP calls spread over the code base, resolved against global path constants (`_PS_IMG_DIR_`, `_PS_UPLOAD_DIR_`, `_PS_DOWNLOAD_DIR_`, ...). On `develop` (November 2026), `classes/`, `controllers/` and `src/` hold roughly 800 such calls, including 384 `file_exists`, 108 `unlink`, 79 `file_get_contents`, 69 `scandir`, 51 `copy`, 42 `file_put_contents`, 25 `mkdir` and 19 `move_uploaded_file`.

Consequences of the current situation:

- **A shop cannot store its files anywhere but the local disk.** Running several application instances requires a shared file system (NFS, `s3fs`, ...) at the infrastructure level, with the performance and consistency problems that come with it.
- **There is no single place to plug anything else**: no caching of existence checks, no CDN, no instrumentation, no read-only mode.
- **Code that touches files is hard to test**: direct calls to `file_put_contents` cannot be mocked, so tests either hit the disk or the code stays untested.
- **0027 depends on it.** It gives modules a private and a public storage through `ModuleStorageProviderInterface`, implemented on the local disk until the storages of this ADR exist.

The idea was raised in discussion #41141, prototyped in PrestaShop#41140 and implemented in [PrestaShop#43255](https://github.com/PrestaShop/PrestaShop/pull/43255), which waits for an ADR. When 0027 was accepted, the Committers Monthly of November 2026 agreed that its module storages are to be backed by the named storages of this ADR.

## Decision

### 1. The core exposes its own storage contract

A small interface in `PrestaShop\PrestaShop\Core\Storage`:

```php
interface StorageInterface
{
    public function fileExists(string $path): bool;
    public function directoryExists(string $path): bool;
    public function read(string $path): string;
    public function readStream(string $path);           // resource
    public function write(string $path, string $contents): void;
    public function writeStream(string $path, $contents): void;
    public function delete(string $path): void;
    public function createDirectory(string $path): void;
    public function deleteDirectory(string $path): void;
    public function copy(string $source, string $destination): void;
    public function move(string $source, string $destination): void;
    public function listFiles(string $path, bool $recursive = false): array;
    public function fileSize(string $path): int;
    public function mimeType(string $path): string;
    public function lastModified(string $path): int;
    public function publicUrl(string $path): string;
}
```

Rules:

- **Paths are relative to the storage root** and always use `/`, whatever the backend is. A path leaving the root (`../`) is refused.
- **No type of the underlying library appears in the signatures**, and the library's exceptions are wrapped in exceptions of `Core\Storage\Exception`. The implementation can therefore be replaced without a BC break for the callers.
- **Streams are part of the contract**, so that large files (virtual products, exports) are never fully loaded in memory.
- **The interface is used by modules, not implemented by them.** A shop or a module reaching another backend provides a Flysystem adapter (section 4), never its own `StorageInterface`. Implementing it outside the core is not supported, so the core may add a method to it in a minor version; the interface says so in its docblock. Without this rule, the contract could never grow without breaking the promise of 0017.

Behavior, identical on every backend:

| Situation | Behavior |
|---|---|
| `read()`, `readStream()`, `fileSize()`, `mimeType()`, `lastModified()`, `copy()` or `move()` on a missing file | `StorageFileNotFoundException` |
| `write()` or `writeStream()` on an existing file | The file is replaced; missing parent directories are created |
| `copy()` or `move()` to an existing destination | The destination is replaced |
| `delete()` on a missing file, `deleteDirectory()` on a missing directory | Nothing: deleting is idempotent |
| `createDirectory()` on a backend without real directories (object storage) | Nothing, and `directoryExists()` is true as soon as the directory holds a file |
| `listFiles()` | The paths of the files only, relative to the storage root, in no guaranteed order |
| `publicUrl()` on a private storage | `StorageException` (see below) |
| Any other failure (permissions, network, quota) | `StorageOperationException` |

Both exceptions extend `StorageException`, so a caller can catch every storage failure at once.

**`publicUrl()` only works on a public storage.** It returns the URL the file is served at:

- On the local disk, the core builds it from the shop in `ShopContext` (ADR 0024), with its domain, its physical URI and the media servers (`PS_MEDIA_SERVER_*`), as the image URLs are built today; never from a URL fixed in the configuration, so multistore keeps working. Without a single shop in context (CLI, back office in "all shops" context), it uses the default shop; code generating URLs for a given shop, such as a multistore feed, sets the shop context first, as it already has to with `Link`.
- On another backend, it is the public URL of the bucket or of the CDN in front of it, from the adapter's configuration.
- On a private storage, it throws: such a file is served by a controller checking permissions (0027, section 2).

### 2. Storages are named, one per location

Rather than a single storage for the whole project, the core declares one storage per location:

| Name | Root on the local disk | Public | Used by |
|---|---|---|---|
| `images` | `img/` (`_PS_IMG_DIR_`) | Yes | Core: images uploaded or generated for entities |
| `uploads` | `upload/` (`_PS_UPLOAD_DIR_`) | No | Core: product customizations |
| `downloads` | `download/` (`_PS_DOWNLOAD_DIR_`) | No | Core: virtual products, attachments |
| `modules_private` | `var/modules/` | No | Modules, through `ModuleStorageProviderInterface` |
| `modules_public` | `files/modules/` | Yes | Modules, through `ModuleStorageProviderInterface` |

A shop can then move its product images to object storage while keeping everything else local, which is the realistic migration path.

**`images` only covers the data of `img/`**, the folders written at runtime: entity images and their thumbnails (`p/`, `c/`, `m/`, `su/`, `st/`, `e/`, `co/`, `l/`, ...) and the shop logos. `img/` also holds two things the storage does not:

- **Assets shipped with the release** (`img/admin/`, `img/jquery-ui/`, `img/flags/`, `loader.gif`, ...). They are referenced by direct URL, ship with the code like `modules/` and `themes/`, and stay on the local disk of each node.
- **Temporary files** (`img/tmp/`), which are a cache: the code using them keeps a local path, and they move to `var/cache/` when their area is migrated.

What has no storage, on purpose:

- **Code and what the web server or the autoloader read directly**: `modules/`, `themes/`, `translations/`, `vendor/`. They ship with a release and stay on the local disk of each node.
- **Cache and logs**: `var/cache/` and `var/logs/` are local to each node by design, and Smarty, Twig, GD or Imagick need a real directory. 0027 keeps a path for the module cache, and logs go through Monolog.
- **The project root.** A `default` storage rooted at the project (previous version of this proposal) would let any code write anywhere, including the read-only locations above, and cannot be moved to another backend as a whole.

### 3. Storages are injected by name, and resolved by a registry when the name is dynamic

Each storage is a service, `prestashop.storage.<name>`: they are several instances of the same class, so they cannot be named after their FQCN as ADR 0019 recommends. To keep autowiring, the core registers a **named autowiring alias** for each core storage, so a service gets one by the name of its argument:

```php
public function __construct(
    private readonly StorageInterface $imagesStorage,
) {
}
```

Code choosing the storage at runtime (a command taking the storage name as an option, a migration tool) uses a registry instead:

```php
interface StorageRegistryInterface
{
    public function get(string $name): StorageInterface;
    public function has(string $name): bool;
    public function getNames(): array;
}
```

The core names are constants of `Core\Storage\StorageName` (`StorageName::IMAGES`, ...).

Storages are collected through the `prestashop.storage` tag (attribute `storage_name`), so the registry does not have to know the list in advance. A module may register a storage of its own, for example to reach a bucket shared with another application; its name should be prefixed with the technical name of the module (`<module>_<name>`). The compiler pass refuses a tag reusing the name of a core storage, so that a module cannot replace one by accident.

### 4. The module storages of 0027 are views on `modules_private` and `modules_public`

This answers the open question of 0027:

- `ModuleStorageProviderInterface` returns `StorageInterface` instances, not a dedicated interface: the private storage of a module is `modules_private` scoped to `<module>/`, its public storage is `modules_public` scoped to `<module>/`. A module cannot reach another module's folder from its own storages.
- The method names of this ADR apply (`fileExists()`, `listFiles()`), replacing the provisional names of 0027 (`exists()`, `listContents()`). Nothing has shipped with those names.
- The private storage adds the `localPath()` escape hatch defined in 0027, through a `ModulePrivateStorageInterface` extending `StorageInterface`. It is the only method that does not carry over to another backend.
- Modules do not write to the core storages (`images`, `uploads`, `downloads`), as 0027 already reserves those directories to the core.

### 5. The implementation uses League Flysystem

`league/flysystem` is added as a core dependency, with its local adapter (`league/flysystem-local`). Flysystem is the de facto standard file storage abstraction in PHP, actively maintained, MIT licensed, with first-party adapters for S3-compatible storage, Azure, GCS, SFTP, FTP, WebDAV, in-memory and read-only.

- Only the **local adapter** is a core dependency. Other adapters are installed per project (`composer require league/flysystem-async-aws-s3`) and listed under `suggest`. The core must not carry the AWS SDK for the vast majority of shops that stay on the local disk.
- `league/flysystem-memory` is a dev dependency, so file-handling code becomes unit-testable without touching the disk.
- The core depends on Flysystem only in `PrestaShopBundle\Storage`; `Core\Storage` stays library-agnostic.
- **The storages are defined once, as plain services**, in a file loaded by the Symfony containers and by the legacy front-office container alike. `ContainerFinder` returns the legacy container in the front office, so a storage defined in one container only would break there. The Flysystem bundle is not used: its `flysystem.storages` configuration is read by the bundle, which the legacy front-office container does not load, so every storage would have to be declared twice.

### 6. Defaults preserve current behavior

Each storage uses the local adapter, rooted at the directory it uses today, resolved from the existing constants through container parameters (`prestashop.img_dir`, `prestashop.upload_dir`, `prestashop.download_dir`), so shops that override those constants keep working. Files and directories keep the 0644 / 0755 permissions PrestaShop has always used, rather than Flysystem's stricter defaults.

Introducing the layer is therefore not a behavior change, and the `_PS_*_DIR_` constants are **not** deprecated by this ADR.

### 7. A shop configures its backend in one project file

The backend of a storage is an infrastructure decision, so it does not depend on a module staying installed and active, and it is written once for every container:

- Each storage wraps a Flysystem filesystem service, `prestashop.filesystem.<name>`. The core defines them on the local disk.
- **The core loads an optional project file, `app/config/storage.yml`**, after its own definitions, in the Symfony containers and in the legacy front-office container alike. This is a change in `AppKernel` and in `Adapter\ContainerBuilder::loadServicesFromConfig()`, which only loads `config/services/front/services_<env>.yml` today. The file holds plain service definitions redefining the `prestashop.filesystem.<name>` it wants to move, which both containers understand, so the override is written once.
- The file is git-ignored and shipped as `storage.yml.dist`, with commented S3 and SFTP examples, following the `parameters.yml.dist` pattern. The upgrade tool must keep it, as it keeps `parameters.php`.
- **Credentials come from environment variables**, read with `%env()%` in that file, never from the repository or the database.
- **The containers are compiled and cached**: a change in `storage.yml` takes effect after a cache clear, as any change in the service configuration.

A module can still redefine a filesystem service, as with any service, but it is not the documented way: disabling the module would silently move the shop back to the local disk.

### 8. Consumers are migrated incrementally

The abstraction lands first, with no consumer, so that it can be reviewed on its own. Then, one pull request per area, in this order:

1. `ModuleStorageProviderInterface` of 0027, on `modules_private` and `modules_public`.
2. Entity image uploaders (category, manufacturer, supplier, employee, profile).
3. Attachments and virtual product downloads.
4. Product images, last, because thumbnail generation with GD/Imagick needs a real local file: the flow becomes read from the storage into a local temporary file, process, write back, clean up.
5. URL generation: `publicUrl()` where the shop's base URL and a path are concatenated today.

Each migration PR carries its own tests, and documents the behavior it changes. On the local disk, the storage does exactly what the native calls did, so areas 1 to 3 are plain refactoring, with no feature flag: keeping both code paths would double the maintenance and the tests for no behavior change. **A `storage` feature flag guards areas 4 and 5 only**, where the behavior does change (temporary local copies, URLs).

**Legacy code reaches the storages through the container it already uses.** Code in `classes/` has no dependency injection, and already resolves services through `PrestaShop\PrestaShop\Adapter\ContainerFinder` in many places (`Cart`, `Carrier`, `CartRule`, `ObjectModel`, ...). It does the same for the storages; no static accessor is added, as it would be a second global entry point to the container.

## Options considered and rejected

- **Writing directly against Flysystem's `FilesystemOperator`**, with no interface of our own. It saves a wrapper, but every caller would depend on the library's types and exceptions, and replacing it, or moving to its next major, would be a BC break for modules.
- **Letting modules implement `StorageInterface`.** Any method added later would break them, while a Flysystem adapter already covers every backend.
- **A single storage for the whole project**, or a `default` storage rooted at the project root (proof of concept and previous version of this proposal). A shop could not move its images without moving everything, and code could keep writing anywhere, including the read-only `modules/` of 0027.
- **A storage per image type** (`img/p/`, `img/c/`, ...). It would keep the shipped assets out of the way, but a shop moving its images to a bucket moves all of them, and nine storages would carry one decision.
- **Injecting the registry everywhere** (`get('images')`), as in the previous version of this proposal. It hides the dependency behind a string and goes against autowiring (ADR 0019); the registry stays for dynamic names only.
- **A dedicated interface for the module storages** (provisional in 0027). It would duplicate `StorageInterface` with other method names, and modules would get two vocabularies for the same operations.
- **Selecting the driver from the back office.** It would store credentials in the database, which ends up in dumps and backups, and the container is compiled before the database is available in several flows (installation, CLI, legacy front office).
- **Configuring the backend from a module only** (previous version of this proposal). An infrastructure decision would depend on a module being installed and active, and would be written twice, once per container.
- **`league/flysystem-bundle` and its `flysystem.storages` configuration** (previous version of this proposal). The legacy front-office container does not load bundles, so every storage, and every override, had to be declared twice.
- **Shipping every adapter in the core**, for the weight reason above.
- **A static accessor for legacy code** (`Storage::get('images')`). It adds a global entry point, while `ContainerFinder` already does the job.
- **A feature flag on every migrated area** (previous version of this proposal). On the local disk most areas do not change behavior, and both code paths would have to be maintained and tested.

## Consequences

- A new core dependency (`league/flysystem` and its local adapter), pinned to a version supporting the minimum PHP version of the branch.
- PrestaShop becomes able to run several instances against shared object storage, with no file system mounted between them, for the directories that have a storage.
- The module storages of 0027 can move to another backend with no change in the modules, except where they use `localPath()`.
- Code handling files becomes unit-testable with the in-memory adapter.
- A shop can serve its images and the public files of its modules from a bucket or a CDN through `publicUrl()`.
- **Friendly image URLs do not follow the images to another backend.** Product and category images are served through rewrite rules generated by `Tools::generateHtaccess()` towards `img/p/` and `img/c/`, which only work on the local disk. With `images` on a bucket, `Link` returns `publicUrl()` instead, unless the CDN in front of the bucket rewrites the same URLs. This is part of area 5 and of its feature flag.
- A new optional project file, `app/config/storage.yml`, has to be documented, kept out of the repository and preserved by the upgrade tool.
- Modules and custom code that keep using `_PS_*_DIR_` and native calls will not see files stored in another backend. Until the core consumers are migrated and the practice is documented, mixed setups are possible; this is the main risk of the whole effort.
- Object storage has no atomic rename, no real directories and a per-call latency. Code written against the storage must stop assuming otherwise, which is precisely what the migration PRs have to review area by area.

## Open questions

1. **Existence checks in the front office.** It calls `file_exists()` per image, which becomes a network round trip per call against object storage. Caching the result or assuming existence is a behavior change, to be settled with the product images migration.
