---
title: Storing Azure SQL Database Backups for up to 10 years | Microsoft Docs
description: Learn how Azure SQL Database supports storing backups for up to 10 years.
keywords: ''
services: sql-database
documentationcenter: ''
author: anosov1960
manager: jhubbard
editor: ''

ms.assetid: 66fdb8b8-5903-4d3a-802e-af08d204566e
ms.service: sql-database
ms.custom: business continuity
ms.devlang: NA
ms.topic: article
ms.tgt_pltfrm: NA
ms.workload: NA
ms.date: 11/22/2016
ms.author: carlrab; sashan

---
# Storing Azure SQL Database Backups for up to 10 years
Many applications have regulatory, compliance, or other business purposes that require you to retain the automatic full database backups beyond the 7-35 days provided by SQL Database's [automatic backups](sql-database-automated-backups.md).

The **Long-Term Backup Retention** feature enables you to store your Azure SQL Database backups in an Azure Recovery Services vault for up to 10 years. You can store up to 1000 databases per vault. You can select any backup in the vault to restore it as a new database.

> [!NOTE]
> You can enable up to 200 databases per vault during a 24-hour period. Therefore, we recommend that you use a separate vault for each server to minimize the impact of this limit. 
> 
> 

## How does SQL Database Long-Term Retention work?

Long-term retention of backups allows you to associate an Azure SQL Database server with an Azure Recovery Services vault. 

* The vault must be created in the same Azure subscription that created the SQL server and in the same geographic region and resource group. 
* You then configure a retention policy for any database. The policy causes the weekly full database backups be copied to the Recovery Services vault and retained for the specified retention period (up to 10 years). 
* You can then restore from any of these backups to a new database in any server in the subscription. The copy is performed by Azure storage from existing backups and has no performance impact on the existing database.

## How do I enable Long-Term Retention?

To configure long-term backup retention for a database:

1. Create an Azure Recovery Services vault in the same region, subscription, and resource group as your SQL Database server. 
2. Register the server to the vault
3. Create an Azure Recovery Services Protection Policy
4. Apply the protection policy to the databases that require long-term backup retention

## How do I restore a database stored with the Long-Term Retention feature?

To recover from a long-term retention backup:

1. List the vault where the backup is stored
2. List the container that is mapped to your logical server
3. List the data source within the vault that is mapped to your database
4. List the recovery points available to restore
5. Restore from the recovery point to the target server within your subscription

## How much does Long-Term Retention cost?

Long-term retention of an Azure SQL database is charged according to the [Azure backup services pricing rates](https://azure.microsoft.com/pricing/details/backup/).

After the Azure SQL Database server is registered to the vault, you are charged for the total storage that is used by the weekly backups stored in the vault.

## Configuring Long-Term Retention in the Azure portal

On the Azure SQL Database server blade, you can configure long-term retention and, if necessary, create an Azure Recovery Services vault.

- To configure long-term retention of automated backups in an Azure Recovery Services vault, see [configure long-term backup retention](sql-database-configure-long-term-retention.md)
- To recover a database from a backup in long-term retention, see [recover from a backup in long-term retention](sql-database-restore-from-long-term-retention.md)
- To view backups in the Azure Recovery Services vault, see [view backups in long-term retention](sql-database-view-backups-in-vault.md)

> [!TIP]
> For a tutorial, see [Get Started with Backup and Restore for Data Protection and Recovery](sql-database-get-started-backup-recovery.md)
>

## Configuring Long-Term Retention using PowerShell

Use the following steps to configure long-term retention using PowerShell.
1. Create a recovery service vault
   
   ```
   New-AzureRmResourceGroup -Name $ResourceGroupName –Location 'WestUS' 
   $vault = New-AzureRmRecoveryServicesVault -Name <string> -ResourceGroupName $ResourceGroupName -Location 'WestUS' 
   Set-AzureRmRecoveryServicesBackupProperties   -BackupStorageRedundancy LocallyRedundant  -Vault $vault
   ```
2. Register your Azure SQL Database Server to the vault so databases within the server can have backups stored for long term.
   
   ```
   Set-AzureRmSqlServerBackupLongTermRetentionVault -ResourceGroupName 'RG1' -ServerName 'Server1' –ResourceId $vault.Id
   ```
3. Create a retention policy for storing the backups.
   
   ```
   #retrieve the default in-memory policy object for AzureSQLServer workload and set the retention period
   $RP1 = Get-AzureRmRecoveryServicesBackupRetentionPolicyObject -WorkloadType AzureSQLDatabase
   #Sets the retention value to two years
   $RP1.RetentionDurationType='Years'
   $RP1.RetentionCount=2
   #register the policy for use with any SQL database
   Set-AzureRMRecoveryServicesVaultContext -Vault $vault
   $policy = New-AzureRmRecoveryServicesBackupProtectionPolicy -name 'SQLBackup1' –WorkloadType AzureSQLDatabase -retentionPolicy $RP1
   ```
4. Enable long-term retention for the SQL Database you want the backups stored in the vault.
   
   ```
   #for your database you can select any policy created in the vault with which your server is registered
   Set-AzureRmSqlDatabaseBackupLongTermRetentionPolicy –ResourceGroupName 'RG1' –ServerName 'Server1' -DatabaseName 'DB1' -State 'enabled' -ResourceId $policy.Id
   ```
5. List the server associated with the vault. Each server is associated with a specific container in the vault. You can list the registered servers by running the following commands:
   
   ```
   #each server has an associated container in the vault
   Set-AzureRMRecoveryServicesVaultContext -Vault $vault
   $container=Get-AzureRmRecoveryServicesBackupContainer –ContainerType AzureSQL   
   #each database has an associated backup item in the respective container
   Get-AzureRmRecoveryServicesBackupItem –container $container
   ```
6. List the databases that have a retention policy in the container. Each database has an associated backup item in the respective container. The backup item name is derived from the database name.
   
    ```
    #list the backup items in the container
    Get-AzureRmRecoveryServicesBackupItem –container $container
    ```

## Restore from a long-term retention backup

Use the following steps to restore a database from a backup in the Azure Recovery Service vault:

1. Find the Recovery Service container associated with SQL server.
   
   ```
   #the following commands find the container associated with the server 'myserver' under resource group 'myresourcegroup'
   Set-AzureRMRecoveryServicesVaultContext -Vault $vault
   $container=Get-AzureRmRecoveryServicesBackupContainer –ContainerType AzureSQL -Name 'Sql;myresourcegroup;myserver'
   ```
2. Find the backup item associated with a database.
   
    ``` 
    #the following command finds the backup item associated with the database 'mydb'
    $item = Get-AzureRmRecoveryServicesBackupItem -Container $container -WorkloadType AzureSQLDatabase -Name 'mydb' 
    ```
3. Find the backup you want to restore from.
   
   ```
   #The following command lists the backups (also known as the “recovery points”) created in the specific time period.
   $RP=Get-AzureRmRecoveryServicesBackupRecoveryPoint -Item $item –StartDate '2016-02-01' -EndDate '2016-02-20'
   ```
4. Restore from the recovery point into a new Azure SQL Database.
   
   ```
   #This command restores from a selected backup. If there are multiple recovery points in the specified range $RP[0] refers to the first one.
   Restore-AzureRMSqlDatabase –FromLongTermRetentionBackup –ResourceId $RP[0].ID TargetResourceGroupName 'RG2' -TargetServerName 'Server2' -TargetDatabaseName 'DB2' [-Edition <String>] [-ServiceObjectiveName <String>] [-ElasticPoolName <String>] [<CommonParameters>]
   ```

## Disabling Long-term Retention

The Recovery Service automatically handles cleanup of backups based on the provided retention policy. 

* To stop sending the backups for a specific database to the vault, remove the retention policy for that database.
  
    ```
    Set-AzureRmSqlDatabaseBackupLongTermRetentionPolicy –ResourceGroupName 'RG1' –ServerName 'Server1' -DatabaseName 'DB1' -State 'Disabled' -ResourceId $policy.Id
    ```

> [!NOTE]
> The backups already in the vault are not be impacted. They are automatically deleted by the Recovery Service when their retention period expires.
> 
> 

## Removing backups from the Azure Recovery Services vault

To manually remove backups from the vault.

1. Identify the container in the vault for 'myserver'
   
    ```
    Set-AzureRMRecoveryServicesVaultContext -Vault $vault 
    $container=Get-AzureRmRecoveryServicesBackupContainer –ContainerType AzureSQL -FriendlyName 'myserver'
    ```
2. Identify the backup item to delete.
   
    ``` 
    $item=Get-AzureRmRecoveryServicesBackupItem –container $container -Name 'mydb'
    ```
3. Delete the backup items (all backups for the database ‘mydb’)
   
    ```
    $job = Disable-AzureRmRecoveryServicesBackupProtection –item $item -Removerecoverypoints 
    Wait-AzureRmRecoveryServicesBackupJob $job
    ```
4. Delete the container associated with ‘myserver’
   
    ```
    Unregister-AzureRmRecoveryServicesBackupContainer –Container $container $container
    ```

## Long-Term Retention FAQ:

1. Q: Can I manually delete specific backups in the vault?

    A: Not at this point in time, the vault automatically cleans up backups when the retention period has expired.
2. Q: Can I register my server to store Backups to more than one vault?

    A: No, today you can only store backups to one vault at a time.
3. Q: Can I have a vault and server in different subscriptions?

    A: No, currently the vault and server must be in both the same subscription and resource group.
4. Q: Can I use a vault I created in a different region than my server’s region?

    A: No, the vault and server must be in the same region to minimize the copy time and avoid the traffic charges.
5. Q: How many databases can I store in one vault?

    A: Currently we only support up to 1000 databases per vault. 
6. Q. How many vaults can I create per subscription?

    A. You can create up to 25 vaults per subscription.
7. Q. How many databases can I configure per day per vault?

    A. You can only set up 200 databases per day per vault.
8. Q: Does long-term retention work with elastic pools?

    A: Yes. Any database in the pool can be configured with the retention policy.
9. Q: Can I choose the time at which the backup is created?

    A: No, SQL Database controls the backups schedule to minimize the performance impact on your databases.
10. Q: I have TDE enabled for my database. Can I use TDE with the vault? 

    A. Yes, TDE is supported. You can restore the database from the vault even if the original database no longer exists.
11. Q. What happens with the backups in the vault if my subscription is suspended? 

    A. If your subscription is suspended, we retain the existing databases and backups but the new backups are not be copied to the vault. After you reactivate the subscription, the service resumes copying backups to the vault. Your vault become accessibles to the restore operations using the backups that had been copied there before the subscription was suspended. 
12. Q: Can I get access to the SQL Database backup files so I can download / restore to SQL Server?

    A: No, not currently.
13. Q: Is it possible to have multiple Schedules (Daily, Weekly, Monthly, Yearly) within a SQL Retention Policy.

    A: No, this is only available for Virtal Machine backups at this time.
14. Q. What if we set up long-term backup retention on a database that is an active geo-replication secondary?

    A: Currently we don't take backups on replicas, and therefore, there is no option for long-term backup retention on secondaries. However, it is important for a customer to set up long-term backup retention on an active geo-replication secondary for these reasons:
    - When a failover happens and the database becomes a primary, we will take a full backup and this full backup will be uploaded to vault.
    - There is no extra cost to the customer for setting up long-term backup retention on a secondary.



## Next steps
Database backups are an essential part of any business continuity and disaster recovery strategy because they protect your data from accidental corruption or deletion. To learn about the other Azure SQL Database business continuity solutions, see [Business continuity overview](sql-database-business-continuity.md).

