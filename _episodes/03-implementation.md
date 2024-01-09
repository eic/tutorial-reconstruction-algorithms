---
title: "Implementation: algorithm and factory for electron ID example"
teaching: 25
exercises: 15
questions:
objectives:
- "Implement a reconstruction algorithm and factory"
keypoints:
---

## Create algorithm for reconstructed electron ID

First, we need to create an algorithm.  Here is a template for an algorithm header file:

~~~ c++

#pragma once

// #include relvent header files here

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
        // any additional private members go here

    };
} // namespace eicrecon

~~~

Let's look at the algorithm header from our example.

`src/algorithms/reco/ElectronReconstruction.h`:

~~~ c++

#pragma once

#include <edm4eic/MCRecoClusterParticleAssociationCollection.h>
#include <edm4eic/MCRecoParticleAssociationCollection.h>
#include <edm4eic/ReconstructedParticleCollection.h>
#include <edm4hep/MCParticleCollection.h>
#include <spdlog/logger.h>
#include <memory>
#include <vector>


namespace eicrecon {

    class ElectronReconstruction {

    public:

        // Initialization will set the pointer of the logger
        void init(std::shared_ptr<spdlog::logger> logger);

        // Algorithm will apply E/p cut to reconstructed tracks based on truth track-cluster associations
        std::unique_ptr<edm4eic::ReconstructedParticleCollection> execute(
                const edm4hep::MCParticleCollection *mcparts,
                const edm4eic::ReconstructedParticleCollection *rcparts,
                const edm4eic::MCRecoParticleAssociationCollection *rcassoc,
                const std::vector<const edm4eic::MCRecoClusterParticleAssociationCollection*> &in_clu_assoc
        );
        
        // Could overload execute here to, for instance, use track projections 
        // for track-cluster association (instead of truth information)
        
        // Function to set E/p cuts
        void setEnergyOverMomentumCut( double minEoP, double maxEoP ) { 
            min_energy_over_momentum = minEoP; 
            max_energy_over_momentum = maxEoP; 
        }

    private:
        std::shared_ptr<spdlog::logger> m_log;  // pointer of logger
        double m_electron{0.000510998928};      // electron mass
        double min_energy_over_momentum{0.9};   // default minimum E/p
        double max_energy_over_momentum{1.2};   // default maximum E/p

    };
} // namespace eicrecon

~~~

`src/algorithms/reco/ElectronReconstruction.cc`:

~~~ c++

#include "ElectronReconstruction.h"

#include <edm4eic/ClusterCollection.h>
#include <edm4hep/utils/vector_utils.h>
#include <fmt/core.h>

namespace eicrecon {

  void ElectronReconstruction::init(std::shared_ptr<spdlog::logger> logger) {
    m_log = logger;
  }

  std::unique_ptr<edm4eic::ReconstructedParticleCollection> ElectronReconstruction::execute(
    const edm4hep::MCParticleCollection *mcparts,
    const edm4eic::ReconstructedParticleCollection *rcparts,
    const edm4eic::MCRecoParticleAssociationCollection *rcassoc,
    const std::vector<const edm4eic::MCRecoClusterParticleAssociationCollection*> &in_clu_assoc
    ) {

        // Step 1. Loop through MCParticle - cluster associations
        // Step 2. Get Reco particle for the Mc Particle matched to cluster
        // Step 3. Apply E/p cut using Reco cluster Energy and Reco Particle momentum

        // Some obvious improvements:
        // - E/p cut from real study optimized for electron finding and hadron rejection
        // - use of any HCAL info?
        // - check for duplicates?

        // output container
        auto out_electrons = std::make_unique<edm4eic::ReconstructedParticleCollection>();

        for ( const auto *col : in_clu_assoc ){ // loop on cluster association collections
          for ( auto clu_assoc : (*col) ){ // loop on MCRecoClusterParticleAssociation in this particular collection
            auto sim = clu_assoc.getSim(); // McParticle
            auto clu = clu_assoc.getRec(); // RecoCluster

            m_log->trace( "SimId={}, CluId={}", clu_assoc.getSimID(), clu_assoc.getRecID() );
            m_log->trace( "MCParticle: Energy={} GeV, p={} GeV, E/p = {} for PDG: {}", clu.getEnergy(), edm4hep::utils::magnitude(sim.getMomentum()), clu.getEnergy() / edm4hep::utils::magnitude(sim.getMomentum()), sim.getPDG() );


            // Find the Reconstructed particle associated to the MC Particle that is matched with this reco cluster
            // i.e. take (MC Particle <-> RC Cluster) + ( MC Particle <-> RC Particle ) = ( RC Particle <-> RC Cluster )
            auto reco_part_assoc = rcassoc->begin();
            for (; reco_part_assoc != rcassoc->end(); ++reco_part_assoc) {
              if (reco_part_assoc->getSimID() == (unsigned) clu_assoc.getSimID()) {
                break;
              }
            }

            // if we found a reco particle then test for electron compatibility
            if ( reco_part_assoc != rcassoc->end() ){
              auto reco_part = reco_part_assoc->getRec();
              double EoverP = clu.getEnergy() / edm4hep::utils::magnitude(reco_part.getMomentum());
              m_log->trace( "ReconstructedParticle: Energy={} GeV, p={} GeV, E/p = {} for PDG (from truth): {}", clu.getEnergy(), edm4hep::utils::magnitude(reco_part.getMomentum()), EoverP, sim.getPDG() );

              // Apply the E/p cut here to select electons
              if ( EoverP >= min_energy_over_momentum && EoverP <= max_energy_over_momentum ) {
                out_electrons->push_back(reco_part.clone());
              }

            } else {
              m_log->debug( "Could not find reconstructed particle for SimId={}", clu_assoc.getSimID() );
            }

          } // loop on MC particle to cluster associations in collection
        } // loop on collections

        m_log->debug( "Found {} electron candidates", out_electrons->size() );
        return out_electrons;
    }

}

~~~

## Create reconstructed electron factory

Next, we will create a factory to call our algorithm and save the output.  Our algorithm requires reconstructed tracks, calorimeter clusters, and associations between the two.  As previously stated, the current implementation of the simple electron ID uses the truth association between tracks and clusters (association using matching between clusters and track projections will be implemented later).  Thus, we need two sets of associations: association between the truth particle and the reconstructed charged particle, and association between the truth particle and the calorimeter cluster.  Obviously, we will not create these objects from scratch.  Rather, we will get them from the factories (and underlying algorithms) implemented to create these objects.


`src/global/reco/ReconstructedElectrons_factory.h` in branch `nbrei_variadic_omnifactories`:

~~~ c++

#pragma once

#include "extensions/jana/JOmniFactory.h"

#include "algorithms/reco/ElectronReconstruction.h"


namespace eicrecon {

class ReconstructedElectrons_factory : public JOmniFactory<ReconstructedElectrons_factory> {
private:

    // Underlying algorithm
    std::unique_ptr<eicrecon::ElectronReconstruction> m_algo;

    // Declare inputs
    PodioInput<edm4hep::MCParticle> m_in_mc_particles {this, "MCParticles"};
    PodioInput<edm4eic::ReconstructedParticle> m_in_rc_particles {this, "ReconstructedChargedParticles"};
    PodioInput<edm4eic::MCRecoParticleAssociation> m_in_rc_particles_assoc {this, "ReconstructedChargedParticleAssociations"};

    VariadicPodioInput<edm4eic::MCRecoClusterParticleAssociation> m_in_clu_assoc {this};

    // Declare outputs
    PodioOutput<edm4eic::ReconstructedParticle> m_out_reco_particles {this};

    // Declare parameters here, e.g.
    // ParameterRef<double> m_samplingFraction {this, "samplingFraction", config().sampFrac};
    // ParameterRef<std::string> m_energyWeight {this, "energyWeight", config().energyWeight};

    // Declare services here, e.g.
    // Service<DD4hep_service> m_geoSvc {this};

public:
    void Configure() {
        // This is called when the factory is instantiated.
        // Use this callback to make sure the algorithm is configured.
        // The logger, parameters, and services have all been fetched before this is called
        m_algo = std::make_unique<eicrecon::ElectronReconstruction>();

        // If we had a config object, we'd apply it like so:
        // m_algo->applyConfig(config());

        // If we needed geometry, we'd obtain it like so
        // m_algo->init(m_geoSvc().detector(), m_geoSvc().converter(), logger());

        m_algo->init(logger());
    }

    void ChangeRun(int64_t run_number) {
        // This is called whenever the run number is changed.
        // Use this callback to retrieve state that is keyed off of run number.
        // This state should usually be managed by a Service.
        // Note: You usually don't need this, because you can declare a Resource instead.
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

        logger()->debug( "Event {}: Found {} reconstructed electron candidates", event_number, output->size() );

        m_out_reco_particles() = std::move(output);
        // JANA will take care of publishing the outputs for you.
    }
};
} // namespace eicrecon

~~~
Next, we register this with the `reco` plugin in src/global/reco.cc:
```c++
    app->Add(new JOmniFactoryGeneratorT<ReconstructedElectrons_factory>(
        "ReconstructedElectrons",
        {"MCParticles", "ReconstructedChargedParticles", "ReconstructedChargedParticleAssociations",
        "EcalBarrelScFiClusterAssociations",
        "EcalEndcapNClusterAssociations",
        "EcalEndcapPClusterAssociations",
        "EcalEndcapPInsertClusterAssociations",
        "EcalLumiSpecClusterAssociations",
        },
        {"ReconstructedElectrons"},
        app
    ));
```


And finally, we add its output collection name to the output include list in src/services/io/podio/JEventProcessorPODIO:

```c++
    "ReconstructedElectrons",

```
