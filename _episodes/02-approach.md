---
title: "Approach"
teaching: 10
exercises: 0
questions:
objectives:
- "Understand relevent EICrecon concepts/considerations"
keypoints:
---

## Working with PODIO objects


## Algorithms vs. factories

We make a crisp distinction between *algorithms* and *factories*. 

### What is an Algorithm?

*Algorithms* are classes that perform one kind of calculation we need and they do so in a generic, framework-independent way. The core of an Algorithm is a method called `execute` which inputs some PODIO collections and outputs some other PODIO collections. Algorithms don't know or care where the inputs come from and where they go. Algorithms also don't know much about where their parameters come from; rather, they are passed a `Config` structure which contains the parameters' values. The nice thing about algorithms is that they are simple to design and test, and easy to reuse for different detectors, frameworks, or even entire experiments.


Most of what makes an Algorithm an Algorithm is convention. (These are largely inspired by the KISS principle in software engineering!) There is an ongoing effort to create a "framework-less framework" for formally expressing Algorithms using templates, which lives at https://github.com/eic/algorithms. Eventually, we may encourage users to have all Algorithms inherit from the `Algorithm<Input<...>, Output<...>>` templated interface. For now, however, just follow the Algorithm conventions that we will go over later.


### What is a Factory?

*Factories* are classes that attach Algorithms to the JANA framework. Once you have written an Algorithm, you should write a Factory so that JANA can use it. The factory layer handles issues like obtaining all of the inputs from other factories, publishing the outputs so that other factories can use them or they can be written to file, obtaining the correct parameters, making sure the Algorithm has been initialized, and making sure that the correct calibrations are loaded when the run number changes.


Here's an example to help illustrate what goes into the Algorithm and what goes into the Factory. Consider calorimeter clustering. The clustering algorithm should be independent of any individual detector, and have a set of parameters that control its behavior and live in a plain-old-data `Config` object. You could copy-paste this code into a codebase that uses a completely different reconstruction framework and it would still work, as long as you were using the same datamodel (e.g. `edm4hep`). Each detector could have its own factory (if it calls the algorithm in a substantially different way) or they may all use the same factory (if the factories only differ in their parameter values). The parameter values themselves could be hardcoded to the factory, but we strongly prefer to set them externally using a factory generator. This gives us a cleaner separation of configuration from code, and will let us do fun things in the future such as wiring factories together from an external config file, or performing parameter studies.


## Which factory base class should I use?

There are a number of different kinds of factories available in JANA which we have used within EICrecon at different points in time. Luckily, if you are writing an Algorithm from scratch, there is only one you will need to be familiar with: `JOmniFactory`. However, some of the earlier ones are still around, and just in case you need to modify or reuse those, here is a quick history lesson. 


`JFactoryT<T>` is JANA's fundamental factory base class. However, we don't use it in EICrecon because it has the following limitations:

1. It has difficulty with PODIO data. PODIO data needs very special handling, otherwise it will leak memory or corrupt your object associations. To address this, we developed `JFactoryPodioT`, which extends `JFactoryT` to support PODIO correctly.

2. *It can only output one collection.* This might seem fine at first, but frequently we need to output "Association" collections alongside the primary output collection. To address this, we developed `JMultifactory`, which supports multiple outputs, including PODIO data. 

3. If you want to reuse an Algorithm in a different context, you need to duplicate the JFactoryT/JPodioFactoryT/JMultifactory. Until this point, collection and parameter names were hardcoded inside individual factories. To get around this, we developed `JChainMultifactoryT` so that we could create multiple instances of the same factory and assign them different collection and parameter names in a logical way.

4. It requires a deeper understanding of JANA internals to use correctly. The user is allowed to perform actions inside the factory callbacks that don't necessarily make sense. We remedied this issue by developing `JOmniFactory`, which *declares* what it needs upfront, and JANA *provides* it only when it makes sense. `JOmniFactory` supports all of the functionality developed for points (1), (2), and (3), and presents a simpler interface. 


In summary, always use `JOmniFactory` if you are writing something new. All existing factories in EICrecon are in the process of being migrated right now: https://github.com/eic/EICrecon/issues/1176.


## What are the basic pieces of an OmniFactory?
// TODO: Explain inputs, outputs, parameters, services, resources
// TODO: Explain JOmniFactoryGeneratorT
// TODO: Explain callbacks


## What are the basic pieces of a JChainMultifactory?
// TODO: Explain Init, ChangeRun, Process, Finish
// TODO: Explain JChainMultifactoryGeneratorT


## I've created an Algorithm and a corresponding OmniFactory. How do I run it?

// TODO: Explain why we need a factory generator and where the factory generator should live
// TODO: Explain the factory prefix and how you can rewire parameter and collection names at the command line/config file
// TODO: Discuss the podio::output_include_collections and how the lists get defaulted/overridden

```bash
eicrecon -Ppodio:output_include_collections=MyNewCollectionName in.root
```



