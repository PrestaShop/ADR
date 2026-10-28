# ADR

Architecture Decision Records for the PrestaShop project

## What is this?

The purpose of this repository is to track the history of design choices on the PrestaShop project.

To learn more about ADRs, read [this article][adr].

## How do I propose a subject?

1) Use [ADR tools][adr-tools] to build your proposal, add it to the status table below, then submit a Pull Request.

2) New proposals initial status is "In discussion". If the subject is accepted for discussion, it is assigned an ID and added to the table below on `master` branch.

3) Once the discussion is over and after all necessary ADR modifications, a vote is cast in the related Pull Request.

4) All project maintainers vote, and their vote is registered. Decisions are accepted by simple majority. Discussed ADRs are merged even if they are not accepted.

## ADR status

| ADR ID | Date       | Discussion           | Title                                                                             | Status           |
|--------|------------|----------------------|-----------------------------------------------------------------------------------|------------------|
| 0001   | 2019-01-22 | -                    | [Record architecture decisions][0001]                                             | ✅ Accepted       |
| 0002   | 2019-02-15 | -                    | [Mixed use of composer and zip modules][0002]                                     | ✅ Accepted       |
| 0003   | 2019-10-15 | -                    | [Use of autowiring][0003] - amended in [FQCN Autowiring][0019]                    | ❌ Rejected       |
| 0004   | 2020-04-17 | -                    | [Keep QA and Devs HTML selectors separate][0004]                                  | ❌ Rejected       |
| 0005   | 2019-01-22 | -                    | [Define ACL rules for Symfony pages][0005]                                        | ❌ Abandoned      |
| 0006   | 2020-02-13 | -                    | [Registration of a bundle-like module][006]                                       | ❌ Abandoned      |
| 0007   | 2020-03-09 | [Pull Request][0007] | Module advanced installation                                                      | 💬 In discussion |
| 0008   | 2020-03-23 | -                    | [Ajax error handling][0008]                                                       | ❌ Abandoned      |
| 0009   | 2020-08-20 | -                    | [Expose js components using window variable][0009]                                | ✅ Accepted       |
| 0010   | 2020-12-15 | -                    | [Module version bump convention when Core is updated][0010]                       | ❌ Abandoned      |
| 0011   | 2021-01-05 | -                    | [Use constants for configuration variables][0011]                                 | ❌ Abandoned      |
| 0012   | 2021-01-13 | [Pull Request][0012] | Module advanced installation                                                      | 💬 In discussion |
| 0014   | 2021-08-25 | -                    | Drop the "No Use Of Legacy In New Code" restriction                               | ✅ Accepted       |
| 0015   | 2021-09-16 | [Pull Request][0015] | Split business logic from DOM logic of JS                                         | 💬 In discussion |
| 0016   | 2022-01-07 | [Pull Request][0016] | Use variadic to type-hint PHP arrays                                              | 💬 In discussion |
| 0017   | 2022-04-17 | -                    | [Backward compatibility promise][0017]                                            | ✅ Accepted       |
| 0018   | 2022-05-16 | -                    | [Horizontal migration][0018]                                                      | ✅ Accepted       |
| 0019   | 2022-11-16 | -                    | [FQCN Autowiring][0019]                                                           | ✅ Accepted       |
| 0020   | 2023-02-22 | -                    | [Module versioning when min compat with the core is updated][0020]                | ✅ Accepted       |
| 0021   | 2023-10-27 | -                    | [Model and Repositories aiming to replace the usages of legacy ObjectModel][0021] | ❌ Rejected       |
| 0022   | 2023-05-30 | [Pull Request][0022] | [Module storage folders][0022]                                                    | 💬 In discussion |
| 0023   | 2023-10-27 | -                    | [CQRS API quidelines][0023]                                                       | ✅ Accepted       |
| 0024   | 2023-10-27 | -                    | [Refactoring of PrestaShop context][0024]                                         | ✅ Accepted       |
| 0025   | 2023-10-27 | [Pull Request][0025] | [Doctrine entities migration][0025]                                               | 💬 In discussion |
| 0026   | 2026-10-20 | [Pull Request][0026] | [Pluggable file storage][0026]                                                    | 💬 In discussion |

[adr]: http://thinkrelevance.com/blog/2011/11/15/documenting-architecture-decisions
[adr-tools]: https://github.com/npryce/adr-tools/
[0001]: 0001-record-architecture-decisions.md
[0002]: 0002-mixed-use-of-composer-and-zip-modules.md
[0003]: 0003-use-of-autowiring.md
[0004]: 0004-keep-qa-and-devs-html-selectors-separate.md
[0005]: https://github.com/PrestaShop/ADR/pull/1
[0006]: https://github.com/PrestaShop/ADR/pull/7
[0007]: https://github.com/PrestaShop/ADR/pull/8
[0008]: https://github.com/PrestaShop/ADR/pull/9
[0009]: 0009-expose-js-components-using-window-variable.md
[0010]: https://github.com/PrestaShop/ADR/pull/14
[0011]: https://github.com/PrestaShop/ADR/pull/16
[0012]: https://github.com/PrestaShop/ADR/pull/17
[0014]: 0014-drop-the-no-use-of-legacy-in-new-code-restriction.md
[0015]: https://github.com/PrestaShop/ADR/pull/21
[0016]: https://github.com/PrestaShop/ADR/pull/24
[0017]: https://github.com/PrestaShop/ADR/pull/25
[0018]: https://github.com/PrestaShop/ADR/pull/26
[0019]: https://github.com/PrestaShop/ADR/pull/29
[0020]: https://github.com/PrestaShop/ADR/pull/32
[0021]: 0021-model-and-repositories.md
[0022]: https://github.com/PrestaShop/ADR/pull/34
[0023]: 0023-cqrs-api-guidelines.md
[0024]: https://github.com/PrestaShop/ADR/pull/36
[0025]: https://github.com/PrestaShop/ADR/pull/37
[0026]: 0026-pluggable-file-storage.md
