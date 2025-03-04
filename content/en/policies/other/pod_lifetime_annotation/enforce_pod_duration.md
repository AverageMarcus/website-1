---
title: "Enforce pod duration"
category: Sample
version: 1.4.2
subject: Pod
policyType: "validate"
description: >
    This validation is valuable when annotations are used to define durations, such as to ensure a Pod lifetime annotation does not exceed some site specific max threshold. Pod lifetime annotation can be no greater than 8 hours.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//other/pod_lifetime_annotation/enforce_pod_duration.yaml" target="-blank">/other/pod_lifetime_annotation/enforce_pod_duration.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: pod-lifetime
  annotations:
    policies.kyverno.io/title: Enforce pod duration
    policies.kyverno.io/category: Sample
    policies.kyverno.io/minversion: 1.4.2
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      This validation is valuable when annotations are used to define durations,
      such as to ensure a Pod lifetime annotation does not exceed some site specific max threshold.
      Pod lifetime annotation can be no greater than 8 hours.
spec:
  validationFailureAction: audit
  background: true
  rules:
  - name: pods-lifetime
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Pod lifetime exceeds limit of 8h"
      deny:
        conditions:
        - key: "{{ request.object.metadata.annotations.\"pod.kubernetes.io/lifetime\" }}"
          operator: DurationGreaterThan
          value: "8h"
```
