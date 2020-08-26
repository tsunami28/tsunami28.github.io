# SQL-Horror-Story-Part2

Hopefully you didn't recognize yourself in my previous [post](https://doctorsolution.xyz/sql-horror-story/). After we cooled off our heads we started thinking about the solution/workaround because, once again, client was not yet ready to invest in HA solution.
<!--more-->

As it was a future Production environment we had implemented LTR backups on all databases (and off course PiTR backups are there by default).
With this knowing our first idea was to do a restore of a LTR backup to a different working server and to change config files in app - maximum downtime of an hour.

PowerShell to the rescue :)
```PowerShell
# First we wanted to grab all the latest backups
# https://docs.microsoft.com/en-us/powershell/module/az.sql/get-azsqldatabaselongtermretentionbackup?view=azps-4.4.0
Get-AzSqlDatabaseLongTermRetentionBackup -Location westeurope -ServerName "YOUR SERVER NAME" -OnlyLatestPerDatabase

# Then we initiate restore
# https://docs.microsoft.com/en-us/powershell/module/az.sql/restore-azsqldatabase?view=azps-4.4.0
# restore a specific LTR backup as an P1 database on the server $serverName of the resource group $resourceGroup
Restore-AzSqlDatabase -FromLongTermRetentionBackup -ResourceId $ltrBackup.ResourceId -ServerName $serverName -ResourceGroupName $resourceGroup -TargetDatabaseName $dbName -ServiceObjectiveName P1

# Your script should look something like this
$serverName = "YOUR BAD SERVER NAME"
$SqlServer = Get-AzSqlServer -ServerName $serverName
 
$targetElasticPool = Get-AzSqlElasticPool -ServerName "YOUR GOOD SERVER NAME" -ResourceGroupName "YOUR GOOD SERVER RG NAME"
 
$backupsLTR = Get-AzSqlDatabaseLongTermRetentionBackup -Location $SqlServer.Location -ServerName $SqlServer.ServerName -OnlyLatestPerDatabase
 
foreach ($backup in $backupsLTR) {
    # restore a specific LTR backup as an P1 database on the server $serverName of the resource group $resourceGroup
    Restore-AzSqlDatabase -FromLongTermRetentionBackup -ResourceId $backup.ResourceId -ServerName $targetElasticPool.ServerName -ResourceGroupName $backup.ResourceGroupName `
        -TargetDatabaseName $backup.DatabaseName -ElasticPoolName $targetElasticPool.ElasticPoolName
}
```
This procedure worked, but we had just one issue with it - LTR backups were "old" - 1 to 7 days and since it was a CMS in case, this data lose was unacceptable.

Easy, we thought. If we can restore LTRs we should be able to restore PiTR backups too. Looking at the procedure [here](https://docs.microsoft.com/en-us/powershell/module/az.sql/restore-azsqldatabase?view=azps-4.6.0) it looked pretty strait forward (only that "UTCDateTime" bugged me a bit).

So, we tried this.

```PowerShell
<# If you try to do a restore in time through portal, you will notice this info:
"Choose a restore point between earliest restore point and latest backup time which is 6 minute before current time."
That is why for "pointintime" property I took current time (I'm in Amsterdam) and went back 130 minutes (120 for UTC difference and 10 more because of the info up) #>
$Database = Get-AzSqlDatabase -ResourceGroupName "YOUR BAD SERVER RG NAME" -ServerName "YOUR BAD SERVER NAME" -DatabaseName "THE ONE YOU WANT TO RESTORE"
Restore-AzSqlDatabase -FromPointInTimeBackup -PointInTime (Get-Date).AddMinutes(-130) -ResourceGroupName $Database.ResourceGroupName -ServerName "YOUR GOOD SERVER NAME" `
 -TargetDatabaseName "PICK ONE" -ResourceId $Database.ResourceID -ElasticPoolName "YOUR GOOD SERVER NAME POOL"
```

Unfortunately, a new headache. Above commands were giving me some errors that I couldn't overcome for more than 2 days. Got very frustrated with that :/
After a lot of reading and combining information from different places I found out one important thing - if you want to restore PiTR to a different server, DO NOT use "restore". o.O

In order to restore PiTR backup we actually do not use restore procedures but creating new DB out from the backup.

In case any Server is down we may choose the working one and in it follow the procedure:
1. Create Database
2. Specify name and choose to put it in Elastic pool (we use Elastic pools, you might not be)
3. In "Additional settings" use Backup as source and choose the needed DB

I'm leaving scripting of this for you to explore ;)

Enjoy your free time and stay safe!
