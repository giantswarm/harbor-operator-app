# Setting up external postgres to restore from S3

## Why?

This is important as having an external database allows us to index S3 and persist data between different harbor instances.

## Prerequisites

- You should have a harbor cluster set up with S3 storage enabled (fullstack.yaml): 

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

- Set up a postgres database on your cloud provider (we're using AWS).
- This can be done in RDS.
- Set your user to harbor-admin and create an appropriate password.

## Steps

1. Whitelist your nodes external IPs on the incoming firewall rules for the databases VPC.

- Use `kubectl get nodes -o wide` to obtain the external IPs of your nodes.
- In AWS navigate to RDS > Databases > \<your Postgres DB\> and look under connectivity & security.
- Click on the link under VPC security groups, inbound rules, edit inbound rules.
- Add each of the nodes IPs using port 5432 and custom TCP/PostgreSQL

2. Configure your external database to work with harbor.

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

3. Add the neccessary configuration to the fullstack.yaml

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

References for the fullstack.yaml:

For configuring external postgreSQL:
https://github.com/goharbor/harbor-operator/blob/master/docs/CRD/custom-resource-definition.md

For configuring the fullstack.yaml:
https://github.com/goharbor/harbor-operator/blob/master/manifests/samples/full_stack.yaml

4. Apply your fullstack and observe your objects being saved between instances