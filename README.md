QCR
===

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Latest Release](https://img.shields.io/github/release/tliron/qcr.svg)](https://github.com/tliron/qcr/releases/latest)
[![Go Report Card](https://goreportcard.com/badge/github.com/tliron/qcr)](https://goreportcard.com/report/github.com/tliron/qcr)

QCR = Queued Custom Resources

The Problem
-----------

Kubernetes will not let you create a custom resource before its custom resource definition is
registered. This unfortunate limitation makes it impossible for your workloads to be fully
declarative because it requires you to separate your declaration into at least two phases, the
first of which creates the CRDs with the second creating the CRs.


The Solution
------------

QCR works around this limitation by allowing you to queue CRs by embedding them in specially
annotated ConfigMaps. It is an operator that will monitor these ConfigMaps and make sure to create
the CRs as soon as the CRDs are registered. It can at your discretion delete the ConfigMap when
successful or update the CR when the ConfigMap changes.

Because there is always a chance that creating the CR will fail if it does not validate against
the CRD's schema, QCR announces such failures as Kubernetes events in order to make it easy to
see the problem and collect data.

The end result is that you can declare your entire workload all at once, both CRDs and CRs.

Note that for this to work, QCR's service account needs to have RBAC permissions for CRD creation,
thus likely only sysadmins would be able to install it. However, once installed it should make
life easier for non-sysdamins.


Example
-------

Here is an all-at-once CRD and CR:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition

metadata:
  name: resources.qcr.github.com

spec:
  group: qcr.github.com
  names:
    singular: resource
    plural: resources
    kind: Resource
    listKind: ResourceList
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        required: [ spec ]
        properties:
          spec:
            type: object
            properties:
              property1:
                type: string
              property2:
                type: string
              property3:
                type: string

---

apiVersion: v1
kind: ConfigMap

metadata:
  name: hello-world
  annotations:
    qcr.github.com/manifest: hello-world
    qcr.github.com/keep: 'false'

data:
  hello-world: |
    apiVersion: qcr.github.com/v1
    kind: Resource

    metadata:
      name: hello-world
      labels:
        app.kubernetes.io/name: hello-world

    spec:
      property1: value1
      property2: value2
      property3: value3
```
