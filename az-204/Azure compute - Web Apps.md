## VM are IaaS whereas WebApps are PaaS
- Azure Web Apps has features such as autoscaling and security
- DevOps capability such as CD (cont deployment)
- You dont maintain underlying resources
- can host java/ .net/ python etc

## Creation of Web App resource (aka Compute Machines)
- Create with a unique web app name
- Select a .NET runtime stack, or python/java version
- Select OS - linux or Windows
- Select or Create a new App Service Plan (tells the CPU/Memory etc to be used) and redundancy

## Publish
- publishing is very easy from VS
- select subscription, resource group and web app name and publish

## Azure SQL DB
- You can download and install SSMS tool (SQL Server Mgmt Studio) locally and access Azure SQL DB via that

## Connecting to Az DB from c#
- Install NuGet pkg System.Data.SqlClient
- Establish connection using SqlConnection() and use SqlCommand to execute your SQL command
- Use SqlDataReader to read the data 

## Integration with GitHub
- Go to WebApp -> Deployment -> Deployment Center
- In Settings select GitHub (Other options are Bitbucket, Local Git, Azure repos, External Git, OneDrive, Dropbox)
- Provide GitHub creds and it will retrieve the code, build it and deploy
- Now if you commit a change in main branch of GitHub, change will be deployed automatically on WebApp

## Connection Strings
- Can be stored in Azure itself
- go to WebApp => Settings -=> Configuration => Application Settings =? Connection Strings
- create a new conn string and give it name, value and type (SQLAzure)
- we can then use IConfiguration via Dep Injection in c# code to get access to this conn string we created

## Web App Logging
- Application logging - your log messages
- Web Server logging - records raw HTTP request data
- Detailed error messages - stores copies of .htm error pages sent to client
- Deployment logging - publish related
- You can turn the logs under Web App => Monitoring
- You can see the real time logs inder Mointoring => Log Stream

## Auto Scaling
- With web apps we can configure auto scaling based on certain metrices such as CPU percentage
- You can create rules to scale in/out the web apps
- You can specify min/max count for web apps



