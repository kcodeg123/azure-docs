---
title: "Migration tool - Azure Database for PostgreSQL Single Server to Flexible Server - Concepts"
titleSuffix: Azure Database for PostgreSQL Flexible Server
description: Concepts about migrating your Single server to Azure Database for PostgreSQL Flexible server.
author: shriram-muthukrishnan
ms.author: shriramm
ms.reviewer: maghan
ms.date: 01/25/2024
ms.service: postgresql
ms.topic: conceptual
ms.custom:
  - references_regions
---

# Migration tool - Azure Database for PostgreSQL - Single Server to Flexible Server

[!INCLUDE [applies-to-postgres-single-flexible-server](../includes/applies-to-postgresql-single-flexible-server.md)]

Azure Database for PostgreSQL powered by the PostgreSQL community edition is available in two deployment modes:
- Flexible Server
- Single Server

Flexible Server is the next generation managed PostgreSQL service in Azure that provides maximum flexibility over your database, built-in cost-optimizations, and offers several improvements over Single Server. We recommend new customers to get started with flexible server and highly recommend existing single server customers to migrate to Flexible server to have the best experience of running PostgreSQL workloads on Azure.

In this article, we provide compelling reasons for single server customers to migrate to flexible server and a walk-through of tackling this migration in a simple, efficient and a hassle-free way.

## Why should you choose Flexible Server?

- **[Superior performance](../flexible-server/overview.md)** - Flexible server runs on Linux VM that is best suited to run PostgreSQL engine as compared to Windows environment, which is the case with Single Server.

- **[Cost Savings](../flexible-server/how-to-deploy-on-azure-free-account.md)** – Flexible server allows you to stop and start server on-demand to lower your TCO. Your compute tier billing is stopped immediately, which allows you to have significant cost savings during development, testing and for time-bound predictable production workloads.

- **[Support for new PG versions](../flexible-server/concepts-supported-versions.md)** - Flexible server currently supports PG version 11 and onwards till version 16. Newer community versions of PostgreSQL are supported only in flexible server.

- **Minimized Latency** – You can collocate your flexible server in the same availability zone as the application server that results in a minimal latency. This option isn't available in Single server.

- **[Connection Pooling](../flexible-server/concepts-pgbouncer.md)** - Flexible server has a built-in connection pooling mechanism using **pgBouncer** that can support thousands of active connections with low overhead.

- **[Server Parameters](../flexible-server/concepts-server-parameters.md)** - Flexible server offers a richer set of server parameters when compared to Single server for server configuration and tuning.

- **[Custom Maintenance Window](../flexible-server/concepts-maintenance.md)** - You can schedule the maintenance window of flexible server for a specific day and time of the week. Single server doesn't have this flexibility.

- **[High Availability](../flexible-server/concepts-high-availability.md)** - Flexible server supports HA within same availability zone and across availability zones by configuring a warm standby server that is in sync with the primary.

- **[Security](../flexible-server/concepts-security.md)** - Flexible server offers multiple layers of information protection and encryption to protect your data.

A feature-set comparison between Single server and Flexible server is available [here](../flexible-server/concepts-compare-single-server-flexible-server.md).

All the learnings acquired by running customer workloads on Azure database for PostgreSQL - Single Server is incorporated into Flexible server and we recommend using Flexible server for all your new PostgreSQL deployments on Azure.

## How to migrate from Single Server to Flexible Server?

Let us first look at the methods you can consider performing the migration from Single server to Flexible server.

**Offline Migration** – In an offline migration, all applications connecting to your single server are stopped and the database(s) is copied to flexible server.

**Online Migration** - In an online migration, applications connecting to your single server aren't stopped while database(s) are copied to flexible server. The initial copy of the databases is followed by replication to keep flexible server in sync with the single server. A cutover is performed when the flexible server is in complete sync with the single server resulting in minimal downtime.

The following table gives an overview of Offline and Online modes of migration.

| Mode | Pros | Cons |
| :--- | :--- | :--- |
| Offline | - Simple, easy and less complex to execute.<br />- Very fewer chances of failure.<br />- No restrictions in terms of database objects it can handle | Downtime to applications. |
| Online | - Very minimal downtime to application.<br />- Ideal for large databases and for customers having limited downtime requirements.<br />| - Replication used in online migration has multiple restrictions listed in this [doc](https://www.postgresql.org/docs/current/logical-replication-restrictions.html) (e.g Primary Keys needed in all tables)<br />- Tough and more complex to execute than offline migration. <br />- Greater chances of failure due to complexity of migration. <br />There's an impact on the source server's storage and compute if the migration runs for a long time. The impact needs to be monitored closely during migration. |

> [!IMPORTANT]  
> Offline migration is the recommended way to perform migrations from single server to flexible server. Customers should consider online migrations only if their downtime requirements are not met.

The following table lists the different tools available for performing the migration from single server to flexible server.

| Tool | Mode | Pros | Cons |
| :--- | :--- | :--- | :--- |
| Single to Flex Migration tool (**Recommended**) | Offline or Online* | - Managed migration service.<br />- No complex setup/pre-requisites required<br />- Simple to use portal-based migration experience<br />- Fast offline migration tool<br />- No limitations in terms of size of databases it can handle. | Downtime to applications. |
| pg_dump and pg_restore | Offline | - Tried and tested tool that is in use for a long time<br />- Suited for databases of size less than 10 GB<br />| - Need prior knowledge of setting up and using this tool<br />- Slow when compared to other tools<br />Significant downtime to your application. |

> [!NOTE]  
> The Single to Flex Migration tool is available in all Azure regions and currently supports **Offline** migrations. Support for **Online** migrations is currently available in France Central, Germany West Central, North Europe, South Africa North, UAE North, all regions across Asia, Australia, UK and public US regions. In other regions, Online migration can be enabled by the user at a subscription-level by registering for the **Online PostgreSQL migrations to Azure PostgreSQL Flexible server** preview feature as shown in the image.

:::image type="content" source="media\concepts-single-to-flexible\online-migration-feature-switch.png" alt-text="Screenshot of online PostgreSQL migrations to Azure PostgreSQL Flexible server." lightbox="media\concepts-single-to-flexible\online-migration-feature-switch.png":::

The next section of the document gives an overview of the Single to Flex Migration tool, its implementation, limitations, and the experience that makes it the recommended tool to perform migrations from single to flexible server.

## Single to Flexible Migration tool - Overview

The single to flex migration tool is a hosted solution where we spin up a purpose-built docker container in the target Flexible server VM and drive the incoming migrations. This docker container spins up on-demand when a migration is initiated from a single server and gets decommissioned once the migration is completed. The migration container uses a new binary called [pgcopydb](https://github.com/dimitri/pgcopydb) that provides a fast and efficient way of copying databases from one server to another. Though pgcopydb uses the traditional pg_dump and pg_restore for schema migration, it implements its own data migration mechanism that involves multi-process streaming parts from source to target. Also, pgcopydb bypasses pg_restore way of index building and drives that internally in a way that all indexes are built concurrently. So, the data migration process is quicker with pgcopydb. Following is the process diagram of the new version of the migration tool.

:::image type="content" source="media\concepts-single-to-flexible\concepts-flow-diagram.png" alt-text="Diagram that shows the Migration from Single Server to Flexible Server." lightbox="media\concepts-single-to-flexible\concepts-flow-diagram.png":::

## Pre-migration validations

We noticed many migrations fail due to setup issues on source and target server. Most of the issues can be categorized into the following buckets:
- Issues related to authentication/permissions for the migration user on source and target server.
- [Prerequisites](#migration-prerequisites) not being taken care of, before running the migration.
- Unsupported features/configurations between the source and target.

Pre-migration validation helps you verify if the migration setup is intact to perform a successful migration. Checks are done against a rule set and any potential problems along with the remedial actions are shown to take corrective measures.

### How to use pre-migration validation?

A new parameter called **Migration option** is introduced while creating a migration.

:::image type="content" source="media\concepts-single-to-flexible\pre-migration-validations.png" alt-text="Diagram that shows the migration option in the tool." lightbox="media\concepts-single-to-flexible\pre-migration-validations.png":::

You can pick any of the following options
- **Validate** - Use this option to check your server and database readiness for migration to the target. **This option will not start data migration and will not require any downtime to your servers.**
The result of the Validate option can be
    - **Succeeded** - No issues were found and you can plan for the migration
    - **Failed** - There were errors found during validation, which can fail the migration. Go through the list of errors along with their suggested workarounds and take corrective measures before planning the migration.
    - **Warning** - Warnings are informative messages that you need to keep in mind while planning the migration.

    Plan your migrations better by performing pre-migration validations in advance to know the potential issues you might encounter while performing migrations.

- **Migrate** - Use this option to kickstart the migration without going through validation process. It's recommended to perform validation before triggering a migration to increase the chances for a successful migration. Once validation is done, you can use this option to start the migration process.

- **Validate and Migrate** - In this option, validations are performed and then migration gets triggered if all checks are in **succeeded** or **warning** state. Validation failures don't start the migration between source and target servers.

We recommend customers to use pre-migration validations in the following way:
1) Choose **Validate** option and run pre-migration validation on an advanced date of your planned migration.
2) Analyze the output and take any remedial actions of any errors.
3) Run Step 1 again till the validation is successful
4) Start the migration using the **Validate and Migrate** option on the planned date and time.

> [!NOTE]  
> Pre-migration validations is generally available in all public regions. Support for CLI will be introduced at a later point in time.

## Migration of users/roles, ownerships and privileges

Along with data migration, the tool automatically provides the following built-in capabilities:
- Migration of users/roles present on your source server to the target server.
- Migration of ownership of all the database objects on your source server to the target server.
- Migration of permissions of database objects on your source server such as GRANTS/REVOKES to the target server.

> [!NOTE]  
> This functionality is enabled by default for flexible servers in all Azure public regions and gov clouds. It will be enabled for flexible servers in China regions soon. Also, please note that this feature is currently disabled for PostgreSQL version 16 servers, and support for it will be introduced in the near future.

## Limitations

- You can have only one active migration or validation to your Flexible server.
- The source and target server must be in the same Azure region. Cross region migrations are enabled only for servers in India, China and UAE as Flexible server might not be available in all regions within these geographies.
- The tool takes care of the migration of data and schema. It doesn't migrate managed service features such as server parameters, connection security details and firewall rules.
- The migration tool shows the number of tables copied from source to target server. You need to manually validate the data in target server post migration.
- The tool migrates only user databases. System databases like azure_sys, azure_maintenance or template databases such as template0, template1 will not be migrated.

> [!NOTE]  
> The following limitations are applicable only for flexible servers on which the migration of users/roles functionality is enabled.

- Azure Active Directory users present on your source server are not migrated to the target server. To mitigate this limitation, manually create all Azure Active Directory users on your target server using this [link](../flexible-server/how-to-manage-azure-ad-users.md) before triggering a migration. If Azure Active Directory users aren't created on target server, migration fails with appropriate error message.
- If the target flexible server uses SCRAM-SHA-256 password encryption method, connection to flexible server using the users/roles on single server fails since the passwords are encrypted using md5 algorithm. To mitigate this limitation, choose the option **MD5** for **password_encryption** server parameter on your flexible server.

## Experience

Get started with the Single to Flex migration tool by using any of the following methods:

- [Migrate using the Azure portal](../migrate/how-to-migrate-single-to-flexible-portal.md)
- [Migrate using the Azure CLI](../migrate/how-to-migrate-single-to-flexible-cli.md)

## Prerequisites

Here, we go through the phases of an overall database migration journey, with guidance on how to use Single to Flex migration tool in the process.

### Get started

#### Application compatibility

Single server supports PG version 9.6,10 and 11 while Flexible server supports PG version 11, 12, 13 and 14. Given the differences in supported versions, you might be moving across versions while migrating from single to flexible server. If that is the case, make sure your application works well with the version of flexible server you're trying to migrate to. If there are breaking changes, make sure to fix them on your application before migrating to flexible server. Use this [link](https://www.postgresql.org/docs/14/appendix-obsolete.html) to check for any breaking changes while migrating to the target version.

#### Migration prerequisites

The following pre-requisites need to be taken care of before using the Single to Flex Migration tool for migration

##### Network connectivity between Single and Flexible Server

The following table summarizes the possible network configuration on single and flexible server.

| Server Type | Public Access | Private Access |
| :--- | :--- | :--- |
| Single Server | Public access turned **ON** with firewall rules or VNet rules (service endpoints). | Public access turned OFF with private end points configured. |
| Flexible Server | Public access with firewall rules | Server deployed inside a VNet and delegated subnet. |

> [!NOTE]  
> Currently private end points are not supported in Flexible server. It will be supported at a later point time.

The following table summarizes the list of networking scenarios supported by the Single to Flex migration tool.

| Single Server Config | Flexible Server Config | Supported by Migration tool |
| :--- | :--- | :--- |
| Public Access | Public Access | Yes |
| Public Access | Private Access | Yes |
| Private Access | Public Access | No |
| Private Access | Private Access | Yes |

**Steps needed to establish connectivity between your Single and Flexible Server**
- If your single server is public access (case #1 and case #2 in the above table), there's nothing needed from your end. The single to flex migration tool automatically establishes connection between single and flexible server and the migration goes through.
- If your single server is in private access, then the only supported scenario is when your Flexible server is inside a VNet. If your flexible server is deployed in the same VNet as the private end point of your Single server, connections between single server and flexible server should automatically work provided there's no network security group(NSGs) blocking the connectivity between subnets. If flexible server is deployed in another VNet, [peering should be established between the VNets](../../virtual-network/tutorial-connect-virtual-networks-portal.md) for the connection to work between Single and Flexible server.

##### Allowlist required extensions

The migration tool automatically allowlists all extensions used by your single server databases on your flexible server except for the ones whose libraries need to be loaded at the server start.

Use the following select command to list all the extensions used on your Single server databases.

```sql
    select * from pg_extension;
```

Check if the list contains any of the following extensions:
- PG_CRON
- PG_HINT_PLAN
- PG_PARTMAN_BGW
- PG_PREWARM
- PG_STAT_STATEMENTS
- PG_AUDIT
- PGLOGICAL
- WAL2JSON

If yes, then follow the below steps.

Go to the server parameters blade and search for **shared_preload_libraries** parameter. PG_CRON and PG_STAT_STATEMENTS extensions are selected by default. Select the list of above extensions used by your single server databases to this parameter and select Save.

:::image type="content" source="media\concepts-single-to-flexible\shared-preload-libraries.png" alt-text="Diagram that shows allow listing of shared preload libraries on Flexible Server." lightbox="media\concepts-single-to-flexible\shared-preload-libraries.png":::

For the changes to take effect, server restart would be required.

:::image type="content" source="media\concepts-single-to-flexible\save-and-restart.png" alt-text="Diagram that shows save and restart option on Flexible Server." lightbox="media\concepts-single-to-flexible\save-and-restart.png":::

Use the **Save and Restart** option and wait for the flexible server to restart.

> [!NOTE]  
> If TIMESCALEDB, POSTGIS_TOPOLOGY, POSTGIS_TIGER_GEOCODER, POSTGRES_FDW or PG_PARTMAN extensions are used in your single server, please raise a support request since the migration tool does not handle these extensions.

##### Create Azure Active Directory users on target server

> [!NOTE]  
> This pre-requisite is applicable only for flexible servers on which the migration of users/roles functionality is enabled.

Execute the following query on your source server to get the list of Azure Active Directory users.

```sql
SELECT r.rolname
    FROM
      pg_roles r
      JOIN pg_auth_members am ON r.oid = am.member
      JOIN pg_roles m ON am.roleid = m.oid
    WHERE
      m.rolname IN (
        'azure_ad_admin',
        'azure_ad_user',
        'azure_ad_mfa'
      );
```
Create the Azure Active Directory users on your target flexible server using this [link](../flexible-server/how-to-manage-azure-ad-users.md) before creating a migration.

### Migration

Once the pre-migration steps are complete, you're ready to carry out the migration of the production databases of your single server. At this point, you've finalized the day and time of production migration along with a planned downtime for your applications.

- Create a flexible server with a **General-Purpose** or **Memory Optimized** compute tier. Pick a minimum 4VCore or higher SKU to complete the migration quickly. Burstable SKUs are blocked for use as migration target servers.
- Don't include HA option while creating flexible server. You can always enable it with zero downtime once the migration from single server is complete. Don't create any read-replicas yet on the flexible server.
- Before initiating the migration, stop all the applications that connect to your production server.
- Checkpoint the source server by running **checkpoint** command and restart the source server.
This command ensures any remaining applications or connections are disconnected. Additionally, you can run **select * from pg_stat_activity;** after the restart to ensure no applications is connected to the source server.

Trigger the migration of your production databases using the **Migrate** or **Validate and Migrate** option in the migration tool. The migration requires close monitoring, and the monitoring user interface of the migration tool comes in handy. Check the migration status over the period of time to ensure there is progress and wait for the migration to complete.

### Post migration

- Once the migration is complete, verify the data on your flexible server and make sure it's an exact copy of the single server.
- Post verification, enable HA option as needed on your flexible server.
- Change the SKU of the flexible server to match the application needs. This change needs a database server restart.
- If you change any server parameters from their default values in single server, copy those server parameter values in flexible server.
- Copy other server settings like tags, alerts, firewall rules (if applicable) from single server to flexible server.
- Make changes to your application to point the connection strings to flexible server.
- Monitor the database performance closely to see if it requires performance tuning.

## Related content

- [Migrate to Flexible Server by using the Azure portal](../migrate/how-to-migrate-single-to-flexible-portal.md)
- [Migrate to Flexible Server by using the Azure CLI](../migrate/how-to-migrate-single-to-flexible-cli.md)
