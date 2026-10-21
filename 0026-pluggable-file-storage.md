# 26. Pluggable file storage

Date: 2026-10-20

## Status

In discussion

Related to [0022 - Module storage folders](https://github.com/PrestaShop/ADR/pull/34), which defines *where* modules may write on the local disk. This ADR defines *through which abstraction* the core and modules read and write files, so that the location is no longer necessarily a local disk. Both can be accepted independently; if the committers prefer, this ADR can be merged into 0022.

## Context

PrestaShop accesses files through native PHP calls (`file_put_contents`, `unlink`, `copy`, `move_uploaded_file`, `scandir`, ...) spread over the code base, resolved against global path constants (`_PS_IMG_DIR_`, `_PS_UPLOAD_DIR_`, `_PS_DOWNLOAD_DIR_`, ...). In `classes/`, `controllers/` and `src/` there are roughly 800 such calls, including 365 `file_exists`, 104 `unlink`, 87 `file_get_contents`, 79 `scandir`, 59 `copy`, 47 `mkdir`, 44 `file_put_contents` and 21 `move_uploaded_file`.

Consequences of the current situation:

- **A shop cannot store its files anywhere but the local disk.** Running several application instances requires a shared file system (NFS, `s3fs`, ...) at the infrastructure level, with the performance and consistency problems that come with it. PrestaShop is far from being cloud native.
- **There is no single place to plug anything else**: no caching of existence checks, no CDN, no instrumentation, no read-only mode.
- **Modules write files anywhere** in the project, because the core offers no sanctioned way to ask for a place to write.
- **Code that touches files is hard to test**: direct calls to `file_put_contents` cannot be mocked, so tests either hit the disk or the code stays untested.

This was raised in [discussion #41141](https://github.com/PrestaShop/PrestaShop/discussions/41141) and prototyped in [PR #41140](https://github.com/PrestaShop/PrestaShop/pull/41140) by @matthieu-rolland, and implemented in [PR #43255](https://github.com/PrestaShop/PrestaShop/pull/43255), which is kept as a draft until this ADR is settled.

Two decisions have to be taken: the abstraction the core exposes, and how a shop configures its backend.

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

- **Paths are relative to the storage root** and always use `/`, whatever the backend is.
- **No type of the underlying library appears in the signatures**, and the library's exceptions are wrapped in `Core\Storage\Exception\StorageException`. The implementation can therefore be replaced without a BC break for the callers.
- `publicUrl()` exists so that files can be served by a bucket or a CDN instead of the shop itself.
- Streams are part of the contract, so that large files (virtual products, exports) are never fully loaded in memory.

### 2. Storages are named and resolved through a factory

Rather than a single storage for the whole project, the core declares one storage per purpose, resolved by name:

```php
interface FactoryStorageInterface
{
    public function get(string $name): StorageInterface;
    public function has(string $name): bool;
    public function getNames(): array;
}
```

The core provides `images`, `uploads`, `downloads` and `default` (project root, a fallback for code working with project-wide paths). A shop can then move its product images to object storage while keeping everything else local, which is the realistic migration path.

Storages are services collected through the `prestashop.storage` tag (attribute `storage_name`), so a module declares an additional storage without redefining the factory, and the factory does not have to know the list in advance.

### 3. The implementation uses League Flysystem

`league/flysystem-bundle` is added as a core dependency. Flysystem is the de facto standard file storage abstraction in PHP, actively maintained, MIT licensed, with first-party adapters for S3-compatible storage, Azure, GCS, SFTP, FTP, WebDAV, in-memory and read-only.

- Only the **local adapter** is a core dependency. Other adapters are installed per project (`composer require league/flysystem-async-aws-s3`) and listed under `suggest`. The core must not carry the AWS SDK for the vast majority of shops that stay on the local disk.
- `league/flysystem-memory` is a dev dependency, so file-handling code becomes unit-testable without touching the disk.
- The core depends on Flysystem only in `PrestaShopBundle\Storage`; `Core\Storage` stays library-agnostic.

### 4. Defaults preserve current behavior

Each storage is configured with the local adapter, rooted at the directory it uses today, resolved from the existing constants through container parameters (`prestashop.img_dir`, `prestashop.upload_dir`, `prestashop.download_dir`), so shops that override those constants keep working. Files and directories keep the 0644 / 0755 permissions PrestaShop has always used, rather than Flysystem's stricter defaults.

Introducing the layer is therefore not a behavior change, and the `_PS_*_DIR_` constants are **not** deprecated by this ADR.

### 5. A shop configures its backend without editing core files

Storages are plain services, and PrestaShop already loads the service definitions of installed modules into every container, so a shop overrides `prestashop.filesystem.<name>` from a module of its own. Credentials come from environment variables, never from the repository or the database.

Two options were ruled out:

- **Selecting the driver from the back office.** It would store credentials in the database, which ends up in dumps and backups, and the container is compiled before the database is available in several flows (installation, CLI, legacy front office).
- **Shipping every adapter in the core**, for the weight reason above.

One point is left open for the committers: whether the core should also load an optional project-level configuration file (`app/config/config_local.yml`, and its equivalent for the legacy front-office container), so that a hosting setup does not depend on a module being installed and active, and does not have to declare the override twice. This would follow the existing `parameters.yml.dist` pattern.

### 6. Consumers are migrated incrementally, behind a feature flag

The abstraction lands first, with no consumer, so that it can be reviewed on its own. Then, one pull request per area, in this order:

1. Entity image uploaders (category, manufacturer, supplier, employee, profile).
2. Attachments and virtual product downloads.
3. Product images, last, because thumbnail generation with GD/Imagick needs a real local file: the flow becomes read from the storage into a local temporary file, process, write back, clean up.
4. URL generation: `publicUrl()` where the shop's base URL and a path are concatenated today.

Each migration PR is guarded by a `storage` feature flag, carries its own tests, and documents the behavior it changes. Areas that must stay local are out of scope: `var/cache`, `modules/`, `themes/`, `translations/`, logs and anything the autoloader or the web server reads directly.

Two points have to be settled during that migration, as they are behavior changes rather than refactoring:

- **Existence checks.** The front office calls `file_exists()` per image; against object storage that is a network round trip per call. Either the result is cached or existence is assumed.
- **Legacy access.** Code in `classes/` has no dependency injection, so reaching a storage requires either a dedicated static accessor or a container lookup per call site.

## Consequences

- A new core dependency (`league/flysystem-bundle`), pinned to a version supporting the minimum PHP version of the branch.
- PrestaShop becomes able to run several instances against shared object storage, with no file system mounted between them.
- Modules get a sanctioned way to read and write files, complementing ADR 0022: *where* is defined by the storage they ask for, not by a path they build themselves.
- Code handling files becomes unit-testable with the in-memory adapter.
- A shop can serve its images from a bucket or a CDN through `publicUrl()`.
- Modules and custom code that keep using `_PS_*_DIR_` and native calls will not see files stored in another backend. Until the core consumers are migrated and the practice is documented, mixed setups are possible; this is the main risk of the whole effort and the reason for the feature flag.
- Object storage has no atomic rename, no real directories and a per-call latency. Code written against the storage must stop assuming otherwise, which is precisely what the migration PRs have to review area by area.
