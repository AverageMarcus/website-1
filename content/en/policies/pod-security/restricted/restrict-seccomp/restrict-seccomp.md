---
title: "Restrict Seccomp"
category: Pod Security Standards (Restricted)
version: 
subject: Pod
policyType: "validate"
description: >
    The runtime default seccomp profile must be required, or only specific additional profiles should be allowed. This policy, requiring Kubernetes v1.19 or later, ensures that only the `RuntimeDefault` or `Localhost` is used as a `type` and that it is not unset.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//pod-security/restricted/restrict-seccomp/restrict-seccomp.yaml" target="-blank">/pod-security/restricted/restrict-seccomp/restrict-seccomp.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-seccomp
  annotations:
    policies.kyverno.io/title: Restrict Seccomp
    policies.kyverno.io/category: Pod Security Standards (Restricted)
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      The runtime default seccomp profile must be required, or only specific
      additional profiles should be allowed. This policy, requiring Kubernetes
      v1.19 or later, ensures that only the `RuntimeDefault` or `Localhost` is
      used as a `type` and that it is not unset.
spec:
  background: true
  validationFailureAction: enforce
  rules:
  - name: restrict-seccomp
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: >-
        Use of custom Seccomp profiles is disallowed. The fields
        spec.securityContext.seccompProfile.type,
        spec.containers[*].securityContext.seccompProfile.type, and
        spec.initContainers[*].securityContext.seccompProfile.type
        must be set to `RuntimeDefault` or `Localhost`.
      anyPattern:
      - spec:
          securityContext:
            seccompProfile:
              type: "RuntimeDefault | Localhost"
      - spec:
          containers:
          - securityContext:
              seccompProfile:
                type: "RuntimeDefault | Localhost"
  - name: restrict-seccomp-initcontainers
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: >-
        Use of custom Seccomp profiles is disallowed. The fields
        spec.securityContext.seccompProfile.type,
        spec.containers[*].securityContext.seccompProfile.type, and
        spec.initContainers[*].securityContext.seccompProfile.type
        must be set to `RuntimeDefault` or `Localhost`.
      anyPattern:
      - spec:
          securityContext:
            seccompProfile:
              type: "RuntimeDefault | Localhost"
      - spec:
          =(initContainers):
          - securityContext:
              seccompProfile:
                type: "RuntimeDefault | Localhost"

```
