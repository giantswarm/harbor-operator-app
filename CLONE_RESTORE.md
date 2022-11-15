<h1>Creating and Restoring in-cluster Postgres Database Backups in Harboor</h1>

<h2>Why do we want to  create backups</h2>
- To be able to recover harbor indexed data in case of disaster so we can still have the list of cached artifacts stored in harbor

There are currently two ways to configure a database in harbor
1. Standard database configurations can be used to set the existing pre-deployed or cloud database services as the dependent database of the deploying Harbor.
2. The in-cluster database configuration can be configured to let the <b>Harbor operator automatically deploy an in-cluster PostgreSQL database service using the zalando postgres operator</b> with HA supported as the dependent database of the deploying Harbor. 

The second method is the preferred method because removes the need to manage a separate postgres instance.

<h2> Creating and storing backups in S3 using WALG </h2> 
In other to create backups stored in s3 the following fields must be set in the `pod-config` configmap the postgres operator will read environment variables from.

```
USE_WALG_BACKUP: "true"
USE_WALG_RESTORE: "true"
BACKUP_NUM_TO_RETAIN: "14" # For 2 backups per day, keep 7 days of base backups
BACKUP_SCHEDULE: "*/10 * * * *"
AWS_ACCESS_KEY_ID: < * >
AWS_SECRET_ACCESS_KEY: < * >
AWS_REGION: < * >
WALG_DISABLE_S3_SSE: "true"
WALE_DISABLE_S3_SSE: "true"
WAL_S3_BUCKET: < * > # Required, the bucket the backups will be created in
```

Setting these parameters will make sure the postgres creates backups in the specified s3 bucket

Follow these steps to create a harbor cluster with backup enabled on a GiantSwarm Management cluster

1. Create a namespace
   `kubectl create ns harbor-operator-ns`

2. Create a config map the `application.giantswarm.io/v1alpha1` App CR will use to configure the harbor operator helm chart
   
```
apiVersion: v1
kind: ConfigMap
metadata:
name: harbor-operator-user-values
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
        tag: v1.8.2
      configGeneral:
        docker_image: registry.opensource.zalan.do/acid/spilo-14:2.1-p7
      configKubernetes:
        secret_name_template: "{username}.{cluster}.credentials"
        enable_pod_antiaffinity: "true"
        pod_environment_configmap: "harbor-operator-ns/pod-config" # "<namespace app will be deployed>/<preferred name of the configmap postgres operator will get its env vars from>"
      configAwsOrGcp:
        aws_region: us-east-1
        wal_s3_bucket: <bucket name in s3 used for storing postgres backup>
 ```
3. Create the APP custom resource for harbor operator

```
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
      name: "harbor-operator-user-values" # This should be the same as the name of the configmap created in step 1
      namespace: "harbor-operator-ns"
  version: 0.0.0-c0b7706dbb0aaaa608ea25ed0f0ab14fa4a733db # Version of a recent helm chart artifact
```
4. Create the configmap used by the postgres operator to configure postgres pods, this should correlate with the
   `pod_environment_configmap` value set in step 1.
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: pod-config
  namespace: harbor-operator-ns
data:
  USE_WALG_BACKUP: "true"
  USE_WALG_RESTORE: "true"
  BACKUP_NUM_TO_RETAIN: "14" # For 2 backups per day, keep 7 days of base backups
  BACKUP_SCHEDULE: "*/10 * * * *"
  AWS_ACCESS_KEY_ID: 
  AWS_SECRET_ACCESS_KEY: 
  AWS_REGION: us-east-1
  WALG_DISABLE_S3_SSE: "true"
  WALE_DISABLE_S3_SSE: "true"
```

5. Create the Harbor CR
   1. Create a new namespace to deploy harbor in 
      `kubectl create namespace harbor-cluster`
   2. Create a txt file that holds your AWS secretkey and create a kubernetes secret using the file
      `kubectl create secret -n harbor-cluster generic secretkey --from-file=secret=./password.txt`
   3. You can use this reference manifest file to configure the harbor cluster and it's dependencies [here](https://docs.google.com/document/d/1g1sKP0fG-2qC9aLRfdMv75zJZ_B-S3I1B3upeBKgu0A)
   4. At this point, the pods will fail, reference problem 2 under the "Challenges faced while implementing this solution with Harbor Operator" to resolve it.
   5. Delete the `harbor-operator-postgres-operator-*` pod in the `harbor-operator-ns` namespace and wait for the pods to be recreated.

To confirm if backups are being created;
1. exec into one of the postgres pods e.g.
   `kubectl exec postgresql-harbor-cluster-harbor-cluster-0 -n harbor-cluster -it --sh`
2. In the pod, run `envdir /run/etc/wal-e.d/env wal-g backup-list` you'll see a list of backups created

<h2> Restoring backups from s3 using the clone method </h2>
Although using an in-cluster database configuration that Harbor operator manages has its advantages, it also comes with a few disadvantages, a major one being some of the configurable
parameters in the postgres CR are hard coded in the Harbor Operator, a few examples are the `patroni` and `clone`parameters (more on these later).

We don't have direct access to the `clone` parameter in the postgres operator, e.g
``` 
clone:
    uid: ""
    cluster: ""
```
We have to find a way to simulate what this parameter does by passing in the needed parameters in the `pod-config` configmap. These are the parameters needed to trigger the clone scripts 
on postgres pod startup


PS: This is the ideal solution for the restoring a base backup as they were tested with regular postgres clusters.

In order to restore backups from s3 these are the steps to follow:
1. The password to the old cluster needs to be known and stored immediately it is created (This password is generated by the postgres operator)
   `k get secret standby.<old-cluster-name>.credentials -n <namespace harbor CR was deployed in> -oyaml > secret.yaml`
   The "creationTimestamp", "ownerReferences", "resourceVersion" and "uid" sections can be removed from the yaml file.
2. Include the following parameters to the `pod-config` configmap used to create the initial cluster to tell the postgres operator to restore from an existing backup.
   
   Note that if you want to also want the restored database to also create backups, you'll have to add the referenced parameters in step 4 of `Creating and storing backups in S3 using WALG`
   ```
   CLONE_SCOPE: "" # The name of the cluster you want to clone, ideally it is the folder name within the spilo folder in s3
   CLONE_METHOD: "CLONE_WITH_WALE"
   CLONE_HOST: "" # The same value as CLONE_SCOPE
   CLONE_USER: "postgres"
   CLONE_WAL_S3_BUCKET: "" # s3 bucket name the backup is stored in
   CLONE_WAL_BUCKET_SCOPE_SUFFIX: "" # This is the id of the cluster, usually in the clone_scope folder in s3 (s3://<bucket-name>/spilo/<clone-scope>/<scope-suffix>) and it has to start with "/" e.g "/99e85b8a-3f92-4c86-8d42-5056348c1e7e"
   CLONE_PORT: "5432"
   CLONE_USE_WALG_RESTORE: "true"
   USEWALG_RESTORE: "true"
   CLONE_AWS_REGION: ""
   CLONE_AWS_ACCESS_KEY_ID:
   CLONE_AWS_SECRET_ACCESS_KEY:
   ```
3. You can use this reference manifest file to configure the harbor cluster and it's dependencies [here](https://docs.google.com/document/d/1g1sKP0fG-2qC9aLRfdMv75zJZ_B-S3I1B3upeBKgu0A)
4. At this point, the pods will fail, reference problem 2 under the "Challenges faced while implementing this solution with Harbor Operator" to resolve it.
5. Delete the `harbor-operator-postgres-operator-*` pod in the `harbor-operator-ns` namespace and wait for the pods to be recreated.
6. Go to the configured harbor url (set as externalURL in the HarborCluster CR) to confirm your old data has been restored 

<h2> Challenges faced while implementing this solution with Harbor Operator</h2>
While this should work as it is, we encountered some bottlenecks as we implemented the solution. I'll list them and also mention ways we mitigated them

1. `Problem:` The s3 path wasn't correct, majorly because the tools within the postgres pod (WALE, WALG, PATRONI) were outdated.

    `Solution:` Update the spilo image version used by the postgres operator to a more recent version and that should fix the issue.


2. `Problem: ` `FATAL:  no pg_hba.conf entry for host "[local]", user "postgres", database "postgres", SSL off`. This was caused because the Harbor Operator hardcoded the [pg_hba.config](https://github.com/goharbor/harbor-operator/blob/master/pkg/cluster/controllers/database/generate.go#L94) and 
    it doesn't include the `postgres` user access to make replications. 

   `Solution:` `This is a possible solution that needs to be pushed upstream`.

    To resolve this, edit the postgres operator CR and included the following under the patroni:pg_hba section and delete the pods.
    
    PS: This is not a sustainable solution as it requires a manual intervention anytime a harbor cluster is created

    ```
   pg_hba:
    - hostssl all all 0.0.0.0/0 md5
    - host    all all 0.0.0.0/0 md5
    - local   all             all                                   trust
    - local   replication     postgres                    trust
    - hostssl replication     postgres all                md5
    - local   replication     standby                    trust
    - hostssl replication     standby all                md5
    - hostssl all             +zalandos    all                pam
    - hostssl all             all                all                md5
   ```
3. `Problem: ` The clone_with_wale.py script doesn't use the right s3 path to download the backups for restoration

   `Solution: ` The "CLONE_AWS_ENDPOINT" environment variable affects the s3 path so remove it if you have it


### Post Recovery Options

After a database has been recovered successfully from a backup from s3, the CLONE parameters can be removed from the `pod-config` configmap or left alone, this is because the postgresql service will always pick the
latest data available regardless if the CLONE parameters are specified or not.
