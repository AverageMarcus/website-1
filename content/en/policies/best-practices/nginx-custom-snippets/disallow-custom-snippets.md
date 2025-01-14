---
title: "Disallow Custom Snippets"
category: Best Practice
version: 1.4.3
subject: ConfigMap, Ingress
policyType: "validate"
description: >
    Users that can create or update ingress objects can use the custom snippets  feature to obtain all secrets in the cluster (CVE-2021-25742). This policy  disables allow-snippet-annotations in the ingress-nginx configuration and  blocks *-snippet annotations on an Ingress. See: https://github.com/kubernetes/ingress-nginx/issues/7837
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//best-practices/nginx-custom-snippets/disallow-custom-snippets.yaml" target="-blank">/best-practices/nginx-custom-snippets/disallow-custom-snippets.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-ingress-nginx-custom-snippets
  annotations:
    policies.kyverno.io/title: Disallow Custom Snippets
    policies.kyverno.io/category: Best Practice
    policies.kyverno.io/subject: ConfigMap, Ingress
    policies.kyverno.io/minversion: 1.4.3
    policies.kyverno.io/description: >-
      Users that can create or update ingress objects can use the custom snippets 
      feature to obtain all secrets in the cluster (CVE-2021-25742). This policy 
      disables allow-snippet-annotations in the ingress-nginx configuration and 
      blocks *-snippet annotations on an Ingress.
      See: https://github.com/kubernetes/ingress-nginx/issues/7837
spec:
  validationFailureAction: enforce
  rules:
    - name: check-config-map
      message: "ingress-nginx allow-snippet-annotations must be set to false"
      match:
        resources:
          kinds:
            - ConfigMap      
      validate:
        pattern:
          data:
            =(allow-snippet-annotations) : "false"
    - name: check-ingress-annotations
      message: "ingress-nginx custom snippets are not allowed"
      match:
        resources:
          kinds:
            - Ingress      
      validate:
        pattern:
          metadata:
            =(annotations):
              X(*-snippets): "?*"


```
