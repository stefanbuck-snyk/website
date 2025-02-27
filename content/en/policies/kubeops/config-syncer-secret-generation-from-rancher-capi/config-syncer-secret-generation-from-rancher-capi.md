---
title: "Kubeops Config Syncer Secret Generation From Rancher CAPI Secret"
category: Kubeops
version: 1.7.1
subject: Secret
policyType: "generate"
description: >
    This policy generates and synchronizes a Kubeops Config Syncer merged kubeconfig Secret from Rancher managed cluster CAPI secrets. This kubeconfig Secret is required by the Kubeops Config Syncer for it to sync ConfigMaps/Secrets from the Rancher management cluster to downstream clusters.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//kubeops/config-syncer-secret-generation-from-rancher-capi/config-syncer-secret-generation-from-rancher-capi.yaml" target="-blank">/kubeops/config-syncer-secret-generation-from-rancher-capi/config-syncer-secret-generation-from-rancher-capi.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: config-syncer-secret-generation-from-rancher-capi
  annotations:
    policies.kyverno.io/title: Kubeops Config Syncer Secret Generation From Rancher CAPI Secret
    policies.kyverno.io/category: Kubeops
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Secret
    kyverno.io/kyverno-version: 1.8.0
    policies.kyverno.io/minversion: 1.7.1
    kyverno.io/kubernetes-version: "1.23"
    policies.kyverno.io/description: >-
      This policy generates and synchronizes a Kubeops Config Syncer merged kubeconfig
      Secret from Rancher managed cluster CAPI secrets. This kubeconfig Secret is
      required by the Kubeops Config Syncer for it to sync ConfigMaps/Secrets from
      the Rancher management cluster to downstream clusters.
spec:
  generateExistingOnPolicyUpdate: true
  rules:
  - name: source-rancher-non-local-cluster-and-capi-secret
    match:
      all:
      - resources:
          kinds:
          - provisioning.cattle.io/v1/Cluster
    exclude:
      any:
      - resources:
          namespaces:
          - fleet-local
    context:
    - name: secretList
      apiCall:
        urlPath: "/api/v1/namespaces/{{request.object.metadata.namespace}}/secrets"
        jmesPath: "items[?metadata.name.contains(@, '-kubeconfig')]"
    - name: kubeconfigClustersData
      variable:
        value: |
          {{ secretList | [].{
              "name": metadata.name,
              "cluster": data.value | base64_decode(@) | parse_yaml(@).clusters[].cluster[] | [0] }
          }}
        jmesPath: 'to_string(@)'
    - name: kubeconfigUsersData
      variable:
        value: |
          {{ secretList | [].{
              "name": metadata.name,
              "user": data.value | base64_decode(@) | parse_yaml(@).users[].user[] | [0] }
          }}
        jmesPath: 'to_string(@)'
#    - name: kubeconfigContextsData # Enable when Kyverno variable substitution bug is resolved
#      variable:
#        value: |
#          {{ secretList | [].{
#              "name": metadata.name,
#              "context": { "cluster": metadata.name, "user": metadata.name } }
#          }}
#        jmesPath: 'to_string(@)'
    - name: kubeconfigContextsData # Use until Kyverno variable substitition bug is resolved
      variable:
        value: |
          {{ secretList | [].{
              "name": metadata.name,
              "context": ['CLUSTER_BEGIN', metadata.name, 'CLUSTER_END', 'USER_BEGIN', metadata.name, 'USER_END'] }
          }}
        jmesPath: "to_string(@) | replace_all(@, '[\"CLUSTER_BEGIN\",', '{\"cluster\":') | replace_all(@, '\"CLUSTER_END\",\"USER_BEGIN\",', '\"user\":') | replace_all(@, ',\"USER_END\"]', '}')"
    - name: kubeconfigData
      variable:
        value: |
          {
            "apiVersion": "v1",
            "kind": "Config",
            "clusters": {{ kubeconfigClustersData }},
            "users": {{ kubeconfigUsersData }},
            "contexts": {{ kubeconfigContextsData }}
          }
        jmesPath: 'to_string(@)'
    generate:
      synchronize: true
      apiVersion: v1
      kind: Secret
      name: kubed
      namespace: kube-system
      data:
        type: Opaque
        data:
          kubeconfig: "{{ kubeconfigData | base64_encode(@) }}"

```
