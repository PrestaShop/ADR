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


ADR ID | Date       | Discussion           | Title                                                | Status
------ | -----------| -------------------- | --------------------------------------------------- -| -----------------
0001   | 2019-01-22 | ~                    | [Record architecture decisions][0001]                | ✅ Accepted
0002   | 2019-02-15 | ~                    | [Mixed use of composer and zip modules][0002]        | ✅ Accepted
0003   | 2019-10-15 | ~                    | [Use of autowiring][0003]                            | ❌ Rejected
0004   | 2020-04-17 | ~                    | [Keep QA and Devs HTML selectors separate][0004]     | ❌ Rejected
0005   | 2019-01-22 | [Pull Request][0005] | Define ACL rules for Symfony pages                   | 💬 In discussion
0006   | 2020-02-13 | [Pull Request][0006] | Registration of a bundle-like module                 | 💬 In discussion
0007   | 2020-03-09 | [Pull Request][0007] | Module advanced installation                         | 💬 In discussion
0008   | 2020-03-23 | [Pull Request][0008] | Ajax error handling                                  | 💬 In discussion
0009   | 2020-08-20 | ~                    | [Expose js components using window variable][0009]   | ✅ Accepted
0010   | 2020-12-15 | [Pull Request][0010] | Module version bump convention when Core is updated  | 💬 In discussion



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
