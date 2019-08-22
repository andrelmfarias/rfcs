# Title of RFC

| Status        | Proposed                                             |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Marianne Monteiro, Andre Farias                      |
| **Sponsor**   | ?                                                    |
| **Updated**   | 2019-08-21                                           |

## Objective

- Have solid benchmarks and efficiency tests for PySyft using toy (interesting) examples
- Have solid unit tests measuring the efficiency of operations on a low level (matmul, exp, add, ...)

**When we say efficiency we mean: memory efficiency and time efficiency.**

We need this mainly to:
- Have garantees about execution time. This is great for developers and users.
- Avoid breaking things. Track operations time.
- Help debugging. We have a lot of dependencies which can result on efficiency changes, having this will
help as track and debug efficiency problems.
[We already have relevant issues open related to efficiency problems](https://github.com/OpenMined/PySyft/issues?q=is%3Aopen+is%3Aissue+label%3Aefficiency)

### Criteria

- These changes should include not only local tests/benchmarks (VirtualWorkers) but
also remote tests/benchmarks (WebSocketWorkers)
- It should be easily extendable (aka adding new tests/benchmarks should not be a painful experience)
- Benchmarks execution will potentially require a lot of computing time, so this should be in the unit tests.
We need to find a way to run this by demand and ideally before releasing a new syft version

A clear and concise description of what will be implemented and why.

## Motivation

This problem is valuable to solve because:
- it already affects users directly
- it is one of the main reasons why a user may choose not use syft
- currently is a pain for developers to debug and fix efficiency related problems

## User Benefit

Users will have explictly garantees / examples of what performance to expect of Syft (currently this does not exist).

## Design Proposal

*TODO*

This is the meat of the document, where you explain your proposal. If you have
multiple alternatives, be sure to use sub-sections for better separation of the
idea, and list pros/cons to each approach. If there are alternatives that you
have eliminated, you should also list those here, and explain why you believe
your chosen approach is superior.

Factors to consider include:

* performance implications
* dependencies
* maintenance
* platforms and environments impacted (e.g. hardware, cloud, other software
  ecosystems)
* compatibility
* user impact (how will this change impact users, and how will that be managed?)
* codebase impact (will this require a lot of changes in the codebase or change how something operates?)
* alternative solutions (divide this into sections)
* concerns (challenges? deadlines? dependencies?)

If convenient add images, diagrams, code snippets.


## Related resources

*TODO*

Link any relevant resources related to this RFC. Example: issues, PRs, blog posts, ...

## Questions and Discussion Topics

*TODO*

Seed this with open questions you require feedback on from the RFC process.
