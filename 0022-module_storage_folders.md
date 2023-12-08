# 20. Module storage folders

Date: 2023-05-30

## Status

Draft

## Context

Currently module developers save dynamic and static data inside module folder as there is no documented place for
storing that kind of data.

This causes problems with scaling and makes it harder to keep files "clean" in the module folder.

This ADR defines different places for module developers to save their data.

* Public files
  * Module api provides a method `getPublicStorageFolder()`
  * Core prepares new folder for the module in `var/public/modules/[modulename]/`
  * Module can use that folder as they want (create new subfolders, create files, delete files)
  * This folder is publicly accessible in `[shopurl]/var/storage/modules/[modulename]`
* Private files
  * Module api provides a method `getPrivateStorageFolder()`
  * Core prepares new folder for the module in `var/private/modules/[modulename]/`
  * Module can use that folder as they want (create new subfolders, create files, delete files)
  * This folder IS NOT publicly accessible
  * Module can still provide a url that can allow the data to be accessed (for example password protected)
* Temporary files
  * Module api provides a method `getTmpFolder()`
  * Core prepares new folder for the module in `var/cache/[environment]]modules/[modulename]/`
  * Module can use that folder as they want (create new subfolders, create files, delete files)
  * Core also providers method `getTmpFileName()` that gives a tmpfile name that is unique
  * This folder is deleted when cache is cleared

### Consequences

Module developers MUST use the api methods to get the folders and keep their files in those folders.

This keeps the module "strictly code" and files are saved elsewhere.
