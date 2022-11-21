# Creating in-cluster Postgres Database Backups in Harbor

## Why do we want to create backups

To be able to recover harbor indexed data in case of disaster so we can still have the list of cached artifacts stored in harbor

There are currently two ways to configure a database in harbor
1. Standard database configurations can be used to set the existing pre-deployed or cloud database services as the dependent database of the deploying Harbor.
1. The in-cluster database configuration can be configured to let the **Harbor operator automatically deploy an in-cluster PostgreSQL database service using the zalando postgres operator** with HA supported as the dependent database of the deploying Harbor.

The second method is the preferred method because it removes the need to manage a separate postgres instance.

## Installing harbor-operator-app with S3 backups using WALG

Setting these parameters will make sure the postgres creates backups in the specified s3 bucket.

Follow these steps to create a harbor cluster with backup enabled on a GiantSwarm Management cluster:

1. Create a namespace
   `kubectl create ns harbor-operator-ns`

2. Create the configmap used by the postgres operator to configure postgres pods, this should correlate with the
   `pod_environment_configmap` value set in the next step:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: pod-config
     namespace: harbor-operator-ns
   data:
     USE_WALG_BACKUP: "true"
     USE_WALG_RESTORE: "true"
     BACKUP_NUM_TO_RETAIN: "14" # For 2 backups per day, keep 7 days of base backups
     BACKUP_SCHEDULE: "*/10 * * * *" # Execute backups every 10 minutes
     AWS_ACCESS_KEY_ID: <ACCESS_KEY_ID>
     AWS_SECRET_ACCESS_KEY: <SECRET_ACCESS_KEY>
     AWS_REGION: <AWS_BUCKET_REGION>
     WALG_DISABLE_S3_SSE: "true"
     WALE_DISABLE_S3_SSE: "true"
   ```

3. Create a the harbor-operator-app configmap the `application.giantswarm.io/v1alpha1` App CR will use to configure the harbor operator helm chart:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
   name: harbor-operator-values
   namespace: harbor-operator-ns
   data:
     values: |
       installCRDs: true
       redis-operator:
         enabled: true
       minio-operator:
         enabled: false
       postgres-operator:
         enabled: true
         image:
           tag: v1.8.2 #v1.8.2 or later
         configGeneral:
           docker_image: registry.opensource.zalan.do/acid/spilo-14:2.1-p7
         configKubernetes:
           secret_name_template: "{username}.{cluster}.credentials"
           enable_pod_antiaffinity: "true"
           pod_environment_configmap: "harbor-operator-ns/pod-config" # "<harbor-operator-app namespace>/<name of the pod configmap>"
         configAwsOrGcp:
           aws_region: <S3_BACKUP_BUCKET_REGION>
           wal_s3_bucket: <S3_BACKUP_BUCKET>
    ```

4. Create the APP custom resource for the harbor operator:

   ```yaml
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
     name: harbor-operator
     namespace: harbor-operator-ns
     userConfig:
       configMap:
         name: "harbor-operator-values" # This should be the same as the name of the configmap created in step 2
         namespace: "harbor-operator-ns"
     version: <HARBOR_OPERATOR_APP_VERSION> # Version of a recent helm chart artifact
   ```

5. Create the Harbor CR
   1. Create a new namespace to deploy harbor in
      ```sh
      kubectl create namespace harbor-cluster
      ```
   2. Create a txt file that holds your AWS secretkey and create a kubernetes secret using the file:
      ```sh
      kubectl create secret -n harbor-cluster generic secretkey --from-file=secret=./password.txt
      ```
   3. You can use this [reference manifest file] (https://docs.google.com/document/d/1g1sKP0fG-2qC9aLRfdMv75zJZ_B-S3I1B3upeBKgu0A) to configure the harbor cluster and it's dependencies.
   4. At this point, the pods will fail, reference problem 2 under the "Challenges faced while implementing this solution with Harbor Operator" to resolve it.
   5. The `pg_hba` rules need to be updated within the `postgresql.acid.zalan.do` object. Create the following patch file:

      _postgresql-patch.yaml_
      ```yaml
      spec:
        patroni:
          pg_hba:
          - hostssl all             all        0.0.0.0/0 md5
          - host    all             all        0.0.0.0/0 md5
          - local   all             all        trust
          - local   replication     postgres   trust
          - hostssl replication     postgres   all       md5
          - local   replication     standby    trust
          - hostssl replication     standby    all       md5
          - hostssl all             +zalandos  all       pam
          - hostssl all             all        all       md5
      ```
   6. Patch the `postgresql.acid.zalan.do` object using the following (_you might need to alter the name of the object based off the cluster name_):
        ```sh
        kubectl -n harbor-cluster patch postgresql.acid.zalan.do postgresql-harbor-cluster-harborcluster --type merge --patch-file postgresql-patch.yaml
        ```
   7. Delete the `harbor-operator-postgres-operator-*` pod in the `harbor-operator-ns` namespace and _wait for the pods to be recreated_(this can take some time so it might be worth deleting the postgres cluster pods also).
      ```sh
      kubectl -n harbor-operator-ns delete pods -l 'app.kubernetes.io/name=postgres-operator'
      ```
      ```sh
      kubectl --namespace harbor-cluster delete pods -l 'application=spilo'
      ```

   8. The password to the POSTGRES cluster database needs to be stored immediately it is created (This password is generated by the postgres operator):
      ```sh
      kubectl get secret standby.<old-cluster-name>.credentials -n <namespace harbor CR was deployed in> -oyaml > secret.yaml
      ```
      This secret will need to be restored into the `harbor-cluster` namespace in the new cluster as part of the [restore process](./RESTORE_POSTGRES_FROM_S3_BACKUP.md).
   
      **NOTE: The "creationTimestamp", "ownerReferences", "resourceVersion" and "uid" sections can be removed from the yaml file.**

## Backup Confirmation

To confirm if backups are being created:
```sh
kubectl exec postgresql-harbor-cluster-harbor-cluster-0 -n harbor-cluster -it -- sh -c 'envdir /run/etc/wal-e.d/env wal-g backup-list'
```