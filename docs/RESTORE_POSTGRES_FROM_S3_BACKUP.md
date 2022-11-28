# Restoring Harbor POSTGRES DB from s3 Backup using the clone method

Although using an in-cluster database configuration that Harbor operator manages has its advantages, it also comes with a few disadvantages, a major one being some of the configurable parameters in the postgres CR are hard coded in the Harbor Operator, a few examples are the `patroni` and `clone`parameters (more on these later).

We don't have direct access to the [clone parameter](https://github.com/zalando/postgres-operator/blob/920f3dee3e4fa0fdf244a12312a219061ed38ba8/charts/postgres-operator/crds/postgresqls.yaml#L110-L135) in the postgres operator, e.g

```yaml
clone:
    uid: ""
    cluster: ""
```
We have to find a way to simulate what this parameter does by passing in the needed parameters in the `pod-config` configmap like we saw in the Installation guide for S3 Backups. These are the parameters needed to trigger the clone scripts on postgres pod startup.

*PS: This is the ideal solution for the restoring a base backup as they were tested with regular postgres clusters.*

In order to restore backups from s3, follow the steps below:

1. This secret will need to be restored into the `harbor-cluster` namespace in the new cluster as part of the restoration process.

2. Add the following environment values to the `pod-config` configmap configured in _step 2_ of the [Installation and Backup Guide](./CONFIGURE_POSTGRES_S3_BACKUP.md) but with the additional `CLONE_` parameters.

 **NOTE: To continue creating backups with the restored database, you'll have to add the referenced parameters in [step 2](./CONFIGURE_POSTGRES_S3_BACKUP.md#installing-harbor-operator-app-with-s3-backups-using-walg)**

 ```yaml
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
   CLONE_AWS_ACCESS_KEY_ID: <ACCESS_KEY_ID>
   CLONE_AWS_SECRET_ACCESS_KEY: <SECRET_ACCESS_KEY>
 ```
3. Please continue on from _Step 3_ within the [Installation and Backup Guide](./CONFIGURE_POSTGRES_S3_BACKUP.md). Then go to the configured harbor url (_set as externalURL in the HarborCluster CR_) to confirm your old data has been restored.



# Challenges faced while implementing this solution with Harbor Operator

While this should work as it is, we encountered some bottlenecks as we implemented the solution. I'll list them and also mention ways we mitigated them:

1. **Problem:** The `s3` path wasn't correct, majorly because the tools within the postgres pod (WALE, WALG, PATRONI) were outdated.

   **Solution:** Update the spilo image version used by the postgres operator to a more recent version and that should fix the issue.


2. **Problem:** `FATAL:  no pg_hba.conf entry for host "[local]", user "postgres", database "postgres", SSL off`. This was caused because the Harbor Operator hardcoded the [pg_hba.config](https://github.com/goharbor/harbor-operator/blob/master/pkg/cluster/controllers/database/generate.go#L94) and it doesn't include the `postgres` user access to make replications.

   **Solution:** `This is a possible solution that needs to be pushed upstream`.

    To resolve this, edit the postgres operator CR and include the following under the patroni:pg_hba section and delete the pods.

    **NOTE: This is not a sustainable solution as it requires a manual intervention anytime a harbor cluster is created. A [PR](https://github.com/goharbor/harbor-operator/pull/979) has been created to resolve this issue.**

    ```yaml
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

3. **Problem:** The `clone_with_wale.py` script doesn't use the correct `s3` path to download the backups for restoration.

   **Solution:** The `CLONE_AWS_ENDPOINT` environment variable affects the `s3` path so remove it if you have it.


## Summary

There exists a solution for backup/restore of postgres. The solution is slightly less awkward for deployment solutions utilising the [zalando postgres operator](https://github.com/zalando/postgres-operator) directly, however it involves more operational overheard since the database is then decoupled from Harbor. The _issues_ outlined above could be resolved to remove friction of using the internal postgres operator (_which still is the zalando postgres operator, but it's configuration isn't as easily configurable_).

Some important notes to consider:
- Backups stored within the Postgres Backup S3 Bucket should be verified with backups listed by the `wal-g` cli within a Production environment setting (_alerting when the condition fails_).
- The standby `postgres` secret (_along with other kubernetes secrets_) should be replicated or backed up outside of the cluster.
- System tests which verify the successful restore of postgres backup data should exist (_configured similarly to that in production_).

### Data

The current configuration of the harbor solution has its stateful data stored within 2 S3 Buckets:
- A bucket is defined for storage of artifacts by harbor which are replication from registries, and
- an S3 bucket used for postgres backups.

**NOTE: Although it's a configuration item, the standby postgres secret created by the postgres operator is extremely sensitive and is required for the successful restoration of postgres data from S3 into a new postgres database.**