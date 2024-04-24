---
title: "Adding an algorithm"
teaching: 5
exercises: 1
objectives:
- "Understand the difference between a factory and an algorithm"
- "Understand where to put the algorithm code"
- "Understand the basic algorithm interface"
- "Understand how to call an algorithm from a factory"
---

## The difference between a factory and an algorithm

*Algorithms* are classes that perform one kind of calculation we need and they do so in a generic, framework-independent way. The core of an Algorithm is a method called `execute` which inputs some PODIO collections and outputs some other PODIO collections. Algorithms don't know or care where the inputs come from and where they go. Algorithms also don't know much about where their parameters come from; rather, they are passed a `Config` structure which contains the parameters' values. The nice thing about algorithms is that they are simple to design and test, and easy to reuse for different detectors, frameworks, or even entire experiments.


Most of what makes an Algorithm an Algorithm is convention. (These are largely inspired by the KISS principle in software engineering!) There is an ongoing effort to create a "framework-less framework" for formally expressing Algorithms using templates, which lives at https://github.com/eic/algorithms. Eventually, we may encourage users to have all Algorithms inherit from the `Algorithm<Input<...>, Output<...>>` templated interface. For now, however, just follow the Algorithm conventions that we will go over next.

## Where to put the algorithm code

All algorithms that are not specific to a single detector should go under `src/algorithms`. Because this falls in the category of reconstruction, we'll put it in `src/algorithms/reco`.


## The basic algorithm interface

Here is a template for an algorithm header file:

~~~ c++

#pragma once

// #include relevant header files here

namespace eicrecon {

    class MyAlgorithmName {

    public:
            
        // init function contains any required initialization
        void init();

        // execute function contains main algorithm processes
        // (e.g. manipulate existing objects to create new objects)
        std::unique_ptr<MyReturnDataType> execute();
        
        // Any additional public members go here 

    private:
        std::shared_ptr<spdlog::logger> m_log;
        // any additional private members (e.g. services and calibrations) go here

    };
} // namespace eicrecon

~~~

## How to call an algorithm from a factory

The code to call an algorithm from a factory generally follows a specific pattern:

```c++
    void Configure() {
        // This is called when the factory is instantiated.
        // Use this callback to make sure the algorithm is configured.
        // The logger, parameters, and services have all been fetched before this is called
        m_algo = std::make_unique<eicrecon::ElectronReconstruction>();

        // Pass config object to algorithm
        m_algo->applyConfig(config());

        // If we needed geometry, we'd obtain it like so
        // m_algo->init(m_geoSvc().detector(), m_geoSvc().converter(), logger());

        m_algo->init(logger());
    }

    void Process(int64_t run_number, uint64_t event_number) {
        // This is called on every event.
        // Use this callback to call your Algorithm using all inputs and outputs
        // The inputs will have already been fetched for you at this point.
        auto output = m_algo->execute(
          m_in_mc_particles(),
          m_in_rc_particles(),
          m_in_rc_particles_assoc(),
          m_in_clu_assoc()
        );

        m_out_reco_particles() = std::move(output);
        // JANA will take care of publishing the outputs for you.
    }
```


> Exercise:
> - Create your own ElectronReconstruction algorithm using the code skeleton above.
> - Print some log messages from your algorithm's `execute()` method. 
> - Have your ElectronReconstruction factory call the algorithm.
> - Run this end-to-end.
{: .challenge}