---
title: "Putting everything together"
teaching: 5
exercises: 0
---

::::::::::::::::::::::::::::::::::::::::::::: questions

- How do the factory, algorithm, config, and generator fit together in the final electron finder?

:::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::: objectives

- Get the electron finder running end-to-end.

:::::::::::::::::::::::::::::::::::::::::::::

## The final ReconstructedElectron factory

Here is the final `ReconstructedElectrons_factory`. The current implementation in `main` is much simpler than the variadic-input version that was originally drafted: by the time we reach the reco stage, charged tracks have already been combined with calorimeter clusters into `edm4eic::ReconstructedParticle` objects (each carrying a `getClusters()` relation), so we only need to read that one collection and apply the E/p cut on it.

`src/factories/reco/ReconstructedElectrons_factory.h`:

```c++

#pragma once

#include "extensions/jana/JOmniFactory.h"

#include "algorithms/reco/ElectronReconstruction.h"

namespace eicrecon {

class ReconstructedElectrons_factory
    : public JOmniFactory<ReconstructedElectrons_factory, ElectronReconstructionConfig> {
public:
  using AlgoT = eicrecon::ElectronReconstruction;

private:
  // Underlying algorithm
  std::unique_ptr<AlgoT> m_algo;

  // Declare inputs
  PodioInput<edm4eic::ReconstructedParticle> m_in_rc_particles{this, "ReconstructedParticles"};

  // Declare outputs
  PodioOutput<edm4eic::ReconstructedParticle> m_out_reco_particles{this};

  // Declare parameters
  ParameterRef<double> m_min_energy_over_momentum{this, "minEnergyOverMomentum",
                                                  config().min_energy_over_momentum};
  ParameterRef<double> m_max_energy_over_momentum{this, "maxEnergyOverMomentum",
                                                  config().max_energy_over_momentum};

public:
  void Configure() {
    // This is called when the factory is instantiated.
    // Use this callback to make sure the algorithm is configured.
    // The logger, parameters, and services have all been fetched before this is called
    m_algo = std::make_unique<AlgoT>(GetPrefix());
    m_algo->level(static_cast<algorithms::LogLevel>(logger()->level()));

    // Pass config object to algorithm
    m_algo->applyConfig(config());

    m_algo->init();
  }

  void Process(int32_t /* run_number */, uint64_t /* event_number */) {
    // This is called on every event.
    // Use this callback to call your Algorithm using all inputs and outputs.
    // The inputs will have already been fetched for you at this point.
    m_algo->process({m_in_rc_particles()}, {m_out_reco_particles().get()});

    logger()->debug("Found {} reconstructed electron candidates", m_out_reco_particles()->size());
  }
};
} // namespace eicrecon

```

Note that `ChangeRun()` is omitted entirely — the JOmniFactory base class provides a default no-op implementation, so a factory that doesn't need to react to run-number changes does not have to override it.

Next, we register this with the `reco` plugin in `src/global/reco/reco.cc`:
```c++
  app->Add(new JOmniFactoryGeneratorT<ReconstructedElectrons_factory>(
      "ReconstructedElectrons", {"ReconstructedParticles"}, {"ReconstructedElectrons"}, {}, app));
```

You can also create additional instances of the same factory with different parameter values. For example, the same `reco.cc` registers a second instance, `ReconstructedElectronsForDIS`, with a wider E/p window:
```c++
  app->Add(new JOmniFactoryGeneratorT<ReconstructedElectrons_factory>(
      "ReconstructedElectronsForDIS", {"ReconstructedParticles"}, {"ReconstructedElectronsForDIS"},
      {
          .min_energy_over_momentum = 0.7, // GeV
          .max_energy_over_momentum = 1.3  // GeV
      },
      app));
```

And finally, we add its output collection name to the output include list in `src/services/io/podio/JEventProcessorPODIO.cc`:

```c++
    "ReconstructedElectrons",

```


## The ElectronReconstruction algorithm

`src/algorithms/reco/ElectronReconstruction.h`:

```c++

#pragma once

#include <algorithms/algorithm.h>
#include <edm4eic/ReconstructedParticleCollection.h>
#include <string>
#include <string_view>

#include "ElectronReconstructionConfig.h"
#include "algorithms/interfaces/WithPodConfig.h"

namespace eicrecon {

using ElectronReconstructionAlgorithm =
    algorithms::Algorithm<algorithms::Input<edm4eic::ReconstructedParticleCollection>,
                          algorithms::Output<edm4eic::ReconstructedParticleCollection>>;

class ElectronReconstruction : public ElectronReconstructionAlgorithm,
                               public WithPodConfig<ElectronReconstructionConfig> {

public:
  ElectronReconstruction(std::string_view name)
      : ElectronReconstructionAlgorithm{name,
                                        {"inputParticles"},
                                        {"outputParticles"},
                                        "selected electrons from reconstructed particles"} {}

  void init() final {};
  void process(const Input&, const Output&) const final;
};

} // namespace eicrecon

```

A few things to note about the modern algorithm interface:

- The algorithm inherits from `algorithms::Algorithm<Input<...>, Output<...>>`, so its inputs and outputs are declared *as types* in the template parameter list. The compiler then checks at the call site that the right collection types are passed in.
- `WithPodConfig<ElectronReconstructionConfig>` provides the protected member `m_cfg`, which is what the algorithm uses to access its configuration values.
- Logging methods (`info`, `debug`, `trace`, `warning`, `error`) are inherited — there is no `m_log` member to manage by hand.
- `process()` is `const`. Any state that has to be set up once per job goes in `init()` (which here is empty); state that depends on the run number can be retrieved on the fly from `algorithms::GeoSvc` and friends.

The algorithm's parameters live in a Config struct at `src/algorithms/reco/ElectronReconstructionConfig.h`:

```c++
namespace eicrecon {

struct ElectronReconstructionConfig {

  double min_energy_over_momentum = 0.9;
  double max_energy_over_momentum = 1.2;
};

} // namespace eicrecon
```

The algorithm itself lives at `src/algorithms/reco/ElectronReconstruction.cc`:

```c++

#include "ElectronReconstruction.h"

#include <edm4eic/ClusterCollection.h>
#include <edm4eic/ReconstructedParticleCollection.h>
#include <edm4hep/utils/vector_utils.h>
#include <fmt/core.h>
#include <podio/RelationRange.h>
#include <gsl/pointers>

#include "algorithms/reco/ElectronReconstructionConfig.h"

namespace eicrecon {

void ElectronReconstruction::process(const Input& input, const Output& output) const {

  const auto [rcparts] = input;
  auto [out_electrons] = output;

  // out_electrons is a *subset* collection of the input ReconstructedParticles —
  // PODIO objects are owned by exactly one collection, and a subset collection
  // lets us refer to existing objects without copying or transferring ownership.
  out_electrons->setSubsetCollection();

  for (const auto particle : *rcparts) {

    // Skip particles without an associated cluster (no calorimeter info)
    if (particle.getClusters().empty()) {
      continue;
    }
    // Skip neutral particles (no track momentum to compare against)
    if (particle.getCharge() == 0) {
      continue;
    }

    double E      = particle.getClusters()[0].getEnergy();
    double p      = edm4hep::utils::magnitude(particle.getMomentum());
    double EOverP = E / p;

    trace("ReconstructedElectron: Energy={} GeV, p={} GeV, E/p = {} for PDG (from truth): {}",
          E, p, EOverP, particle.getPDG());

    // Apply the E/p cut here to select electrons
    if (EOverP >= m_cfg.min_energy_over_momentum &&
        EOverP <= m_cfg.max_energy_over_momentum) {
      out_electrons->push_back(particle);
    }
  }
  debug("Found {} electron candidates", out_electrons->size());
}

} // namespace eicrecon

```

Compared to the truth-link-based draft from earlier in the tutorial, this version is much shorter because:

- The track ↔ cluster matching has already been done upstream and is exposed via `ReconstructedParticle::getClusters()`.
- We don't need any `MCRecoParticleLink` or `MCRecoClusterParticleLink` collections — the only input is the merged `ReconstructedParticles`.
- We don't allocate new particles; we just keep references to the originals via a *subset* collection (`setSubsetCollection()`).

The exercise from earlier episodes — wiring a custom variant that takes truth links as additional inputs — is still a useful one. Compare your version with the version on `main` to see the trade-offs between richness of inputs and code simplicity.

::::::::::::::::::::::::::::::::::::::::::::: keypoints

- The final electron finder is one factory (`ReconstructedElectrons_factory`), one algorithm (`ElectronReconstruction`), and a Config struct, wired up in `src/global/reco/reco.cc`.
- Reading the pre-merged `ReconstructedParticles` (with its `getClusters()` relation) makes the algorithm short — no truth-link inputs are needed.
- The output is a subset collection of selected electrons; register additional generator instances (e.g. `ReconstructedElectronsForDIS`) for different E/p windows.
- Omit `ChangeRun()` when a factory has no run-dependent state; the base class provides a no-op default.

:::::::::::::::::::::::::::::::::::::::::::::
