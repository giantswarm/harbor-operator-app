# Dependencies for deploying harbor-operator-app
## Pre-deployment dependencies
### Operational requirements
In order to deploy the harbor-operator-app in a management cluster (MC), you need to have the following available:
1. S3 Buckets: 
   Two buckets are required for harbor-operator-app to function properly: 
     - One for storing the WAL backups(if you plan to backup the postgres db) and 
     - the other for storing harbor artifacts (Images, Charts etc.)
   It is advised to use separate buckets for the WAL backups and the artifacts. Also, the bucket(s) must be accessible by the MC.
3. AWS Credentials
   The AWS credentials are needed for the harbor-operator to access the S3 bucket(s). The AWS entity (IAM User, IAM Role etc.) must have the following permissions:
    - S3:ListBucket
    - S3:GetObject
    - S3:PutObject
    - S3:CreateObject
    - s3:GetBucket
4. Subdomain
    A subdomain is needed for the harbor-operator to create the ingress for the harbor instance. The subdomain must be accessible by the MC.

### Cluster requirements
The harbor-operator-app requires the following to be present in the MC:
1. PersistentVolumeClaims:
     The harbor operator requires two PVCs to be present in the MC used by the trivy pod to store reports. The PVCs must be present in the same namespace as the harbor-operator-app.
      - harbor-operator-trivy-cache (used by cachePersistentVolume under `spec.trivy.storage` field in the `harborcluster` CR)
      - harbor-operator-trivy-reports (used by reportsPersistentVolume under `spec.trivy.storage` field in the `harborcluster` CR)
2. Admin Secret Password
    The harbor-operator-app requires an admin secret password to be present in the MC. The password is stored in a secret must be present in the same namespace as the harbor-operator-app. 
    The secret must contain the following:
      key - `secret` # The key name must be `secret`
      value - `<base64 encoded password>`
    The secret is name is referenced in the `spec.harborAdminPasswordRef` field in the `HaborCluster` CR.
3. ClusterIssuer
    The harbor-operator-app requires a ClusterIssuer to be present in the MC. The ClusterIssuer is used to create the TLS certificates for the harbor instance.
    The ClusterIssuer must be of type `ClusterIssuer`
4. Certificate
    The harbor-operator-app requires a Certificate to be present in the MC. The Certificate is used to create the TLS certificates for the harbor instance.
    The Certificate must be of type `Certificate`
5. Ingress
    The harbor-operator-app requires an Ingress to be present in the MC. The Ingress is used to create the ingress for the harbor instance.
    The Ingress must be of type `Ingress`. The notary and core services must be exposed via the ingress. 
6. Cloud Provider Credentials (Optional)
   When creating inCluster postgresql (Zlando/PostgreSQL) cluster with backup or restore functionality enabled, the credentials needed to connect to
   the cloud provider can be set as secrets and referenced in the `harbor operator` app CR's configmap  E.g. 
    ```
    postgres-operator:
      configKubernetes:
        pod_environment_secret: "pod-secret"
    ```
   For AWS, the following secret is required:
    ```
    apiVersion: v1
      kind: Secret
        metadata:
           name: pod-secret
           namespace: same-namespace-as-harbor-cluster
        type: Opaque
          data:
            AWS_ACCESS_KEY_ID: <base64 encoded AWS_ACCESS_KEY_ID>
            AWS_SECRET_ACCESS_KEY: <base64 encoded AWS_SECRET_ACCESS_KEY>
            CLONE_AWS_ACCESS_KEY_ID: <base64 encoded AWS_ACCESS_KEY_ID>
            CLONE_AWS_SECRET_ACCESS_KEY: <base64 encoded AWS_SECRET_ACCESS_KEY>
    ```
    The secret must be present in the same namespace as the harbor cluster. 



### Pod Environment requirements
The harbor-operator-app requires the following to be configured in the App CR referenced configmap for the following scenerios:
1. When you want to enable backups of your postgres cluster:
   ```
   postgres-operator:
     enabled: true
      backup:
        enabled: true
        number_of_backups_to_retain: 5
        # postgres-operator.backup.number_of_backups_to_retain -- Number of backups to retain
        backup_schedule: ""
        # postgres-operator.backup.bucket -- Bucket name to store backups
        bucket: ""
        aws:
          region: ""
   ```
2. When you want to restore a postgres db from a backup
   ```
   postgres-operator:
     enabled: true
     restore:
       enabled: true
       # harbor-operator.restore.bucket -- Bucket name to store backups
       scope: ""
       # postgres-operator.restore.scope_suffix -- ID of the source cluster
       scope_suffix: ""
       # postgres-operator.restore.host -- Name of the source cluster (Same as scope)
       host: ""
       # postgres-operator.restore.bucket -- Bucket name source cluster stores backups
       bucket: ""
         aws:
            region: ""
   ```
3. When you want to enable backups for a restored postgres db
   ```
    postgres-operator:
      enabled: true
      backup:
         enabled: true
         # harbor-operator.backup.bucket -- Bucket name to store backups
         bucket: ""
         aws:
            region: ""
     restore:
       enabled: true
       # harbor-operator.restore.bucket -- Bucket name to store backups
       scope: ""
       # postgres-operator.restore.scope_suffix -- ID of the source cluster
       scope_suffix: ""
       # postgres-operator.restore.host -- Name of the source cluster (Same as scope)
       host: ""
       # postgres-operator.restore.bucket -- Bucket name source cluster stores backups
       bucket: ""
         aws:
            region: ""
    ```
