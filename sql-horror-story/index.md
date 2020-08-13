# SQL Horror Story

For the past 8 months I've been working on an Azure project for a financial institution. I've never imagined so many security and compliancy checks. The team did a great job - we passed all advisory scans with high grades and were ready to go live in a couple of days.

Important part of the infrastructure was relying on [Azure SQL](https://docs.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview).
Monday morning. I've made myself a cup of coffee and logged in to the portal and systems to do a base check once again. And this is where our story begins.

<!--more-->

Our applications couldn't reach one Acceptance and one (future) Production SQL server, databases to be precise. I've checked the health of Servers, Databases and Elastic pool and they were all showing healthy state. I tried to login to servers via SSMS and that worked, but if I would try to execute the simplest queries DBs would freeze. More than 80% of the calls would timeout.
Then we noticed one strange behavior looking at the "Activity log" of an affected server - Operation name: Health Event Activated.
```json
"properties": {
        "title": "Degraded",
        "details": null,
        "currentHealthStatus": "Degraded",
        "previousHealthStatus": "Available",
        "type": "Downtime",
        "cause": "PlatformInitiated"
    },
```
This would go in circles of 10-15 minutes from "Health Event Resolved" to "Health Event Activated". While we were opening the case with support team, we noticed that this behavior started sometime around 1AM Saturday!!! Well, lesson learn - we have to monitor more things than we did, but I was shocked that MS team didn't saw this before us (48+h).

This was resolved by high-end MS engineering team late night on Monday and the RCA we received was: "The unavailability was caused by an access violation in Storage layer. We have identified the issue and the defect owner has submitted a fix. We are planning to roll out the fix during the next deployment cycle which happens within a month. We apologize for the inconvenience caused by the issue."

Since the client is not ready to invest in any kind of additional high-availability we had to create a workaround in case this "1 in a million" situation happens again.

Stay tuned for next post, as I will explain what we did in detail.

