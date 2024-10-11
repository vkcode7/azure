## Section 10: Implementing Azure Security and Auth

### MS Entra ID: (previously aka Azure Active Directory) - An Identity Provider
This is a cloud based identity and access management service. Can be used for Azure, MS 365 and other SaaS based softwares. Here you can use it for:

1. Setting up users / applications - Users can be assigned password, roles, groups etc
2. Autheticating users - check credentials and allow them
3. Authorizing users - check their permisisons for various resources/actions

### Role Based Access Control (RBAC)

There are many builtin roles (owner, contributor, reader etc) and you can also define your own custom role<br>
Role can be applied at Subscription Level, Resource Group Level or at Resource Level<br>

Azure builtin role: https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles

### Checking access
To see who is given access on a resource go to that Resource say a Storage Resource -> Access Control (IAM) -> Role Assignments. To give certain access click on "Add" -> "Add Role Assignment" and then in "Job function role" search for "Storage" and you will see storage related builtin role. Select "Storage Account Contributor role" and then add particular users/group or managed identity to that role.

### Applications authentication via App objects

```c#
using Azure.Storage.Blobs;

string conn_str = "DefaultEndpointsProtocol=https;AccountName=vkdata1;AccountKey=6r4XWAK3VQkHhjDAMAGEDQfOBBJVStkmR4oVcibxJ2TeCO46jbm55qQMptG5fMVsrIr/yC+AStx0vXoQ==;EndpointSuffix=core.windows.net";
BlobServiceClient sc = new BlobServiceClient (conn_str);

string containerName="data";
string fileName="script01.ps1";
string path=@"C:\tmp4\script01.ps1";

BlobContainerClient bcClient = sc.GetBlobContainerClient(containerName);

BlobClient blobClient= bcClient.GetBlobClient(fileName);
await blobClient.DownloadToAsync(path);

Console.WriteLine("DownLoad Complete");
```
