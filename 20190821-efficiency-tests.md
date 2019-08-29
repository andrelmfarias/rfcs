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
- Have performance garantees. This is great for developers and users.
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

We plan to implement these tests in an iterative fashion. We propose two types of tests:

### Unit tests

Regular unit tests that are executed with every new commit.

#### Checking memory use and time for running basic operations with MPC

Operations that should be tested:

  * matmul
  * division
  * exp
  * add

We should test that every operations takes less than `X` seconds and that it uses less than `Y` memory.

### Benchmarks

Have concrete benchmarks similar to real use cases. These tests may take a long time to run so they should just be executed
by demand and/or before a new version is released.

Perhaps we could implement a BenchmarkWorker, a simple extension of VirtualWorker which runs time.sleep() for each message based on either a fixed waiting amount (simulating ping / latency), the size of the message (serialization) or the size of the recent cache of messages (bandwidth).
