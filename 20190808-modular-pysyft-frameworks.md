# Modular PySyft Frameworks

| Status        | Proposed        |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Jason Mancuso (jason@dropoutlabs.com), Yann Dupis (yann@dropoutlabs.com) |
| **Sponsor**   | Bobby Wagner (bobby@openmined.org)                  |
| **Updated**   | 2019-08-08                                           |               


## Objective

- Make PySyft framework-agnostic (PyTorch, TensorFlow etc.), supporting a more modular and extensible design
- Identify the core PySyft API, with the eventual goal of offering stability guarantees
- Move framework-specific code to a “framework binding" repository (e.g. PySyft-PyTorch, PySyft-Tensorflow etc.)
- Enable pip installing PySyft with a specific framework or several (e.g. `pip install syft[“torch”]`, `pip install syft[“tensorflow”]`, `pip install syft[“all”]`, etc.)

## Motivation

PySyft aims to be a generic library for encrypted, privacy preserving deep learning. For any data science or deep learning framework, it should provide the same interface and functionalities. The user should be able to install PySyft for one specific framework or several without installing the rest. For example `pip install syft['torch']` or `pip install syft['tensorflow']`. As more frameworks are added, the build process for PySyft main will become bloated and unwieldy. PySyft needs to be made as modular as possible while still maintaining a core API and stability guarantees around that API.

To that end, we propose breaking out framework specific constructions into their own optional repositories. Most of the PySyft functionalities have been abstracted in such a way that it can support several frameworks. However, there are still some framework specific implementations in the code, most significantly around the original torch framework binding.

## User Benefit
Eventually, we want to get to a point at which the core Syft API is stable and extensible, so that it can support an ecosystem for PPML. Making the frameworks modular will be a prerequisite for serious extensions of the API, so we need to do this eventually to support the variety of packages and applications we’re expecting. This change moves PySyft closer to its expected maturity, which benefits all users.

Secondary benefits include:
Users or downstream packages who want to install or deploy PySyft will not need to install packages that they will not use. This is particularly useful in the case of large frameworks like TensorFlow or PyTorch, since installation (and therefore CI testing) slows considerably with these extra packages installed. Eventually, different CI runs can be triggered in parallel for different dependency setups.
Improve the memory footprint of the main package, allowing users of lower memory devices to build PySyft from source without modifying its source code or removing large dependencies.
Helpful for deploying PySyft in large enterprises, as smaller packages are easier for security teams to audit and/or pentest.

## Design Proposal
We propose managing multiple frameworks by moving their implementation into separate “framework binding” repositories.  We have identified 3 core abstractions that are required to be compatible with higher level functionalities, which we’ve termed the Syft API:


- **Serde**: Serialization and deserialization strategies for framework-specific objects.
- **Hook**: The hook which allows Python or framework-specific commands to be used as Remote Procedure Calls (RPCs). The prototypical `TorchHook` does so by monkey-patching the torch module (or torch.Tensor) with RPC analogues of the original functions (or methods).
- **NativeTensor**: The top of a PySyft’s monadic tensor chain. This is usually a requirement for hooked commands to be able to operate on custom PySyft tensors (since frameworks often have type checks in their codebases that we can’t remove without building each from source).  See the [PySyft paper](https://arxiv.org/abs/1811.04017) for more info on the monadic tensor chain abstraction.  The PyTorch framework implements this as `TorchTensor`, and it currently lives in `syft/frameworks/torch/tensors/interpreters/native.py`.

Going forward, PySyft will develop as much of its higher-level functionality in a framework-agnostic way, often aided by common base types or classes (see below for examples). PySyft features like Plans, and eventually multi-worker Plans (i.e. Protocols), will be implemented over abstractions of framework-specific objects. Eventually, this may allow code for federated learning, secure computation, and differential privacy to be developed and maintained in a single place, instead of across different framework binding repositories.

Occasionally, there may be features that are exceptions to this rule; in these cases the feature should come up with a well-defined API and be added to the above list pending approval through a sufficient RFC process.

## Detailed Design

#### Abstracting existing code

Tensor type checks in the existing code will be made generic & agnostic to which framework is in use. The common if `isinstance(obj, torch.Tensor)` in the current code will be converted to `isinstance(obj, FrameworkTensor)`, with `FrameworkTensor = Union[torch.Tensor, tf.Tensor,...]`. This union will be done conditionally depending on which tensor frameworks are installed.  For an example of implementation, see [`syft/frameworks/types.py`](https://github.com/OpenMined/PySyft/blob/d09b003ff02a5f44ccdb4b7000114f783b147a12/syft/frameworks/types.py) in the draft PR associated with this RFC.

All tensor types that don’t critically depend on torch will be made framework-agnostic and moved from syft/frameworks/torch/tensors to a syft/tensors module.  Tensors that critically depend on torch will first be moved into the syft-torch bindings repo. Eventually, we would like these to be made into abstract base classes in syft/tensors, which syft-torch and syft-tensorflow will each extend as needed.

TorchHook will inherit from a `BaseHook` class, which inherits from `abc.ABC`. Subsequent hooks will extend `BaseHook`.  So far, we've identified the following as high priority targets as functionalities required for this abstraction to work:
- `BaseHook.create_wrapper` -- determines how a wrapping NativeTensor should be created (e.g. `torch.Tensor`, `tf.constant`, etc.)
- `BaseHook.create_shape` -- determines how a shape object should be created (e.g. `torch.Size`, `tf.TensorShape`, etc.)
- `hook_args.py` -- any commonly-used, public methods from this module will need to be abstracted. We expect to reuse most of this code, mainly extending the existing logic with the necessary framework-specific code that's functionally equivalent.

#### Packaging

We plan on packaging these repositories in a way that is simplified for users above via pip package dependencies.  The dependency chain will be:
`pip install syft[‘torch’]` ← `pip install syft-torch` ← `pip install syft[‘base’]`

This is not a true circular dependency, since pip knows to skip previously installed dependencies. Parts of the syft codebase that require importing and/or using framework bindings will be handled with dependency checks to avoid import errors. Special care will be taken to avoid circular imports, which may require some light refactoring of the directory structure.  For an example, see the following from `syft/frameworks/tensorflow/__init__.py`:

```python
import syft

if syft.dependency_check.tensorflow_available:
    from syft_tensorflow import bind_tensorflow
    from syft_tensorflow.serde import MAP_TF_SIMPLIFIERS_AND_DETAILERS
    from syft_tensorflow.tensor import TensorFlowTensor

     bind_tensorflow(syft)
```

By default, pip install syft will install the pytorch + tfe version to maintain compatibility with the Udacity course.

## Questions and Discussion Topics

- Where should serde live in the main repo? In theory, it should only be needed in the workers, and this could make the library even more modular, but right now serde is used throughout the higher-level functionalities as well. This is likely out of scope for this RFC, but deserves some thought -- perhaps even an RFC of its own.
- It would be useful to have more definition around how PySyft works with various Syft workers in other languages.  We recognize a crucial need for this, however recommend moving it to a different RFC with different authors/sponsors.
