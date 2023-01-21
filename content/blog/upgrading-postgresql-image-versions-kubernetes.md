---
title: "Upgrading PostgreSQL in Kubernetes"
date: "2023-01-21T11:36:34-06:00"
comments: false
socialShare: true
toc: false
---

I have been running an instance of bitnami's version of Postgresql 12 for a while, and I have recently upgraded/migrated data from version 12 to 15.

This post doesn't cover in-place upgrades; instead, it will require you to spin a newer container version.

## Deploy New Container Image

The first step is to deploy a new Postgres container using the updated image version. This container **MUST NOT** mount the same volume as the older Postgres container. 

Instead, it will need to mount a new volume for the database. The new Postgres server will fail if you mount the older Postgres server volume. This is because Postgres requires the data to be migrated before it can load it.

## Backup

- Install the PostgreSQL command line tool on your local machine

    ```bash
    brew install PostgreSQL
    ```

- Start Kubernetes `port-forward` for postgres service to expose the port on the local system to perform a data dump

    ```bash
    kubectl port-forward svc/postgresql-kube 5432 &
    ```

- Run `pg_dumpall`, a utility for writing out (dumping) all of your PostgreSQL databases of a cluster

    ```bash
    pg_dumpall -h 127.0.0.1 -p 5432 -U postgres -W -f database.sql
    ```

If everything goes successfully, a new file is created on the local filesystem.

### Import

We'll use the Kubernetes copy(`cp`) command to transfer the dump file into the container.

```bash
kubectl cp database.sql postgresql-0:/tmp/database.sql
```

Open a terminal shell on the new postgresql container.

```bash
kubectl exec -it postgresql-0 bash
```

Finally, import the databases.

```bash
I have no name!@postgresql-0:/$ psql -U postgres -W -f /tmp/database.sql
```

Verify the import and stop the old container. You have completed the upgrade successfully now. Congrats!
