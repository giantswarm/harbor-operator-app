<h1>Creating and Restoring in-cluster Postgres Database Backups in Harboor</h1>

<h2>Why do we want to  create backups</h2>
- To be able to recover harbor indexed data in case of disaster so we can still have the list of cached artifacts stored in harbor

There are currently two ways configure a database in harbor
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

You can confirm if backups are being created by running `envdir /run/etc/wal-e.d/env wal-g backup-list`, you'll see a list of backups in s3

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

```
CLONE_SCOPE: ""
CLONE_METHOD: "CLONE_WITH_WALE"
CLONE_HOST: ""
CLONE_USER: "postgres"
CLONE_WAL_S3_BUCKET: ""
CLONE_WAL_BUCKET_SCOPE_SUFFIX: ""
CLONE_PORT: "5432"
CLONE_USE_WALG_RESTORE: "true"
USEWALG_RESTORE: "true"
CLONE_AWS_REGION: ""
CLONE_AWS_ACCESS_KEY_ID: 
CLONE_AWS_SECRET_ACCESS_KEY: 
```

PS: This is the ideal solution for the restoring a base backup as they were tested with regular postgres clusters.


<h2> Challenges faced while implementing this solution with Harbor Operator</h2>
While this should work as it is, we encountered some bottlenecks as we implemented the solution. I'll list them and also mention ways we mitigated them

1. `Problem:` The s3 path wasn't correct, majorly because the tools within the postgres pod (WALE, WALG, PATRONI) were outdated.

    `Solution:` I updated the spilio image version used by the postgres operator to a more recent version and that fixed the issue.


2. `Problem: ` `FATAL:  no pg_hba.conf entry for host "[local]", user "postgres", database "postgres", SSL off`. This was caused because the Harbor Operator harcoded the [pg_hba.config](https://github.com/goharbor/harbor-operator/blob/master/pkg/cluster/controllers/database/generate.go#L94) and 
    it doesn't include the `postgres` user access to make replications. 

   `Solution:` `This is a possible solution that needs to be pushed upstream`. 

    I had to edit the postgres operator CR and included the following under the patroni:pg_hba section and restarted the pods to make it work.
    
    PS: This is not a sustainable solution as it requires a manual intervention anytime a harbor cluster is created

    ```
   pg_hba:
    - hostssl all all 0.0.0.0/0 md5
    - host    all all 0.0.0.0/0 md5
    - local   all             all                                   trust
    - local   replication     postgres                    trust
    - hostssl replication     postgres all                md5
    - hostssl all             +zalandos    all                pam
    - hostssl all             all                all                md5
   ```