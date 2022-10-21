# Postgres options and setup

## Why?

It is important as having a backed up database that allows us to index S3 and persist data between different harbor instances.

## Prerequisites

1. Deploy the harbor operator using the app-cr.yaml

```
apiVersion: v1
data:
  values: |
    installCRDs: true
    redis-operator:
      enabled: true
    minio-operator:
      enabled: false
    postgres-operator:
      enabled: false
kind: ConfigMap
metadata:
  name: harbor-operator-user-values
  namespace: harbor-operator-ns
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
  name: harbor-operator
  namespace: harbor-operator-ns
  userConfig:
    configMap:
      name: "harbor-operator-user-values"
      namespace: "harbor-operator-ns"
  version: 0.0.0-a1333bd7599ddfa8f5c7fb6ce26dc777b84fa7b3
```

2. You should have a harbor cluster set up with S3 storage enabled.

```
  storage:
    kind: S3
    spec:
      s3:
        accesskey: <AWS-key>
        secretkeyRef: <reference to secret containing your AWS secretkey> *
        region: eu-west-2
        regionendpoint: http://s3.eu-west-2.amazonaws.com
        bucket: <your bucket name>
```

\* The secret key should be set to `secret`

## References for the fullstack.yaml

For configuring external postgreSQL:
https://github.com/goharbor/harbor-operator/blob/master/docs/CRD/custom-resource-definition.md

For configuring the fullstack.yaml:
https://github.com/goharbor/harbor-operator/blob/master/manifests/samples/full_stack.yaml

# External postgres setup

## Summary

With this setup we seperate postgres from the kubernetes cluster and connect to it via the egress of the external IPs. This abstraction is good for failure resistance, however, it may be costly and harder to scale.

## Steps

1. Create a postgres database through the provisioner of your choice. For example amazons RDS.

2. Whitelist your nodes external IPs on the incoming firewall rules for the databases VPC.

- Use `kubectl get nodes -o wide` to obtain the external IPs of your nodes.
- In AWS navigate to RDS > Databases > \<your Postgres DB\> and look under connectivity & security.
- Click on the link under VPC security groups, inbound rules, edit inbound rules.
- Add each of the nodes IPs using port 5432 and custom TCP/PostgreSQL

3. Configure your external database to work with harbor.

- Download the psql client using `brew install postgresql`
- Log in to your database using `psql -h <Replace with host address of DB found on RDS> -p 5432 -U harbor -W postgres`
- When prompted enter the password your created earlier to connect to the DB
- Use the following SQL commands to create the nessecary databases inside your instance.

```
CREATE DATABASE notaryserver;
CREATE DATABASE notarysigner;
CREATE DATABASE registry ENCODING 'UTF8';
CREATE DATABASE clair;
CREATE DATABASE core;

CREATE USER harbor;
ALTER USER harbor WITH ENCRYPTED PASSWORD 'change-this-password';

GRANT ALL PRIVILEGES ON DATABASE notaryserver TO harbor;
GRANT ALL PRIVILEGES ON DATABASE notarysigner TO harbor;
GRANT ALL PRIVILEGES ON DATABASE registry TO harbor;
GRANT ALL PRIVILEGES ON DATABASE clair TO harbor;
GRANT ALL PRIVILEGES ON DATABASE core TO harbor;
```

4. Add the neccessary configuration to the fullstack.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: psql-password
  namespace: <namespace you are deploying the fullstack to>
stringData:
  postgresql-password: <your password in plain text>
type: Opaque
```

```
database:
    kind: PostgreSQL
    spec:
      postgresql:
        # PostgreSQL user name to connect as.
        # Defaults to be the same as the operating system name of the user running the application.
        username: harbor # Required
        # Secret containing the password to be used if the server demands password authentication.
        passwordRef: psql-password # Optional
        # PostgreSQL hosts.
        # At least 1.
        hosts:
          # Name of host to connect to.
          # If a host name begins with a slash, it specifies Unix-domain communication rather than
          # TCP/IP communication; the value is the name of the directory in which the socket file is stored.
          - host: <the address of your DB instance> # Required
          # Port number to connect to at the server host,
          # or socket file name extension for Unix-domain connections.
          # Zero, specifies the default port number established when PostgreSQL was built.
            port: 5432 # Optional
        # PostgreSQL has native support for using SSL connections to encrypt client/server communications for increased security.
        # Supports values ["disable","allow","require","verify-ca","verify-full"].
        sslMode: require # Optional
        prefix: "" # Optional (Will prefix the 'core database' if you chose to use this, update the name of your core DB)
```

5. Apply your fullstack and observe your objects being saved between instances

# Internal postgres setup

## Summary

We can use postgres operator in order to set up a postgres database in the cluster, that is not directly tied to harbor. This layer of abstraction potentially adds more faliure resistance, but means more configuration and management of postgres is required.

## Steps

1. Add the helm chart for postgres-operator

`helm repo add postgres-operator-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator`

2. Install the postgres-operator

`helm install postgres-operator postgres-operator-charts/postgres-operator --namespace container-solutions`

3. Apply manifest the following minimal manifest

```
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: harbor-postgres-deployment
  namespace: container-solutions
spec:
  teamId: "harbor"
  volume:
    size: 1Gi
  numberOfInstances: 2
  users:
    harbor:  # database owner
    - superuser
    - createdb
  databases:
    notaryserver: harbor
    notarysigner: harbor
    registry: harbor
    clair: harbor
    core: harbor
  preparedDatabases:
    bar: {}
  postgresql:
    version: "12"
```

4. Edit password key

- Harbor needs the password generated for postgres by the operator deployment.
- The secret key must be set to `postgresql-password` for harbor to be able to recognise it.
- To do this currently we are using a manual solution.

`k edit secrets -n container-solutions harbor.harbor-postgres-deployment.credentials.postgresql.acid.zalan.do`

Change `password` to `postgresql-password`


5. Update the psql-secret in the fullstack to match the value

```
database:
    kind: PostgreSQL
    spec:
      postgresql:
        username: harbor
        passwordRef: harbor.harbor-postgres-deployment.credentials.postgresql.acid.zalan.do
        hosts:
          - host: harbor-postgres-deployment.container-solutions.svc.cluster.local
            port: 5432
        sslMode: require 
        prefix: "" 
```

- You can see passwordRef is pointing towards the edited secret.
- The host uses kubernetes internal DNS routing to connect to the postgres instance.

6. Alternate option

- You can set up postgres in a different namespace.
- This will mean you will have to create a secret in the harbor namespace with a `key:pair` value of `postgresql-password:<generated password from the postgres operator secret>`
- Edit the rest of your yamls to reflect the appropriate namescape and ensure you pass your secret in to the fullstack database configuration.

```
apiVersion: v1
kind: Secret
metadata:
  name: psql-password
  namespace: <namespace you are deploying the fullstack to>
stringData:
  postgresql-password: <password from postgres operator secret>
type: Opaque
```

```
database:
    kind: PostgreSQL
    spec:
      postgresql:
        username: harbor
        passwordRef: postgres-secret
        hosts:
          - host: harbor-postgres-deployment.postgres-ns.svc.cluster.local
            port: 5432
        sslMode: require 
        prefix: "" 
```

- As you can see the host is now pointing to the alternate namespace that the postgres operator has been created in.
- The secret is also created manually and the password populated by the value from the postgres operator secret.

## Debugging

1. To test the connection inside the pod use portforwarding and connect locally to the database.

`kubectl port-forward --namespace=container-solutions pod/harbor-postgres-deployment-0 5432:5432`

`psql -h 127.0.0.1 -p 5432 -U harbor core`

# Internal harbor postgres setup

## Summary 

With this setup postgres is tied directly to harbor through the incluster option avilable in the fullstack yaml. If the harbor is deleted with the fullstack yaml, data will not be persisted as the postgres is also deleted. To circumvent this we will need a way to regularly back up the postgres contents.

## Steps

1. Ensure that when you deploy the app the the configMap has `postgres-operator:` set to `enabled: true`

```
apiVersion: v1
data:
  values: |
    installCRDs: true
    redis-operator:
      enabled: true
    minio-operator:
      enabled: false
    postgres-operator:
      enabled: true
kind: ConfigMap
metadata:
  name: harbor-operator-user-values
  namespace: harbor-operator-ns
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
  name: harbor-operator
  namespace: harbor-operator-ns
  userConfig:
    configMap:
      name: "harbor-operator-user-values"
      namespace: "harbor-operator-ns"
  version: 0.0.0-a1333bd7599ddfa8f5c7fb6ce26dc777b84fa7b3
```
2. Deploy your fullstack yaml with the following database configuration.

```
  database:
    kind: Zlando/PostgreSQL
    spec:
      zlandoPostgreSql:
        operatorVersion: "1.5.0"
        storage: 1Gi
        replicas: 3
        resources:
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 250Mi
```

3. Set up a way to back up the postgres instance so that if it is destroyed it can be recreated.

- (Yet to be discovered)