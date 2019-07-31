# Design doc - Support for Kustomize templates

Spinnaker surfaces a "Bake (Manifest)" stage to turn templates into manifests with the help of a templating engine. Currently, the only supported templating engine is Helm. We propose adding support for [Kustomize](https://github.com/kubernetes-sigs/kustomize) as an alternative templating engine to Helm.

## Introduction
kustomize is a command line tool supporting template-free, structured customization of declarative configuration targetted to k8s-style objects.

Targetted to k8s means that kustomize has some understanding of API resources, k8s concepts like names, labels, namespaces, etc. and the semantics of resource patching.

kustomize has seen rapid growth in adoption in the last 12 months, helped in no small part by it's merging into `kubectl`. It's pure YAML declarative approach to application management is more suite to configuration management as opposed to the `package management` approach taken by Helm. This makes it suited to smaller, less complex use cases where for example, configuration needs to change on a per environment basis.

## Motivation
With it's inclusion into `kubectl` kustomize applications are likely to have a permanent place in the Kubernetes ecosystem, with many users utilizing it for config management. Including kustomize into Spinnaker as a first class templating engine for the "Bake (Manifest)" stage would enable a wide variety of use cases, and simplify pipelines for anyone not wanting to use Helm for templating.

## Alternatives
### Templating languages
- Helm: already supported!
- Jsonnet/Ksonnet: Non-Yaml approach & Closed project, neither with the base in the K8s community of Helm or kustomize.

### Running Kustomize
#### Using a Jenkins job
If is possible at the moment to call out to Jenkins to run kustomize. Manifests can be sent from a step (Jenkins step or Script step) to a Jenkins job where kustomize is executed and the resulting manifests returned to Spinnaker. This is the current solution many teams use. However, this is a convoluted and complected solution to achieve a simple goal: preparing (baking) k8s manifests using a template engine (kustomize).

## Functional design
The functional design will follow very closely the design of the current `HELM2` implementation in the "Bake (Manifest)" stage. The existing `HELM2` functionality will not be altered or effected by the inclusion of kustomize. In use the kustomize template engine will operate in a very similar way to that of `HELM2` with only a few differences in options available. The following is a proposed UI design for the "Bake (Manifest)" stage with kustomize:

![Proposed UI with kustomize](https://github.com/andrewcamerondouglas/spinnaker-kustomize-proposal/raw/master/proposed-ui.png)

The main UI changes would be:
- Add `Kustomize` as an option in "Render engine"
- When `Kustomize` is selected hide the "Namespace" input
- When `Kustomize` is selected show a "Kustomization" input (which is required)
- When `Kustomize` is selected hide the "Overrides" section and all inputs
- (When `Kustomize` is selected potentially the "Name" input could also be hidden.)

Users select `Kustomize` from the "Render engine" drop down and then enter the path to a "Kustomization" directory in the imput. When the pipeline is run the kustomize is executed using the command `kustomize build $kustomization`. The "Kustomization" directory is a base or overlay containing a `kustomise.yaml` file (as is the standard for kustomize - see https://github.com/kubernetes-sigs/kustomize/blob/master/README.md). As with templating using Helm in the “Produces Artifacts” section, Spinnaker has automatically created an embedded/base64 artifact that is bound when the stage completes, representing the fully baked manifest set to be deployed downstream.

If developers are programmatically generating stages, the JSON representation of the same stage would look like the following:

```
{
  "type": "bakeManifest",
  "templateRenderer": "Kustomize",
  "name": "Kustomize template",
  "kustomization": "overlays/staging",
  "inputArtifacts": [
    {
      "account": "gh-artifact",
      "id": "gh-artifact"
    }
  ],
  "expectedArtifacts": [
    {
      "defaultArtifact": {},
      "id": "baked-template",
      "matchArtifact": {
        "kind": "base64",
        "name": "hello-world",
        "type": "embedded/base64"
      },
      "useDefaultArtifact": false
    }
  ]
}
```

## Technical plan
### Prerequisites
There are two options for running kustomize: via `kubectl` (>=1.14) or via the standalone `kustomize` app. For flexibility, and as the primary role of Kustomise in Spinnaker is to bake/template manifests, it has been decided that the standalone `kustomize` cli will be used. This will need to be installed in the container running Rosco.

### Proposed changes
The following changes are proposed, where possible existing functionality and code should be left untouched.

#### Rosco
https://github.com/spinnaker/rosco/tree/master/rosco-manifests/src/main/java/com/netflix/spinnaker/rosco/manifests

- Add kustomize folder with:
  -  `KustomizeBakeManifestRequest.java` 
     -  Defines a String `kustomization`
  -  `KustomizeBakeManifestService.java`
  -  `KustomizeTemplateUtils.java`
     -  Call `kustomize build <kustomization>`
- Amend `BakeManifestRequest.java`
  - Add `Kustomize`

https://github.com/spinnaker/rosco/tree/master/rosco-web/src/main/groovy/com/netflix/spinnaker/rosco/controllers

- Amend `V2BakeryController.java`
  - Add `KustomizeBakeManifestService`

https://github.com/spinnaker/rosco/blob/master/rosco-web/pkg_scripts

- Amend `postInstall.sh`
  - Add `install_kustomize()`

https://github.com/spinnaker/rosco/blob/master

- Amend `Dockerfile`
  - Add step to install Kustomize
- Amend `Dockerfile.slim`
  - Add step to install Kustomize


#### Orca
https://github.com/spinnaker/orca/blob/master/orca-bakery/src/main/groovy/com/netflix/spinnaker/orca/bakery/tasks/manifests

- Amend `CreateBakeManifestTask.java`
  - Add `KustomizeBakeManifestRequest`
  - Add switches to support both Helm and Kustomize
- Amend `BakeManifestContext.java`
  - Add `kustomization`

https://github.com/spinnaker/orca/tree/master/orca-bakery/src/main/groovy/com/netflix/spinnaker/orca/bakery/api/manifests

- Add kustomize folder with:
  - `KustomizeBakeManifestRequest.java`
    - Defines a String `kustomization`
    - (and `outputArtifactName`) - same as helm

- Possible test updates
  - https://github.com/spinnaker/orca/blob/master/orca-bakery/src/test/groovy/com/netflix/spinnaker/orca/bakery/BakerySelectorSpec.groovy
  - https://github.com/spinnaker/orca/blob/master/orca-front50/src/test/groovy/com/netflix/spinnaker/orca/front50/DependentPipelineStarterSpec.groovy


#### Deck
https://github.com/spinnaker/deck/tree/master/app/scripts/modules/core/src/pipeline/config/stages/bakeManifest

- Amend `BakeManifestStageForm.tsx`
  - Add `Kustomize` as a templateRenderer
  - When templateRenderer is `HELM2` show `namespace` input
  - When templateRenderer is `HELM2` hide `kustomisation` input
  - When templateRenderer is `Kustomize` show `kustomisation` input
  - When templateRenderer is `Kustomize` hide `namespace` input
  - When templateRenderer is `Kustomize` hide `Overrides` section


#### spinnaker.github.io
https://github.com/spinnaker/spinnaker.github.io/tree/master/guides/user/kubernetes-v2

- Add `deploy-kustomize` folder
  - With guide
  - Use https://www.spinnaker.io/guides/user/kubernetes-v2/deploy-helm/ to help

### Testing
***Help needed here to identify where tests should be added***

- Possible test updates
  - https://github.com/spinnaker/orca/blob/master/orca-bakery/src/test/groovy/com/netflix/spinnaker/orca/bakery/BakerySelectorSpec.groovy
  - https://github.com/spinnaker/orca/blob/master/orca-front50/src/test/groovy/com/netflix/spinnaker/orca/front50/DependentPipelineStarterSpec.groovy


## Timescales
After obtaining consensus on whether this change should be included in Spinnaker, and if the approach outlined here is sensible, it is expected that adding kustomize will take around two weeks.
