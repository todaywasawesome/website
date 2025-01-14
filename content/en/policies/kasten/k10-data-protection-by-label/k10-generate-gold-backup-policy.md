---
title: "Generate Gold Backup Policy"
category: Kasten K10 by Veeam
version: 1.6.2
subject: Policy
policyType: "generate"
description: >
    Generate a backup policy for any Deployment or StatefulSet that adds the labels "dataprotection: k10-goldpolicy" This policy works best to decide the data protection objectives and simply assign backup via application labels.
---

## Policy Definition
<a href="https://github.com/kyverno/policies/raw/main//kasten/k10-data-protection-by-label/k10-generate-gold-backup-policy.yaml" target="-blank">/kasten/k10-data-protection-by-label/k10-generate-gold-backup-policy.yaml</a>

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: k10-generate-gold-backup-policy
  annotations:
    policies.kyverno.io/title: Generate Gold Backup Policy
    policies.kyverno.io/category: Kasten K10 by Veeam
    kyverno.io/kyverno-version: 1.6.2
    policies.kyverno.io/minversion: 1.6.2
    kyverno.io/kubernetes-version: "1.21-1.22"
    policies.kyverno.io/subject: Policy
    policies.kyverno.io/description: >-
      Generate a backup policy for any Deployment or StatefulSet that adds the labels "dataprotection: k10-goldpolicy"
      This policy works best to decide the data protection objectives and simply assign backup via application labels.
spec:
  background: false
  rules:
  - name: k10-generate-gold-backup-policy
    match:
      any:
      - resources:
          kinds:
            - Deployment
            - StatefulSet
          selector:
            matchLabels:
              dataprotection: k10-goldpolicy # match with a corresponding ClusterPolicy that checks for this label
    generate:
      apiVersion: config.kio.kasten.io/v1alpha1
      kind: Policy
      name: k10-{{request.namespace}}-gold-backup-policy
      namespace: "{{request.namespace}}"
      data:   
        metadata: 
          name: k10-{{request.namespace}}-gold-backup-policy
          namespace: "{{request.namespace}}"
        spec:
          comment: K10 "gold" immutable production backup policy
          frequency: '@daily'
          retention:
            daily: 7
            weekly: 4
            monthly: 12
            yearly: 7
          actions:
          - action: backup
          - action: export
            exportParameters:
              frequency: '@monthly'
              profile:
                name: object-lock-s3
                namespace: kasten-io
              exportData:
                enabled: true
            retention:
              monthly: 12
              yearly: 5
          selector:
            matchExpressions:
              - key: k10.kasten.io/appNamespace
                operator: In
                values:
                  - "{{request.namespace}}"

```
