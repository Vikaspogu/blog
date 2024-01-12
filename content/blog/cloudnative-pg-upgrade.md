---
title: " A Seamless Upgrade: Cloud-Native PostgreSQL Database Version 15 to 16"
date: "2024-01-12T12:11:03-06:00"
draft: false
comments: false
socialShare: true
toc: false
tags:
  - postgresql
  - cloudnative-pg
---

## Introduction

In the ever-evolving landscape of cloud-native technologies, staying up-to-date with the latest software versions is crucial for ensuring optimal performance, security, and access to new features. This blog post will guide you through the process of upgrading a PostgreSQL database instance from version 15 to version 16 in a cloud-native environment.

## Before You Begin

Before diving into the upgrade process, it's essential to perform a few preliminary steps to ensure a smooth transition:

### Backup Your Database

Begin by creating a comprehensive backup of your PostgreSQL database. This step is critical in case anything goes wrong during the upgrade process, allowing you to restore your database to its previous state.

### Check Compatibility

Verify that your applications and any other dependencies are compatible with PostgreSQL 16. Review release notes and documentation to identify any potential issues that may arise.

### Update Extensions and Dependencies

Make sure that all extensions and dependencies are compatible with PostgreSQL 16. Update them as needed to avoid compatibility issues after the upgrade.

## Upgrading Cloud-Native PostgreSQL Database

### Review Release Notes

Start by carefully reviewing the release notes for PostgreSQL 16. Take note of any changes, new features, and potential issues that may affect your database.

### Stop Database Activity

Before initiating the upgrade, stop all database activity to prevent any data inconsistencies or corruption during the process. Notify users in advance to minimize disruptions.

### Upgrading PostgreSQL Cluster

This post doesn’t cover in-place upgrades; instead, it will require you to spin a newer postgresql cluster with version 16 and [bootstrap from another cluster](https://cloudnative-pg.io/documentation/1.22/bootstrap/#bootstrap-from-another-cluster).

- Use the new Postgres version 16 as the cluster image

  ```yaml
  apiVersion: postgresql.cnpg.io/v1
  kind: Cluster
  metadata:
    name: upgrade-cluster-16
  spec:
    instances: 3
    imageName: ghcr.io/cloudnative-pg/postgresql:16.1-13
  ```

- Add external cluster configuration to perform a monolith [import](https://cloudnative-pg.io/documentation/1.22/database_import/#the-monolith-type) using the version 15 cluster as the connection parameters.

  ```yaml
  apiVersion: postgresql.cnpg.io/v1
  kind: Cluster
  metadata:
    name: upgrade-cluster-16
  spec:
    instances: 3
    imageName: ghcr.io/cloudnative-pg/postgresql:16.1-13
    externalClusters:
      - name: cluster-pg96
        connectionParameters:
          # Use the correct IP or host name for the source database
          host: pg96.local
          user: postgres
          dbname: postgres
        password:
          name: cluster-pg96-superuser
          key: password
  ```

- Configure bootstrap source with cluster name from above

  ```yaml
  apiVersion: postgresql.cnpg.io/v1
  kind: Cluster
  metadata:
    name: upgrade-cluster-16
  spec:
    instances: 3
    imageName: ghcr.io/cloudnative-pg/postgresql:16.1-13
    externalClusters:
      - name: cluster-pg96
        connectionParameters:
          # Use the correct IP or host name for the source database
          host: pg96.local
          user: postgres
          dbname: postgres
        password:
          name: cluster-pg96-superuser
          key: password
    bootstrap:
      initdb:
        import:
          type: monolith
          databases:
            - "*"
          roles:
            - "*"
          source:
            externalCluster: cluster-pg96
  ```

- Add configuration to perform [backup](https://cloudnative-pg.io/documentation/1.22/backup_barmanobjectstore/) after the bootstrap import to an object store

  ```yaml
  apiVersion: postgresql.cnpg.io/v1
  kind: Cluster
  metadata:
    name: upgrade-cluster-16
  spec:
    instances: 3
    imageName: ghcr.io/cloudnative-pg/postgresql:16.1-13
    externalClusters:
      - name: cluster-pg96
        connectionParameters:
          # Use the correct IP or host name for the source database
          host: pg96.local
          user: postgres
          dbname: postgres
        password:
          name: cluster-pg96-superuser
          key: password
    bootstrap:
      initdb:
        import:
          type: monolith
          databases:
            - "*"
          roles:
            - "*"
          source:
            externalCluster: cluster-pg96
    backup:
      barmanObjectStore:
        destinationPath: "<destination path here>"
        serverName: upgrade-cluster-16
        s3Credentials:
          accessKeyId:
            name: aws-creds
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: aws-creds
            key: ACCESS_SECRET_KEY
      retentionPolicy: "30d"
  ```

- Apply changes to the cluster and wait for the cluster to be in a healthy state

  ```bash
  kubectl apply -f upgrade-cluster-16.yaml
  ```

- Finally, swap the bootstrap recovery source to the new backup generated from earlier steps

  ```yaml
  apiVersion: postgresql.cnpg.io/v1
  kind: Cluster
  metadata:
    name: main
  spec:
    instances: 3
    imageName: ghcr.io/cloudnative-pg/postgresql:16.1-13
    bootstrap:
      recovery:
        source: upgrade-cluster-16
  ```

- Delete the cluster created in steps 1-4

  ```bash
  kubectl delete -f upgrade-cluster-16.yaml
  ```

## Conclusion

Upgrading a cloud-native PostgreSQL database instance from version 15 to 16 requires careful planning and execution. By following the steps outlined in this blog post, you can ensure a seamless transition, taking advantage of the latest features and improvements while maintaining the integrity of your data. Keep in mind that each cloud provider may have specific guidelines or tools for managing PostgreSQL upgrades, so consult their documentation for additional guidance.
