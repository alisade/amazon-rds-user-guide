# Upgrading the PostgreSQL DB engine for Amazon RDS<a name="USER_UpgradeDBInstance.PostgreSQL"></a>

When Amazon RDS supports a new version of a database engine, you can upgrade your DB instances to the new version\. There are two kinds of upgrades for PostgreSQL DB instances: major version upgrades and minor version upgrades\. 

*Major version upgrades* can contain database changes that are not backward\-compatible with existing applications\. As a result, you must manually perform major version upgrades of your DB instances\. You can initiate a major version upgrade by modifying your DB instance\. However, before you perform a major version upgrade, we recommend that you follow the steps described in [ Choosing a major version upgrade for PostgreSQL ](#USER_UpgradeDBInstance.PostgreSQL.MajorVersion)\. During a major version upgrade, Amazon RDS also upgrades all of your in\-Region read replicas along with the primary DB instance\.

In contrast, *minor version upgrades* include only changes that are backward\-compatible with existing applications\. You can initiate a minor version upgrade manually by modifying your DB instance\. Or you can enable the **Auto minor version upgrade** option when creating or modifying a DB instance\. Doing so means that your DB instance is automatically upgraded after Amazon RDS tests and approves the new version\. If your PostgreSQL DB instance is using read replicas, you must upgrade all of the read replicas before the minor version upgrade of the source instance\. For more details, see [Automatic minor version upgrades for PostgreSQL](#USER_UpgradeDBInstance.PostgreSQL.Minor)\. For information about manually performing a minor version upgrade, see [Manually upgrading the engine version](USER_UpgradeDBInstance.Upgrading.md#USER_UpgradeDBInstance.Upgrading.Manual)\.

**Topics**
+ [Overview of upgrading PostgreSQL](#USER_UpgradeDBInstance.PostgreSQL.Overview)
+ [PostgreSQL version numbers](#USER_UpgradeDBInstance.PostgreSQL.VersionID)
+ [Choosing a major version upgrade for PostgreSQL](#USER_UpgradeDBInstance.PostgreSQL.MajorVersion)
+ [How to perform a major version upgrade](#USER_UpgradeDBInstance.PostgreSQL.MajorVersion.Process)
+ [Automatic minor version upgrades for PostgreSQL](#USER_UpgradeDBInstance.PostgreSQL.Minor)
+ [Upgrading PostgreSQL extensions](#USER_UpgradeDBInstance.PostgreSQL.ExtensionUpgrades)

## Overview of upgrading PostgreSQL<a name="USER_UpgradeDBInstance.PostgreSQL.Overview"></a>

To safely upgrade your DB instances, Amazon RDS uses the `pg_upgrade` utility described in the [PostgreSQL documentation](https://www.postgresql.org/docs/current/pgupgrade.html)\. 

Amazon RDS takes two DB snapshots during the upgrade process if your backup retention period is greater than 0\. The first DB snapshot is of the DB instance before any upgrade changes have been made\. If the upgrade doesn't work for your databases, you can restore this snapshot to create a DB instance running the old version\. The second DB snapshot is taken after the upgrade completes\. 

**Note**  
Amazon RDS takes DB snapshots during the upgrade process only if you have set the backup retention period for your DB instance to a number greater than 0\. To change your backup retention period, see [Modifying an Amazon RDS DB instance](Overview.DBInstance.Modifying.md)\. 

When you upgrade the primary DB instance, all the in\-Region read replicas are also automatically upgraded\. After the upgrade workflow starts, the replica instances wait for the `pg_upgrade` to complete successfully on the primary DB instance\. Then the primary instance upgrade waits for the replica instance upgrades to complete\. You experience an outage until the upgrade is complete\. If you have any older replicas from earlier major versions they aren't upgraded\. 

If your DB instance is in a Multi\-AZ deployment, both the primary writer DB instance and standby DB instances are upgraded\. The writer and standby DB instances are upgraded at the same time\. 

After an upgrade is complete, you can't revert to the previous version of the database engine\. If you want to return to the previous version, restore the DB snapshot that was taken before the upgrade to create a new DB instance\. 

## PostgreSQL version numbers<a name="USER_UpgradeDBInstance.PostgreSQL.VersionID"></a>

The version numbering sequence for the PostgreSQL database engine is as follows: 
+ For PostgreSQL versions 10 and later, the engine version number is in the form *major\.minor*\. The major version number is the integer part of the version number\. The minor version number is the fractional part of the version number\. 

  A major version upgrade increases the integer part of the version number, such as upgrading from 10\.*minor* to 11\.*minor*\.
+ For PostgreSQL versions earlier than 10, the engine version number is in the form *major\.major\.minor*\. The major engine version number is both the integer and the first fractional part of the version number\. For example, 9\.6 is a major version\. The minor version number is the third part of the version number\. For example, for version 9\.6\.12, the 12 is the minor version number\.

  A major version upgrade increases the major part of the version number\. For example, an upgrade from *9\.6*\.12 to *10*\.11 is a major version upgrade, where *9\.6* and *10* are the major version numbers\.

## Choosing a major version upgrade for PostgreSQL<a name="USER_UpgradeDBInstance.PostgreSQL.MajorVersion"></a>

Major version upgrades can contain database changes that are not backward\-compatible with previous versions of the database\. This functionality can cause your existing applications to stop working correctly\. 

As a result, Amazon RDS doesn't apply major version upgrades automatically\. To perform a major version upgrade, you modify your DB instance manually\. Make sure that you thoroughly test any upgrade to verify that your applications work correctly before applying the upgrade to your production DB instances\. When you do a PostgreSQL major version upgrade, we recommend that you follow the steps described in [How to perform a major version upgrade](#USER_UpgradeDBInstance.PostgreSQL.MajorVersion.Process)\. 

You can upgrade a PostgreSQL database to its next major version\. From some PostgreSQL database versions, you can skip to a higher major version when upgrading\. If your upgrade skips a major version, the read replicas are also upgraded to that target major version\. The following table lists the source PostgreSQL database versions and the associated target major versions available for upgrading\.

**Note**  
Upgrade targets are enabled to a higher version released at the same time as the source minor version or later\.  
If a database uses the `PostGIS` extension, you can't skip major versions for some source to target combinations\. For these circumstances, upgrade to a recent minor version, then upgrade to PostgreSQL 12, and finally upgrade to your desired target version\.  
The `pgRouting` extension isn't supported for an upgrade that skips a major version to versions 11\.x\. A major version is skipped when the upgrade goes from versions 9\.4\.x, 9\.5\.x, or 9\.6\.x to versions 11\.x\. You can drop the `pgRouting` extension and then add it again after an upgrade\.   
The `tsearch2` and `chkpass` extensions aren't supported in PostgreSQL 11 or later\. If you are upgrading to version 11\.x, drop these extensions before the upgrade\.


| Current source version | Preferred upgrade targets | Newest upgrade target | 
| --- | --- | --- | 
| 12\.5 | [13\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version131) | [13\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version131) | 
| 12\.4, 12\.3, 12\.2 | [13\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version131) | [13\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version131) | 
| 11\.x | [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | 
| 10\.15 | [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110), [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | 
| 10\.14 | [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110), [12\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version124) | [12\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version124) | 
| 10\.13 | [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110), [12\.3](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version123) | [12\.3](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version123) | 
| 10\.12 | [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110), [12\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version122) | [12\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version122) | 
| 10\.11, 10\.10, 10\.9, 10\.7, 10\.6, 10\.5, 10\.4, 10\.3, 10\.1 | [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110) | [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110) | 
| 9\.6\.20 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110), [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | 
| 9\.6\.19 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.9](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version119), [12\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version124) | [12\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version124) | 
| 9\.6\.18 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.8](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version118), [12\.3](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version123) | [12\.3](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version123) | 
| 9\.6\.17 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.7](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version117), [12\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version122) | [12\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version122) | 
| 9\.6\.16 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.6](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version116) | [11\.6](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version116) | 
| 9\.6\.15 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version115) | [11\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version114) | 
| 9\.6\.14 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version114) | [11\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version114) | 
| 9\.6\.12 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version112) | [11\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version112) | 
| 9\.6\.11 | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version111) | [11\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version111) | 
| 9\.6\.10, 9\.6\.9, 9\.6\.8, 9\.6\.6, 9\.6\.5, 9\.6\.3, 9\.6\.2, 9\.6\.1  | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015) | [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015) | 
| 9\.5\.24 | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.15](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1015), [11\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1110), [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | [12\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version125) | 
| 9\.5\.23 | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.14](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1014), [11\.9](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version119), [12\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version124) | [12\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version124) | 
| 9\.5\.22 | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.13](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1013), [11\.8](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version118), [12\.3](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version123) | [12\.3](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version123) | 
| 9\.5\.21  | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.12](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1012), [11\.7](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version117), [12\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version122) | [12\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version122) | 
| 9\.5\.20  | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.11](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1011), [11\.6](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version116) | [11\.6](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version116) | 
| 9\.5\.19 | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.10](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version1010), [11\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version115) | [11\.5](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version115) | 
| 9\.5\.18 | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.9](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version109), [11\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version114) | [11\.4](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version114) | 
| 9\.5\.16 | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [10\.7](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version107), [11\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version112) | [11\.2](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version112) | 
| 9\.5\.15 | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620), [11\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version111) | [11\.1](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version111) | 
| 9\.5\.14, 9\.5\.13, 9\.5\.12, 9\.5\.10, 9\.5\.9, 9\.5\.9, 9\.5\.7, 9\.5\.6, 9\.5\.4  | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620) | [9\.6\.20](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.version9620) | 

To get a list of all valid upgrade targets for a current source version, use the [ `describe-db-engine-versions`](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-db-engine-versions.html) CLI command\. For example:

```
export REGION=AWS-Region                         
export ENDPOINT=https://rds.AWS-Region.amazonaws.com

aws rds describe-db-engine-versions --engine postgres --region $REGION --endpoint $ENDPOINT --output text --query '*[].ValidUpgradeTarget[?IsMajorVersionUpgrade==`true`].{EngineVersion:EngineVersion}' --engine-version DB-current-version
```

## How to perform a major version upgrade<a name="USER_UpgradeDBInstance.PostgreSQL.MajorVersion.Process"></a>

We recommend the following process when upgrading an Amazon RDS PostgreSQL DB instance:

1. **Have a version\-compatible parameter group ready** – If you are using a custom parameter group, you have two options\. You can specify a default parameter group for the new DB engine version\. Or you can create your own custom parameter group for the new DB engine version\. 

   If you associate a new parameter group with a DB instance, reboot the database after the upgrade completes\. If the instance needs to be rebooted to apply the parameter group changes, the instance's parameter group status shows `pending-reboot`\. You can view an instance's parameter group status in the console or by using a describe command, such as [https://docs.aws.amazon.com/cli/latest/reference/rds/describe-db-instances.html](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-db-instances.html)\.

1. **Check for unsupported DB instance classes** – Check that your database's instance class is compatible with the PostgreSQL version you are upgrading to\. For more information, see [Supported DB engines for DB instance classes](Concepts.DBInstanceClass.md#Concepts.DBInstanceClass.Support)\.

1. **Check for unsupported usage:**
   + **Prepared transactions** – Commit or roll back all open prepared transactions before attempting an upgrade\. 

     You can use the following query to verify that there are no open prepared transactions on your instance\. 

     ```
     SELECT count(*) FROM pg_catalog.pg_prepared_xacts;
     ```
   + **Reg\* data types** – Remove all uses of the *reg\** data types before attempting an upgrade\. Except for `regtype` and `regclass`, you can't upgrade the *reg\** data types\. The pg\_upgrade utility can't persist this data type, which is used by Amazon RDS to do the upgrade\. 

     To verify that there are no uses of unsupported *reg\** data types, use the following query for each database\. 

     ```
     SELECT count(*) FROM pg_catalog.pg_class c, pg_catalog.pg_namespace n, pg_catalog.pg_attribute a 
       WHERE c.oid = a.attrelid 
           AND NOT a.attisdropped 
           AND a.atttypid IN ('pg_catalog.regproc'::pg_catalog.regtype, 
                              'pg_catalog.regprocedure'::pg_catalog.regtype, 
                              'pg_catalog.regoper'::pg_catalog.regtype, 
                              'pg_catalog.regoperator'::pg_catalog.regtype, 
                              'pg_catalog.regconfig'::pg_catalog.regtype, 
                              'pg_catalog.regdictionary'::pg_catalog.regtype) 
           AND c.relnamespace = n.oid 
           AND n.nspname NOT IN ('pg_catalog', 'information_schema');
     ```

   

1. **Handle read replicas** – An upgrade also upgrades the in\-Region read replicas along with the primary instance\.

   You can't upgrade read replicas separately\. If you could, it could lead to situations where the primary and replica instances have different PostgreSQL major versions\. However, replica upgrades might increase downtime on the primary instance\. To prevent a replica upgrade, promote the replica to a standalone instance or delete it before starting the upgrade process\.

   The upgrade process recreates the replica's parameter group based on the replica instance's current parameter group\. You can apply a custom parameter group to a replica only after the upgrade completes by using the [ `modify-db-parameter-group`](https://docs.aws.amazon.com/cli/latest/reference/rds/modify-db-parameter-group.html) CLI command\.

   Read replicas on the virtual private cloud \(VPC\) platform are upgraded but replicas on the EC2\-Classic platform aren't upgraded\. Any EC2\-Classic replicas are left in the replication terminated state after the upgrade process completes\. To move a DB instance from the EC2\-Classic platform into a VPC, see [Moving a DB instance not in a VPC into a VPC](USER_VPC.md#USER_VPC.Non-VPC2VPC)\. For more information, see [Working with PostgreSQL read replicas in Amazon RDS](USER_PostgreSQL.Replication.ReadReplicas.md)\.

1. **Perform a backup** – We recommend that you perform a backup before performing the major version upgrade so that you have a known restore point for your database\. If your backup retention period is greater than 0, the upgrade process creates DB snapshots of your DB instance before and after upgrading\. To change your backup retention period, see [Modifying an Amazon RDS DB instance](Overview.DBInstance.Modifying.md)\. To perform a backup manually, see [Creating a DB snapshot](USER_CreateSnapshot.md)\.

1. **Upgrade certain extensions before the major version upgrade** – If you plan to skip a major version with the upgrade, you need to update certain extensions *before* performing the major version upgrade\. Upgrading from versions 9\.4\.x, 9\.5\.x, or 9\.6\.x to versions 11\.x skip a major version\. The extensions to update include:
   + `address_standardizer`
   + `address_standardizer_data_us`
   + `postGIS`
   + `postgis_tiger_geocoder`
   + `postgis_topology`

   Run the following command for each extension you are using\. 

   ```
   ALTER EXTENSION PostgreSQL-extension UPDATE TO 'new-version'
   ```

   For more information, see [Upgrading PostgreSQL extensions](#USER_UpgradeDBInstance.PostgreSQL.ExtensionUpgrades)\.

1. **Drop certain extensions before the major version upgrade** – An upgrade that skips a major version to version 11\.x doesn't support updating the `pgRouting` extension\. Upgrading from versions 9\.4\.x, 9\.5\.x, or 9\.6\.x to versions 11\.x skip a major version\. It's safe to drop the `pgRouting` extension and then reinstall it to a compatible version after the upgrade\. For the extension versions you can update to, see [PostgreSQL extensions supported on Amazon RDS](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.FeatureSupport.Extensions)\.

   The `tsearch2` and `chkpass` extensions are no longer supported for PostgreSQL versions 11 or later\. If you are upgrading to version 11\.x, drop the `tsearch2`, and `chkpass` extensions before the upgrade\.

1. **Drop unknown data types** – Drop `unknown` data types depending on the target version\.

   PostgreSQL version 10 stopped supporting the `unknown` data type\. If a version 9\.6 database uses the `unknown` data type, an upgrade to a version 10 shows an error message such as the following: 

   ```
   Database instance is in a state that cannot be upgraded: PreUpgrade checks failed: 
   The instance could not be upgraded because the 'unknown' data type is used in user tables. 
   Please remove all usages of the 'unknown' data type and try again."
   ```

   To find the `unknown` data type in your database so you can remove the offending column or change it to a supported data type, use the following SQL:

   ```
   SELECT DISTINCT data_type FROM information_schema.columns WHERE data_type ILIKE 'unknown';
   ```

1. **Perform an upgrade dry run** – We highly recommend testing a major version upgrade on a duplicate of your production database before attempting the upgrade on your production database\. To create a duplicate test instance, you can either restore your database from a recent snapshot or do a point\-in\-time restore of your database to its latest restorable time\. For more information, see [Restoring from a snapshot](USER_RestoreFromSnapshot.md#USER_RestoreFromSnapshot.Restoring) or [Restoring a DB instance to a specified time](USER_PIT.md)\. For details on performing the upgrade, see [Manually upgrading the engine version](USER_UpgradeDBInstance.Upgrading.md#USER_UpgradeDBInstance.Upgrading.Manual)\. 

   During the major version upgrade, the `public` and `template1` databases and the `public` schema in every database on the instance are temporarily renamed\. These objects appear in the logs with their original name and a random string appended\. The string is appended so that custom settings such as `locale` and `owner` are preserved during the major version upgrade\. After the upgrade completes, the objects are renamed back to their original names\. 
**Note**  
During the major version upgrade process, you can't do a point\-in\-time restore of your instance\. After Amazon RDS performs the upgrade, it takes an automatic backup of the instance\. You can perform a point\-in\-time restore to times before the upgrade began and after the automatic backup of your instance has completed\. 

1. **If an upgrade fails with precheck procedure errors, resolve the issues** – During the major version upgrade process, Amazon RDS for PostgreSQL first runs a precheck procedure to identify any issues that might cause the upgrade to fail\. The precheck procedure checks all potential incompatible conditions across all databases in the instance\. 

   If the precheck encounters an issue, it creates a log event indicating the upgrade precheck failed\. The precheck process details are in an upgrade log named `pg_upgrade_precheck.log` for all the databases of a DB instance\. Amazon RDS appends a timestamp to the file name\. For more information about viewing logs, see [Accessing Amazon RDS database log files](USER_LogAccess.md)\.

   If a replica upgrade fails at precheck, replication on the failed replica is broken and the replica is put in the terminated state\. Delete the replica and recreate a new replica based on the upgraded primary instance\.

   Resolve all of the issues identified in the precheck log and then retry the major version upgrade\. The following is an example of a precheck log\.

   ```
   ------------------------------------------------------------------------
   Upgrade could not be run on Wed Apr 4 18:30:52 2018
   -------------------------------------------------------------------------
   The instance could not be upgraded from 9.6.11 to 10.6 for the following reasons.
   Please take appropriate action on databases that have usage incompatible with the requested major engine version upgrade and try the upgrade again.
   
   * There are uncommitted prepared transactions. Please commit or rollback all prepared transactions.* One or more role names start with 'pg_'. Rename all role names that start with 'pg_'.
   
   * The following issues in the database 'my"million$"db' need to be corrected before upgrading:** The ["line","reg*"] data types are used in user tables. Remove all usage of these data types.
   ** The database name contains characters that are not supported by RDS PostgreSQL. Rename the database.
   ** The database has extensions installed that are not supported on the target database version. Drop the following extensions from your database: ["tsearch2"].
   
   * The following issues in the database 'mydb' need to be corrected before upgrading:** The database has views or materialized views that depend on 'pg_stat_activity'. Drop the views.
   ```

1. **If a replica upgrade fails while upgrading the database, resolve the issue** – A failed replica is placed in the `incompatible-restore` state and replication is terminated on the DB instance\. Delete the replica and recreate a new replica based on the upgraded primary instance\.

   A replica upgrade might fail for the following reasons:
   + It was unable to catch up with the primary instance even after a wait time\.
   + It was in a terminal or incompatible lifecycle state such as storage\-full, incompatible\-restore, and so on\.
   + When the primary instance upgrade started, there was a separate minor version upgrade running on the replica\.
   + The replica instance used incompatible parameters\.
   + The replica instance was unable to communicate with the primary instance to synchronize the data folder\.

1. **Upgrade your production instance** – When the dry\-run major version upgrade is successful, you should be able to upgrade your production database with confidence\. For more information, see [Manually upgrading the engine version](USER_UpgradeDBInstance.Upgrading.md#USER_UpgradeDBInstance.Upgrading.Manual)\. 

After the major version upgrade is complete, we recommend the following:
+ Run the `ANALYZE` operation to refresh the `pg_statistic` table\.
+ A PostgreSQL upgrade doesn't upgrade any PostgreSQL extensions\. To upgrade extensions, see [Upgrading PostgreSQL extensions](#USER_UpgradeDBInstance.PostgreSQL.ExtensionUpgrades)\. 
+ Optionally, use Amazon RDS to view two logs that the pg\_upgrade utility produces\. These are `pg_upgrade_internal.log` and `pg_upgrade_server.log`\. Amazon RDS appends a timestamp to the file name for these logs\. You can view these logs as you can any other log\. For more information, see [Accessing Amazon RDS database log files](USER_LogAccess.md)\.

  You can also upload the upgrade logs to Amazon CloudWatch Logs\. For more information, see [Publishing PostgreSQL logs to Amazon CloudWatch Logs](USER_LogAccess.Concepts.PostgreSQL.md#USER_LogAccess.Concepts.PostgreSQL.PublishtoCloudWatchLogs)\.
+ To verify that everything works as expected, test your application on the upgraded database with a similar workload\. After the upgrade is verified, you can delete this test instance\.

## Automatic minor version upgrades for PostgreSQL<a name="USER_UpgradeDBInstance.PostgreSQL.Minor"></a>

If you enable the **Auto minor version upgrade** option when creating or modifying a DB instance, you can have your DB instance automatically upgraded\.

For each RDS for PostgreSQL major version, one minor version is designated by RDS as the automatic upgrade version\. After a minor version has been tested and approved by Amazon RDS, the minor version upgrade occurs automatically during your maintenance window\. RDS doesn't automatically set newer released minor versions as the automatic upgrade version\. Before RDS designates a newer automatic upgrade version, several criteria are considered, such as the following:
+ Known security issues
+ Bugs in the PostgreSQL community version
+ Overall fleet stability since the minor version was released

You can use the following AWS CLI command and script to determine the current automatic upgrade minor versions\. 

```
aws rds describe-db-engine-versions --engine postgres | grep -A 1 AutoUpgrade| grep -A 2 true |grep PostgreSQL | sort --unique | sed -e 's/"Description": "//g'
```

**Note**  
If no results are returned, there is no automatic minor version upgrade available and scheduled\.

A PostgreSQL DB instance is automatically upgraded during your maintenance window if the following criteria are met:
+ The DB instance has the **Auto minor version upgrade** option enabled\.
+ The DB instance is running a minor DB engine version that is less than the current automatic upgrade minor version\.

For more information, see [Automatically upgrading the minor engine version](USER_UpgradeDBInstance.Upgrading.md#USER_UpgradeDBInstance.Upgrading.AutoMinorVersionUpgrades)\. 

**Note**  
A PostgreSQL upgrade doesn't upgrade PostgreSQL extensions\. To upgrade extensions, see [Upgrading PostgreSQL extensions](#USER_UpgradeDBInstance.PostgreSQL.ExtensionUpgrades)\. 

## Upgrading PostgreSQL extensions<a name="USER_UpgradeDBInstance.PostgreSQL.ExtensionUpgrades"></a>

A PostgreSQL engine upgrade doesn't upgrade most PostgreSQL extensions\. To update an extension after a version upgrade, use the `ALTER EXTENSION UPDATE` command\. 

**Note**  
If you are running the `PostGIS` extension in your Amazon RDS PostgreSQL DB instance, make sure that you follow the [PostGIS upgrade instructions](https://postgis.net/docs/postgis_installation.html#upgrading) in the PostGIS documentation before you update the extension\. 

To upgrade an extension, use the following command\. 

```
ALTER EXTENSION extension_name UPDATE TO 'new_version'
```

For the list of supported versions of PostgreSQL extensions, see [PostgreSQL extensions supported on Amazon RDS](CHAP_PostgreSQL.md#PostgreSQL.Concepts.General.FeatureSupport.Extensions)\.

To list your currently installed extensions, use the PostgreSQL [pg\_extension](https://www.postgresql.org/docs/current/catalog-pg-extension.html) catalog in the following command\.

```
SELECT * FROM pg_extension;
```

To view a list of the specific extension versions that are available for your installation, use the PostgreSQL [ pg\_available\_extension\_versions](https://www.postgresql.org/docs/current/view-pg-available-extension-versions.html) view in the following command\.

```
SELECT * FROM pg_available_extension_versions;
```