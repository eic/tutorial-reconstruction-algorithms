---
title: "Adding an algorithm"
teaching: 5
exercises: 1
---

::::::::::::::::::::::::::::::::::::::::::::: questions

- What is the difference between a factory and an algorithm?
- Where does algorithm code live, what does its interface look like, and how does a factory call it?

:::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::: objectives

- Understand the difference between a factory and an algorithm.
- Understand where to put the algorithm code.
- Understand the basic algorithm interface.
- Understand how to call an algorithm from a factory.

:::::::::::::::::::::::::::::::::::::::::::::

## The difference between a factory and an algorithm

*Algorithms* are classes that perform one kind of calculation we need and they do so in a generic, framework-independent way. The core of an Algorithm is a method called `process` which takes a tuple of input PODIO collections and a tuple of output PODIO collections. Algorithms don't know or care where the inputs come from and where they go. Algorithms also don't know much about where their parameters come from; rather, they are passed a `Config` structure which contains the parameters' values. The nice thing about algorithms is that they are simple to design and test, and easy to reuse for different detectors, frameworks, or even entire experiments.


In EICrecon, all new algorithms inherit from the templated `algorithms::Algorithm<Input<...>, Output<...>>` interface (provided by the [eic/algorithms](https://github.com/eic/algorithms) "framework-less framework"), and use the `WithPodConfig<ConfigT>` mixin to attach a configuration struct. The `algorithms::Algorithm` base provides logging facilities (`info()`, `debug()`, `trace()`, ...) and a structured way of declaring inputs and outputs by their PODIO collection types.

## Where to put the algorithm code

All algorithms that are not specific to a single detector should go under `src/algorithms`. Because this falls in the category of reconstruction, we'll put it in `src/algorithms/reco`.


## The basic algorithm interface

Here is a template for an algorithm header file:

```c++

#pragma once

#include <algorithms/algorithm.h>
// #include relevant edm4eic / edm4hep collection headers here

#include "MyAlgorithmNameConfig.h"
#include "algorithms/interfaces/WithPodConfig.h"

namespace eicrecon {

    using MyAlgorithmNameAlgorithm =
        algorithms::Algorithm<algorithms::Input<MyInputCollection>,
                              algorithms::Output<MyOutputCollection>>;

    class MyAlgorithmName : public MyAlgorithmNameAlgorithm,
                            public WithPodConfig<MyAlgorithmNameConfig> {

    public:

        MyAlgorithmName(std::string_view name)
            : MyAlgorithmNameAlgorithm{name,
                                       {"inputCollectionName"},
                                       {"outputCollectionName"},
                                       "Short description of what this algorithm does."} {}

        // init() is called once before processing starts. Most algorithms do not need it.
        void init() final {};

        // process() does the actual work for each event. The Input/Output tuples
        // contain pointers to the PODIO collections.
        void process(const Input&, const Output&) const final;

    };
} // namespace eicrecon

```

A few things worth noting:

- The class is *templated* on the list of input and output collection types. The `Input` and `Output` aliases inside the class expand into `std::tuple` of pointers (`gsl::not_null<const T*>` for inputs, `T*` for outputs).
- `process()` is `const` — algorithms must not mutate their own state during event processing. Run-by-run state should be set up in `init()` instead.
- Logging is inherited from `algorithms::AlgorithmBase`, so inside `process()` you simply call `info(...)`, `debug(...)`, or `trace(...)` directly — no logger pointer needs to be passed in.
- The configuration struct is held by `WithPodConfig` and accessible as the protected member `m_cfg`.

The corresponding implementation file unpacks the input and output tuples with structured bindings:

```c++

#include "MyAlgorithmName.h"

namespace eicrecon {

    void MyAlgorithmName::process(const Input& input, const Output& output) const {

        const auto [in_particles] = input;
        auto [out_particles]      = output;

        // ... fill out_particles using in_particles and m_cfg ...
    }

} // namespace eicrecon

```

## How to call an algorithm from a factory

The code to call an algorithm from a factory generally follows a specific pattern:

```c++
    void Configure() {
        // This is called when the factory is instantiated.
        // Use this callback to make sure the algorithm is configured.
        // The logger, parameters, and services have all been fetched before this is called

        // Construct the algorithm with the factory's prefix as its name —
        // this is what hooks the algorithm's logger up to the same prefix as the factory.
        m_algo = std::make_unique<eicrecon::ElectronReconstruction>(GetPrefix());

        // Forward the JANA log level down to the algorithm.
        m_algo->level(static_cast<algorithms::LogLevel>(logger()->level()));

        // Pass the config object to the algorithm.
        m_algo->applyConfig(config());

        // Call init() once. Note that init() takes no arguments — services
        // (e.g. geometry) are accessed by the algorithms framework via the
        // algorithms::ServiceSvc / algorithms::GeoSvc, not by passing pointers in.
        m_algo->init();
    }

    void Process(int32_t /* run_number */, uint64_t /* event_number */) {
        // This is called on every event. The inputs will have already been fetched.
        // Call process() with brace-enclosed lists of input pointers and output pointers.
        m_algo->process({m_in_particles()}, {m_out_particles().get()});
    }
```


::::::::::::::::::::::::::::::::::::::::::::: challenge

## Exercise

- Create your own ElectronReconstruction algorithm using the code skeleton above.
- Print some log messages from your algorithm's `process()` method using `info(...)` / `debug(...)`.
- Have your ElectronReconstruction factory call the algorithm.
- Run this end-to-end.

::::::::::::::: solution

Create `ElectronReconstruction.h`/`.cc` (plus a `ElectronReconstructionConfig.h`) under
`src/algorithms/reco`, inheriting from `algorithms::Algorithm<Input<...>, Output<...>>` and
`WithPodConfig`. Add `debug(...)` calls in `process()`. In the factory's `Configure()`, construct
the algorithm with `GetPrefix()`, `applyConfig(config())`, and `init()`; in `Process()` call
`m_algo->process({...}, {...})`. Rebuilding EICrecon and running `eicrecon` at debug log level
should show your messages.

:::::::::::::::

:::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::: keypoints

- Algorithms do framework-independent work; their core is a `const process(const Input&, const Output&)` method.
- Reusable algorithms live under `src/algorithms` (here `src/algorithms/reco`).
- Algorithms inherit logging and declare inputs/outputs as template types; config lives in `m_cfg` via `WithPodConfig`.
- A factory constructs the algorithm in `Configure()` (`applyConfig`, `init`) and calls `process()` in `Process()`.

:::::::::::::::::::::::::::::::::::::::::::::
