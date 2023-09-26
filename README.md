# Argo CD and Release Management with Helm Charts and ApplicationSets

## The Problem 
With Argo CD or OpenShift GitOps it is very easy to deploy Helm Charts and leverage the power of templating. 
However, while working with different cluster environment, one question sometimes arises: 
__How can I make sure that a new version of a Chart is first deployed on the DEV-environment only, before it is installed on the PROD-environment.__

In other words, a new Helm Chart version has been released and it should be tested first before it might have any unwanted side-affects on the production environment. 

For example, let's imagine we have the Chart: "my-super-chart" with the version "1.0.0". It is installed on the dev- and production clusters. 
A new version with some changes has been created, increasing the version number to "1.0.1". Now we have to make sure that this change is first deployed on the dev-cluster only, so it can be verified and if everything is fine, it might be rolled out to production.

The simple solution is to use different **targetRevision** for each cluster in the Application objects of Argo CD. 
But how when there is an ApplicationSet in place? How to configure the targetRevision inside the ApplicationSet? We must make sure that we define the targetRevision for each cluster separately. 

## Prerequisites

The following requisites are mandatory to work with releases:

- Helm Chart Repository. This might be any artifactory, or even GitHub. If you would like to know how to I use GitHub to release my charts, check out the [GitHub actions](https://github.com/tjungbauer/helm-charts/tree/main/.github/workflows) at my repository
- multiple clusters, for example a development cluster (dev) and a production cluster (prod)

## ApplicationSet 

The following ApplicationSet is using the list generator to rollout a configuration (etcd encryption) to multiple clusters. It will create an Application object per cluster in Argo CD.
The clusters are **dev** and **prod**. In addition, it defines the setting **chart_version**. This is the trigger which version will be deployed on which cluster. 
Dev will use the very latest version **1.0.16** while the production environment is still using **1.0.13**. Once the tests have been successful the version for production can be changed to the required number. 

The actual magic happens in the very last line of the ApplicationSet **targetRevision: '{{ chart_version }}'**. This will create the Application objects pointing to the version defined in the list generator. 

```yaml 
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: etcd-encryption
  namespace: openshift-gitops
spec:
  generators:
    - list:
        elements:
          - chart_version: 1.0.16
            cluster: dev
            url: 'https://kubernetes.default.svc'
          - chart_version: 1.0.13
            cluster: prod
            url: 'https://api.ocp.aws.ispworld.at:6443'
  template:
    metadata:
      name: '{{ cluster }}-etcd-encryption'
    spec:
      destination:
        namespace: default
        server: '{{ url }}'
      info:
        - name: Description
          value: Enable Cluster ETCD Encryption
      project: '{{ cluster }}'
      source:
        - chart: generic-cluster-config
          repoURL: 'https://charts.stderr.at/'
          targetRevision: '{{ chart_version }}'
```

## Applications

The snippets of the two Application objects that are created by the ApplicationSet look like: 

### DEV environment 

```yaml
[...]
spec:
  destination:
    namespace: default
    server: 'https://kubernetes.default.svc'
  info:
    - name: Description
      value: Enable Cluster ETCD Encryption
  project: dev
  source:
    - chart: generic-cluster-config
      repoURL: 'https://charts.stderr.at/'
      targetRevision: 1.0.16
```

### PROD environment 
```yaml
[...]
spec:
  destination:
    namespace: default
    server: 'https://api.ocp.aws.ispworld.at:6443'
  info:
    - name: Description
      value: Enable Cluster ETCD Encryption
  project: prod
  source:
    - chart: generic-cluster-config
      repoURL: 'https://charts.stderr.at/'
      targetRevision: 1.0.13
```

## Summary 
With this simple change it is possible to define the Chart version in the ApplicationSet per cluster and make sure that only the wanted version is deployed onto the appropriate cluster. 