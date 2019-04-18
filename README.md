# Manifests
This repo is a [bespoke configuration](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/glossary.md#bespoke-configuration) of kustomize targets used by kubeflow. These targets are traversed by kubeflow's CLI `kfctl`. Each target is compatible with the kustomize CLI and can be processed indendently by kubectl or the kustomize command. 

## Organization
Various subdirectories within the repo contain a kustomize target (base or overlay subdirectory). Currently overlays hold platform resources where platform is one of "gcp|minikube". These targets are processed by kfctl during generate and apply phases and is detailed in [Kfctl Processing](#kfctl-processing). 


### Kustomize targets (🎯)
```
📦 application   ⇲ 🗳 base
📦 argo          ⇲ 🗳 base
📦 common        ⇲
                 ⎹→🗳 ambassador/🎯base
                 ⎹→🗳 basic-auth/🎯base
                 ⎹→🗳 centraldashboard/🎯base
                 ⎹→🗳 echo-server/🎯base
                 ⎹→🗳 spartakus/🎯base
📦 gcp           ⇲                                   
                 ⎹→🗳 cert-manager/overlay/🎯gcp
                 ⎹→🗳 cloud-endpoints/overlay/🎯gcp
                 ⎹→🗳 gcp-credentials-admission-webhook/overlay/🎯gcp
                 ⎹→🗳 gpu-driver/overlay/🎯gcp
                 ⎹→🗳 iap-ingress/overlay/🎯gcp
                 ⎹→🗳 metric-collector/overlay/🎯gcp
                 ⎹→🗳 prometheus/overlay/🎯gcp
📦 jupyter        ⇲                                   
                 ⎹→🗳 jupyter/🎯base
                 ⎹→🗳 jupyter-web-app/🎯base
                 ⎹→🗳 notebook-controller/🎯base
📦 katib         ⇲                                   
📦 kubebench     ⇲ 🗳 base
📦 metacontroller⇲ 🗳 base
📦 pipeline      ⇲ 
                 ⎹→🗳 minio/🎯base
                 ⎹→🗳 mysql/🎯base
                 ⎹→🗳 persistent-agent/🎯base
                 ⎹→🗳 pipelines-runner/🎯base
                 ⎹→🗳 pipelines-ui/🎯base
                 ⎹→🗳 pipelines-viewer/🎯base
                 ⎹→🗳 scheduledworkflow/🎯base
```

## Kfctl Processing 
Kfctl will traverse these directories to find and build kustomize targets based on the configuration file `app.yaml`. App.yaml is derived from a file in the kubeflow [config](https://github.com/kubeflow/kubeflow/tree/master/bootstrap/config) directory. Each target processed by kfctl will result in an output yaml file. The output file is generated by calling kustomize's API.  The kustomize package manager in kfctl will read app.yaml and apply the packages, components and componentParams to kustomize in the following way:

- **📦 packages** 
  - are always top-level directories under the manifests repo
- **🗳 components** 
  - are also directories but may be a subdirectory in a package.
  - components may also be a top-level directory if there is a base or overlay in that directory in which case the component name is equal to the package name. 
  - otherwise a component is a sub-directory 
  - in all cases a component's name in app.yaml must match the directory name.
  - components are output as `<component>.yaml` under the kustomize subdirectory during `kfctl generate...`. 
  - in order to output a component a synthetic kustomization.yaml is created under manifests that sets common parameters and labels that looks like 
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - <component>/{base|overlay/<platform>}
commonLabels:
  app.kubernetes.io/name: <appName>
namespace:
  <namespace>
```
- **component parameters** 
  - are applied to a component's params.env file. There must be an entry whose key matches the component parameter. The params.env file is used to generate a ConfigMap. Entries in params.env are resolved as kustomize vars or referenced in a deployment or statefulset env section in which case no var definition is needed.


Generating yaml output for any target can be done in the following way:

### Install kustomize

`go get -u github.com/kubernetes-sigs/kustomize`

### Run kustomize

#### Example

```bash
git clone https://github.com/kubeflow/manifests
cd manifests/<target>/base
kustomize build | tee <output file>
```

Kustomize inputs to kfctl based on app.yaml which is derived from files under config/ such as [kfctl_default.yaml](https://github.com/kubeflow/kubeflow/blob/master/bootstrap/config/kfctl_default.yaml)):

```
apiVersion: kfdef.apps.kubeflow.org/v1alpha1
kind: KfDef
metadata:
  creationTimestamp: null
  name: kubeflow
  namespace: kubeflow
spec:
  appdir: /Users/kdkasrav/kubeflow
  componentParams:
    ambassador:
    - name: ambassadorServiceType
      value: NodePort
  components:
  - metacontroller
  - ambassador
  - argo
  - centraldashboard
  - jupyter-web-app
  - katib
  - notebook-controller
  - pipeline
  - profiles
  - pytorch-operator
  - tensorboard
  - tf-job-operator
  - application
  manifestsRepo: /Users/kdkasrav/kubeflow/.cache/manifests/pull/13/head
  packageManager: kustomize@pull/13
  packages:
  - application
  - argo
  - common
  - examples
  - gcp
  - jupyter
  - katib
  - metacontroller
  - modeldb
  - mpi-job
  - pipeline
  - profiles
  - pytorch-job
  - seldon
  - tensorboard
  - tf-serving
  - tf-training
  repo: /Users/kdkasrav/kubeflow/.cache/kubeflow/pull/2971/head/kubeflow
  useBasicAuth: false
  useIstio: false
  version: pull/2971
```

Outputs from kfctl (no platform specified):
```
<deployment>  ⇲
              ⎹→kustomize
                        ⎹→ambassador.yaml
                        ⎹→application.yaml
                        ⎹→argo.yaml
                        ⎹→centraldashboard.yaml
                        ⎹→jupyter-web-app.yaml
                        ⎹→metacontroller.yaml
                        ⎹→notebook-controller.yaml
                        ⎹→pipeline.yaml
                        ⎹→profiles.yaml
                        ⎹→pytorch-operator.yaml
                        ⎹→tensorboard.yaml
                        ⎹→tf-job-operator.yaml
```

## Best practices for kustomize targets

- use name prefixes if possible for the set of resources bundled by a target
- do not set namespace in the resources, this should be done by a higher level target


### Bridging kustomize and ksonnet

Equivalent to parameters in ksonnet, kustomize has vars. But the customizable objects are limited to [this list](https://github.com/kubernetes-sigs/kustomize/blob/master/pkg/transformers/config/defaultconfig/varreference.go)

### Installing to a custom namespace

For example, to install in `kubeflow-dev`. From the root of the repo run:

```bash
kustomize edit set namespace kubeflow-dev
```
