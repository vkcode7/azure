## VM are IaaS whereas WebApps are PaaS
- Azure Web Apps has features such as autoscaling and security
- DevOps capability such as CD (cont deployment)
- You dont maintain underlying resources
- can host java/ .net/ python etc

## Creation of Web App resource
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

