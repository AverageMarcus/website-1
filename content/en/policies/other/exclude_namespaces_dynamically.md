---
title: "Exclude Namespaces Dynamically"
category: Sample
version: 
subject: Namespace, Pod
policyType: "validate"
description: >
    It's common where policy lookups need to consider a mapping to many possible values rather than a static mapping. This is a sample which demonstrates how to dynamically look up an allow list of Namespaces from a ConfigMap where the ConfigMap stores an array of strings. This policy validates that any Pods created outside of the list of Namespaces have the label `foo` applied.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//other/exclude_namespaces_dynamically.yaml" target="-blank">/other/exclude_namespaces_dynamically.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: exclude-namespaces-example
  annotations:
    policies.kyverno.io/title: Exclude Namespaces Dynamically
    policies.kyverno.io/category: Sample
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Namespace, Pod
    policies.kyverno.io/description: >-
      It's common where policy lookups need to consider a mapping to many possible values rather than a
      static mapping. This is a sample which demonstrates how to dynamically look up an allow list of Namespaces from a ConfigMap
      where the ConfigMap stores an array of strings. This policy validates that any Pods created
      outside of the list of Namespaces have the label `foo` applied.
spec:
  validationFailureAction: audit
  background: true
  rules:
  - name: exclude-namespaces-dynamically
    context:
      - name: namespacefilters
        # The source ConfigMap should contain an array of strings in either YAML block scalars
        # (Kyverno 1.3.5+) or JSON-encoded format.
        configMap:
          name: namespace-filters
          namespace: default
    match:
      resources:
        kinds:
        - Pod
    preconditions:
    - key: "{{request.object.metadata.namespace}}"
      operator: NotIn
      value: "{{namespacefilters.data.exclude}}"
    validate:
      message: >
        Creating Pods in the {{request.object.metadata.namespace}} namespace,
        which is not in the excluded list of namespaces {{ namespacefilters.data.exclude }},
        is forbidden unless it carries the label `foo`.
      pattern:
        metadata:
          labels:
            foo: "*"
```
