# Mighty Azure portal vol2

Providing good billing overview and reports for customers through CSP is still not fully integrated in either MSP or Azure portal. That's why most providers use 3rd party solutions.
Browsing through one of those for a client I noticed strange reports for a month or two. 
We implemented Azure SQL with Elastic Pool. Great thing with EP is that you pay just for selected DTU and you may create ["unlimited"](https://docs.microsoft.com/en-us/azure/azure-sql/database/resource-limits-dtu-elastic-pools) number of databases. But, if you create a new database and don't add it to the pool, it will be billed separately.
<!--more-->
Well, the reports showed that we spent x amount for EP (which was fine) and y amount for standalone DBs (which was not fine). This indicated that we forgot to put DBs to EP after creation. I started investigating how to find a date/time when the DB was added to the EP. There is no such property on the database and not in Elastic Pool, either. I spent couple of hours behind PowerShell and have to admit that this was a rare moment it didn't help me.

I spoke to a couple of Senior Azure Engineers, but nobody had a solution (or a new idea). Apparently I was the first person even asking for a such an information o.O

I decided to try my luck with MS support and thank you Azure SQL-DB Team for your time and analysis.
So here are the steps to follow:
1. Go to resource group where your database is deployed
2. Click on deployments
3. Filter deployments with "Microsoft.SQL.EditPool"
4. Click on deployment name and choose "Input" from toolbar on the left (examine all deployments if there are multiple)
5. The line "addDatabases" will give you exact info of databases that were added to the EP

Now when I had needed data, I was able to approach 3rd party for troubleshooting their reports (yeah, we added DBs to EP right after creation :))

Stay tuned and stay healthy!

