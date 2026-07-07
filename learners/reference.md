---
title: 'Reference'
---

## Glossary

Algorithm
:   A class that performs one kind of calculation in a generic, framework-independent way. In
    EICrecon, algorithms inherit from `algorithms::Algorithm<Input<...>, Output<...>>` and live
    under `src/algorithms`.

Factory
:   A class that attaches an algorithm to the JANA2 framework, handling inputs, outputs, parameters,
    services, and initialization. New factories use `JOmniFactory` and live under `src/factories`.

JANA2
:   The event-processing framework used by EICrecon. It manages parallel event processing,
    factories, plugins, and services.

JOmniFactory
:   The modern JANA2 factory base class used for all new EICrecon factories. It declares its inputs,
    outputs, parameters, and services as registered members and uses the Curiously Recurring
    Template Pattern.

Factory generator
:   A `JOmniFactoryGeneratorT` object that acts as a recipe JANA2 uses to instantiate factories,
    assigning each instance a unique prefix and its input/output collection names.

Plugin
:   A JANA2 mechanism controlling which parts of EICrecon are compiled and linked. Each detector and
    benchmark corresponds to a plugin that registers its factory generators in an `InitPlugin()`
    method.

PODIO
:   A toolkit that generates data-model classes from a YAML specification. The EIC data model
    `edm4eic` (built on `edm4hep`) is implemented with PODIO.

Subset collection
:   A PODIO collection that refers to objects owned by another collection without owning them
    itself, created with `setSubsetCollection()`.

Config struct
:   A plain-old-data struct holding an algorithm's or factory's parameters, exposed to a factory via
    `config()` and to an algorithm via the `WithPodConfig` mixin's `m_cfg` member.
