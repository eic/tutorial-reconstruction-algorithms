---
title: "Adding an algorithm"
teaching: 5
exercises: 0
questions:
objectives:
- "Understand the difference between a factory and an algorithm"
- "Understand where to put the algorithm code"
- "Understand the basic algorithm interface"
- "Understand how to call an algorithm from a factory"

keypoints:
---

## The difference between a factory and an algorithm

*Algorithms* are classes that perform one kind of calculation we need and they do so in a generic, framework-independent way. The core of an Algorithm is a method called `execute` which inputs some PODIO collections and outputs some other PODIO collections. Algorithms don't know or care where the inputs come from and where they go. Algorithms also don't know much about where their parameters come from; rather, they are passed a `Config` structure which contains the parameters' values. The nice thing about algorithms is that they are simple to design and test, and easy to reuse for different detectors, frameworks, or even entire experiments.


Most of what makes an Algorithm an Algorithm is convention. (These are largely inspired by the KISS principle in software engineering!) There is an ongoing effort to create a "framework-less framework" for formally expressing Algorithms using templates, which lives at https://github.com/eic/algorithms. Eventually, we may encourage users to have all Algorithms inherit from the `Algorithm<Input<...>, Output<...>>` templated interface. For now, however, just follow the Algorithm conventions that we will go over next.

## Where to put the algorithm code

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



