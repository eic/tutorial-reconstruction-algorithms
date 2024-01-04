---
title: "Implementation"
teaching: 10
exercises: 25
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
        
        \\ Any additional public members go here 

    private:
        std::shared_ptr<spdlog::logger> m_log;
        \\ any additional private members go here

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
        std::shared_ptr<spdlog::logger> m_log;  \\ pointer of logger
        double m_electron{0.000510998928};      \\ electron mass
        double min_energy_over_momentum{0.9};   \\ default minimum E/p
        double max_energy_over_momentum{1.2};   \\ default maximum E/p

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

Next, we will create a factory to call our algorithm and save the output.  Our algorithm requires reconstructed tracks, calorimeter clusters, and associations between the two.  
As previously stated, the current implementation of the simple electron ID uses the truth association between tracks and clusters (association using matching between clusters and track projections will be implemented later).  
Thus, we need two sets of associations: association between the truth particle and the reconstructed charged particle, and association between the truth particle and the calorimeter cluster.
Obviously, we will not create these objects from scratch.  Rather, we will get them from the factories (and underlying algorithms) implemented to create these objects.

### Get tracks

The reconstructed charged particle tracks are stored as a `edm4eic::ReconstructedParticleCollection`.  The collection is accessed by:
 
`auto rc_particles = static_cast<const edm4eic::ReconstructedParticleCollection*>(event->GetCollectionBase("ReconstructedChargedParticles"));`

### Get associations

The truth and reconstructed particle associations are stored as a `edm4eic::MCRecoParticleAssociationCollection`.  The collection is accessed by:

`auto rc_particles_assoc = static_cast<const edm4eic::MCRecoParticleAssociationCollection*>(event->GetCollectionBase("ReconstructedChargedParticleAssociations"));`

Associations between truth particles and calorimeter clusters are stored as a `MCRecoClusterParticleAssociationCollection`.  There is a separate collection for each calorimeter.  
For example, the collection for the backward endcap electromagnetic calorimeter is accessed by:

`mc_cluster_assoc = static_cast<const edm4eic::MCRecoClusterParticleAssociationCollection*>(event->GetCollectionBase("EcalEndcapNClusterAssociations"));`

### Get clusters

We access the calorimeter clusters themselves through the association:

`auto clust = mc_cluster_assoc.getRec();`

### Factory implementation

`src/global/reco/ReconstructedElectrons_factory.h`:

~~~ c++

#pragma once

// JANA
#include "extensions/jana/JChainMultifactoryT.h"
#include <JANA/JEvent.h>

// algorithms
#include "algorithms/reco/ElectronReconstruction.h"

// services
#include "extensions/spdlog/SpdlogExtensions.h"
#include "extensions/spdlog/SpdlogMixin.h"

namespace eicrecon {

  class ReconstructedElectrons_factory :
    public JChainMultifactoryT<NoConfig>,
    public SpdlogMixin
  {

    public:

      explicit ReconstructedElectrons_factory(
          std::string tag,
          const std::vector<std::string>& input_tags,
          const std::vector<std::string>& output_tags)
      : JChainMultifactoryT<NoConfig>(std::move(tag), input_tags, output_tags) {
          DeclarePodioOutput<edm4eic::ReconstructedParticle>(GetOutputTags()[0]);
      }

      /** One time initialization **/
      void Init() override {
        // get plugin name and tag
        auto app    = GetApplication();
        auto plugin = GetPluginName();
        auto prefix = plugin + ":" + GetTag();

        // services
        InitLogger(app, prefix, "info");
        m_algo.init(m_log);
      }

      /** Event by event processing **/
      void Process(const std::shared_ptr<const JEvent> &event) override{
        // Step 1. lets collect the Cluster associations from various detectors
        std::vector<const edm4eic::MCRecoClusterParticleAssociationCollection*> in_clu_assoc;
        for(auto& input_tag : GetInputTags()){
          // only collect from the sources that provide ClusterAssociations
          if ( input_tag.find( "ClusterAssociations" ) == std::string::npos ) {
            continue;
          }
          m_log->trace( "Adding cluster associations from: {}", input_tag );
          in_clu_assoc.push_back(
            static_cast<const edm4eic::MCRecoClusterParticleAssociationCollection*>(event->GetCollectionBase(input_tag))
          );
        }

        // Step 2. Get MC, RC, and MC-RC association info
        // This is needed as a bridge to get RecoCluster - RC Particle associations

        auto mc_particles = static_cast<const edm4hep::MCParticleCollection*>(event->GetCollectionBase("MCParticles"));
        auto rc_particles = static_cast<const edm4eic::ReconstructedParticleCollection*>(event->GetCollectionBase("ReconstructedChargedParticles"));
        auto rc_particles_assoc = static_cast<const edm4eic::MCRecoParticleAssociationCollection*>(event->GetCollectionBase("ReconstructedChargedParticleAssociations"));

        // Step 3. Pass everything to "the algorithm"
        // in the future, select appropriate algorithm (truth, fully reco, etc.)
        auto output = m_algo.execute(
          mc_particles,
          rc_particles,
          rc_particles_assoc,
          in_clu_assoc
        );

        m_log->debug( "We have found {} reconstructed electron candidates this event", output->size() );
        // Step 4. Output the collection
        SetCollection<edm4eic::ReconstructedParticle>(GetOutputTags()[0], std::move(output));
      }

    private:

      // underlying algorithm
      eicrecon::ElectronReconstruction m_algo;
  };
}

~~~

















