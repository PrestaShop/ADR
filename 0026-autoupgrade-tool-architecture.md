# 26. AutoUpgrade Tool Architecture

Date: 2024-01-05

## Status

Accepted

## Context

The AutoUpgrade tool is a standalone application that is used to upgrade a PrestaShop instance from one version to another.
This tool is a command line tool but also a standalone web application that can be used by anyone, even if they don't have much technical knowledge about PrestaShop.
To avoid some issues about maintaining the tool, we need to define a clear architecture for it and also some rules about how to develop it, that must be followed by all developers.

This ADR is there to define the architecture of the tool and the rules to follow when developing this tool.

## Repositories structure

As the tool is a standalone application, it **must not** depend on PrestaShop core.

We will have 3 repositories for the tool:
- **[CoreUpgradeBundle](https://github.com/PrestaShop/CoreUpgradeBundle)**: It will contain all the services and classes used to upgrade instances of PrestaShop and all the features of the tool.
- **[CliUpgrade](https://github.com/PrestaShop/CLIUpgrade)**: It will contain the command line application that linked to the **CoreUpgradeBundle** features.
- **WebUpgrade**: It will contain the web application that linked to the **CoreUpgradeBundle** features.

**CliUpgrade**and **WebUpgrade** have **CoreUpgradeBundle** as dependency.

## Core Upgrade Bundle

### General

This is a bundle that contains all the features of the tool. It will contain all the services and classes used to upgrade instances of PrestaShop. All the business logic of the tool but also all the upgrade scripts and SQLs will be in this bundle.

All services must inject a [LoggerInterface](https://symfony.com/doc/current/components/console/logger.html) to log all the actions done by the tool. The logger must be injected in the constructor of the service.

All services must be tested.

### Architecture

TBD

## Cli Upgrade

### General

This is a Symfony Console Application that contains CLI commands relying on the core upgrade bundle.

As we use Symfony Console, this Cli app must use the Symfony guidelines and best practices that you can find [here](https://symfony.com/doc/current/console.html).

**This app is mostly an empty shell and should not contain any business logic.**

To be consistent, the commands should be named with the following convention: `ps-upgrade:command-name` and should be registering in the `src/Command` folder.

Use [OutputInterface](https://symfony.com/doc/current/logging/monolog_console.html) for all the outputs of the tool and respect the [Symfony verbose levels](https://symfony.com/doc/current/console/verbosity.html).

All commands and services used in it, must be tested.

### Architecture

TBD

## Web Upgrade
TBD