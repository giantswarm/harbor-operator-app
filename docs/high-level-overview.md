# Harbor operator app overview

## Purpose

The purpose of this operator is to deploy harbor-operator and its related components through the use of helm. By doing this we have been able to intergrate the deployment into both flux and giantswarm apps. Harbor itself is an image repository composed of many different parts which you can see [here](https://github.com/goharbor/harbor/wiki/Architecture-Overview-of-Harbor). For internal use harbor can be used as way to backup and pull images when other repositories fail/go down. Harbor can also be used as offering to clients.

## How does this project work?

For a more comprehensive look at harbor it is recommended that you check out the [harbor-github-page](https://github.com/goharbor/harbor-operator). Essentially, harbor comes with different deployment components which you can read about in the link in the purpose section. Of these components core and registry are probably the most important. Core is responisble for most of the functionality that harbor excutes meaning routing of API calls and stroing of general configuration. Registry is responisble for hosting where images are acutally stored and could be configured to a variety of different providers. In most cases this will be Amazons S3.

## Using Harbor as a proxy cache

The primary way that we've agreed to use harbor is as a proxy cache, meaning it is just a means to pull images through. We've achieved this by creating containerd configuration for workload clusters which sets harbor as mirror registry and falls back to docker if it fails. This means that when you pull images through a workload cluster they should always attempt to use harbor first. In order to make this work for shortnames we had to set up an nginx proxy which rewrites urls from `giantswarm` to `giantswarm/giantswarm` due to the fact that harbor stores images as `projects/namespace`. This ingress is deployed as part of the [management-cluster-fleet](https://github.com/giantswarm/management-clusters-fleet/)

## Deployment

The operator can be deployed through the `management-clusters-fleet` repository. To do this it is important to provide a `kustomisation.yaml` and a `configmap` in the directory of the relevant cluster as this will subsitute in values such as the harbor url and the method of storage. 

It can also be deployed on management clusters throguh the use `app-custom-resources.yaml` such as:

```
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
  name: harbor-operator-user-values
  namespace: harbor-operator
data:
  values: |
    harbor-operator:
      installCRDs: true
      redis-operator:
        enabled: true
      minio-operator:
        enabled: false
      postgres-operator:
        enabled: true
        image:
          tag: v1.8.2
        configGeneral:
          docker_image: registry.opensource.zalan.do/acid/spilo-14:2.1-p7
        configKubernetes:
          secret_name_template: "{username}.{cluster}.credentials"
          enable_pod_antiaffinity: "true"
          pod_environment_configmap: "harbor-operator/pod-config"
          pod_environment_secret: "pod-secret"
        configAwsOrGcp:
          aws_region: us-east-1
          wal_s3_bucket: harbor-postgres-operator
        backup:
            enabled: true
            number_of_backups_to_retain: 5
            backup_schedule: "30 * * * *"
            bucket: "harbor-postgres-operator"
            aws:
              region: "us-east-1"
        restore:
          enabled: false
---
apiVersion: application.giantswarm.io/v1alpha1
kind: App
metadata:
  labels:
    app-operator.giantswarm.io/version: 0.0.0
    app.kubernetes.io/name: harbor-operator
  name: harbor-operator
  namespace: giantswarm
spec:
  catalog: giantswarm-playground-test
  kubeConfig:
    inCluster: true
  name: harbor-operator-app
  namespace: harbor-operator
  userConfig:
    configMap:
      name: harbor-operator-user-values
      namespace: harbor-operator
  version: 0.0.0-b6f7aad8a1ef063adc6620a4b4472c26d0efd63b
```

It is important to provide the correct values for backup in the configmap when doing this. Similarily, make sure that the app is using the most updated version of `harbor-operator`

Finally, you can deploy locally via helm by running the following command from `/harbor-operator-app/helm/harbor-operator-app` :

```
helm install harbor-operator --namespace harbor-operator -f ./values.yaml .
```

## Notes for the future

- Harbor itelf seems to be a bit unstable and as of such pods might go down from time to time. As long as the core deployment is still up they should be able to recreate themselves in the correct state. This is important because core will hold a lot of the configuration that is set up for the application. (For example user permissions set by admins in UI). In the event that backups have to be used it cannot be garanteed that this data will also be restored, as S3 is used to store image blobs, with postgres acting like a "signpost" to them images.

- Harbor contains funcitonality such as image signing and preheating which we didn't explore. It might be worth looking into these features in the future if you are going to use Harbor extensively.


