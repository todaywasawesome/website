---
title: "Argo Cluster Secret Generation From Rancher CAPI Secret"
category: Argo
version: 1.7.0
subject: Secret
policyType: "generate"
description: >
    This policy generates and synchronizes Argo CD cluster secrets from Rancher  managed cluster CAPI secrets. In this solution, Argo CD integrates with Rancher managed clusters via the central Rancher authentication proxy which shares the network endpoint of the Rancher API/GUI. The policy implements work-arounds for Argo CD issue https://github.com/argoproj/argo-cd/issues/9033 "Cluster-API cluster  auto-registration" and Rancher issue https://github.com/rancher/rancher/issues/38053 "Fix type and labels Rancher v2 provisioner specifies when creating CAPI Cluster Secret".
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//argo/argo-cluster-generation-from-rancher-capi/argo-cluster-generation-from-rancher-capi.yaml" target="-blank">/argo/argo-cluster-generation-from-rancher-capi/argo-cluster-generation-from-rancher-capi.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: argo-cluster-generation-from-rancher-capi
  annotations:
    policies.kyverno.io/title: Argo Cluster Secret Generation From Rancher CAPI Secret
    policies.kyverno.io/category: Argo
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Secret
    kyverno.io/kyverno-version: 1.7.1
    policies.kyverno.io/minversion: 1.7.0
    kyverno.io/kubernetes-version: "1.23"
    policies.kyverno.io/description: >-
      This policy generates and synchronizes Argo CD cluster secrets from Rancher 
      managed cluster CAPI secrets. In this solution, Argo CD integrates with Rancher
      managed clusters via the central Rancher authentication proxy which shares
      the network endpoint of the Rancher API/GUI. The policy implements work-arounds
      for Argo CD issue https://github.com/argoproj/argo-cd/issues/9033 "Cluster-API cluster 
      auto-registration" and Rancher issue https://github.com/rancher/rancher/issues/38053
      "Fix type and labels Rancher v2 provisioner specifies when creating CAPI Cluster Secret".
spec:
  generateExistingOnPolicyUpdate: true
  rules:
    - name: source-rancher-non-local-capi-secret
      match:
        all:
        - resources:
            kinds:
            - Secret
            names:
            - "*-kubeconfig"
# BEGIN future solution blocked by Rancher issue: https://github.com/rancher/rancher/issues/38053
#            selector:
#              matchExpressions:
#              - key: "cluster.x-k8s.io/cluster-name"
#                operator: Exists
# END future solution blocked by above Rancher issue
# BEGIN interim solution while waiting on resolution of above Rancher issue
            namespaces:
            - "fleet-*"
            annotations:
              objectset.rio.cattle.io/owner-gvk: provisioning.cattle.io/v1, Kind=Cluster
      exclude:
        any:
        - resources:
            namespaces:
            - fleet-local
# END interim solution while waiting on resolution of above Rancher issue
      context:
      - name: clusterName
        variable:
          value: "{{ replace_all('{{request.object.metadata.name}}', '-kubeconfig', '') }}"
          jmesPath: 'to_string(@)'
      - name: clusterPrefixedName
        variable:
          value: "{{ join('-', ['cluster', clusterName]) }}"
          jmesPath: 'to_string(@)'
      - name: extraLabels
        variable:
          value:
            argocd.argoproj.io/secret-type: cluster
            clusterId: "{{ clusterName }}"
      - name: metadataLabels
        apiCall:
          urlPath: "/apis/provisioning.cattle.io/v1/namespaces/{{request.object.metadata.namespace}}/clusters/{{clusterName}}"
          jmesPath: "metadata.labels"
      - name: metadataLabels
        variable:
          jmesPath: merge(metadataLabels, extraLabels)
      - name: serverName
        variable:
          value: "{{ request.object.data.value | base64_decode(@) | parse_yaml(@).clusters[0].cluster.server }}"
          jmesPath: 'to_string(@)'
      - name: bearerToken
        variable:
          value: "{{ request.object.data.token | base64_decode(@) }}"
          jmesPath: 'to_string(@)'
      - name: caData
        variable:
          value: "{{ request.object.data.value | base64_decode(@) | parse_yaml(@).clusters[0].cluster.\"certificate-authority-data\" }}"
          jmesPath: 'to_string(@)'
      - name: dataConfig
        variable:
          value: |
            {
              "bearerToken": "{{ bearerToken }}",
              "tlsClientConfig": {
                "insecure": false,
                "caData": "{{ caData }}"
              }
            }
          jmesPath: 'to_string(@)'
      generate:
        synchronize: true
        apiVersion: v1
        kind: Secret
        name: "{{ clusterPrefixedName }}"
        namespace: argocd
        data:
          metadata:
            labels:
              "{{ metadataLabels }}"
          type: Opaque
          data:
            name: "{{ clusterPrefixedName | base64_encode(@) }}"
            server: "{{ serverName | base64_encode(@) }}"
            config: "{{ dataConfig | base64_encode(@) }}"

```
