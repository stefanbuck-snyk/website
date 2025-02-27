---
title: "Block Pod Exec by Pod Name"
category: Sample
version: 1.4.2
subject: Pod
policyType: "validate"
description: >
    The `exec` command may be used to gain shell access, or run other commands, in a Pod's container. While this can be useful for troubleshooting purposes, it could represent an attack vector and is discouraged. This policy blocks Pod exec commands to Pods beginning with the name `myapp-maintenance-`.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//other/block-pod-exec-by-pod-name.yaml" target="-blank">/other/block-pod-exec-by-pod-name.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: deny-exec-by-pod-name
  annotations:
    policies.kyverno.io/title: Block Pod Exec by Pod Name
    policies.kyverno.io/category: Sample
    policies.kyverno.io/minversion: 1.4.2
    policies.kyverno.io/subject: Pod
    policies.kyverno.io/description: >-
      The `exec` command may be used to gain shell access, or run other commands, in a Pod's container. While this can
      be useful for troubleshooting purposes, it could represent an attack vector and is discouraged.
      This policy blocks Pod exec commands to Pods beginning with the name
      `myapp-maintenance-`.
spec:
  validationFailureAction: enforce
  background: false
  rules:
  - name: deny-exec-myapp-maintenance
    match:
      resources:
        kinds:
        - PodExecOptions
    preconditions:
      all:
      - key: "{{ request.operation || 'BACKGROUND' }}"
        operator: Equals
        value: CONNECT
    validate:
      message: Exec'ing into Pods called "myapp-maintenance" is not allowed.
      deny:
        conditions:
          all:
          - key: "{{ request.name }}"
            operator: Equals
            value: myapp-maintenance-*

```
