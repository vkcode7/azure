# Section 10: Implementing Azure Security and Auth

### MS Entra ID: (previously aka Azure Active Directory) - An Identity Provider
This is a cloud based identity and access management service. Can be used for Azure, MS 365 and other SaaS based softwares. Here you can use it for:

1. Setting up users / applications - Users can be assigned password, roles, groups, app objects etc
2. Autheticating users - check credentials and allow them
3. Authorizing users - check their permisisons for various resources/actions

### Role Based Access Control (RBAC)

There are many builtin roles (owner, contributor, reader etc) and you can also define your own custom role<br>
Role can be applied at Subscription Level, Resource Group Level or at Resource Level<br>

Azure builtin role: https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles

### Checking access
To see who is given access on a resource go to that Resource say a Storage Resource -> Access Control (IAM) -> Role Assignments. To give certain access click on "Add" -> "Add Role Assignment" and then in "Job function role" search for "Storage" and you will see storage related builtin role. Select "Storage Account Contributor role" and then add particular users/group or managed identity to that role.

### Using a connection string or Role-Based Access Control (RBAC) each has its own benefits and use cases. Here’s a comparison of both:

#### Benefits of Using Connection Strings:
- Simpler for Programmatic Access: Connection strings contain the storage account keys (account name and access keys), which provide full access to the storage account's resources (blobs, queues, tables, files). This makes them convenient for scripts, local development, or when your application requires full access to the storage account.
  
- Works Offline: Since connection strings are self-contained and do not require any external authentication system (like Azure AD), they can be used offline without any dependency on Azure AD token fetching or renewal.
  
- Compatibility with Older Applications: Some older applications or tools may not support Azure AD authentication or RBAC, and may only work with connection strings or account keys.
- Access Without Azure AD Setup: If you don’t have Azure AD configured or don't want to set up RBAC roles for each user or service, using a connection string is a straightforward approach to give access.
- Full Control: Connection strings grant full control over the storage account. This can be beneficial in situations where you need to manage all operations (e.g., development, testing, internal applications) without configuring specific permissions.

#### Benefits of Using RBAC (Azure AD Authentication):
- Granular Access Control: RBAC allows you to provide fine-grained access control over who can access the storage account. For example, a Blob Data Reader role grants read-only access to blob data, but not write or delete permissions, limiting the risk of accidental or unauthorized actions.
  
- Enhanced Security: Connection strings expose full access to the storage account, making them riskier if compromised. In contrast, with RBAC, users get specific, scoped permissions, reducing the chance of accidental or malicious misuse. Users and services are authenticated with Azure AD, and no sensitive credentials (like account keys) are exposed.
  
- Centralized Identity Management: RBAC leverages Azure AD for identity and access management, allowing you to manage permissions across multiple services and resources through a central identity provider, rather than relying on manually distributing and rotating connection strings or keys.
  
- Keyless Access: No need to manage, store, or rotate connection strings or storage account keys. Azure AD handles token issuance, which is more secure and reduces administrative overhead.

- Audit and Compliance: Azure AD provides auditing and monitoring of who is accessing resources. This is critical for compliance, as it allows tracking and auditing of access and actions on resources.

- Managed Identity Support: RBAC works seamlessly with Managed Identities (used by Azure VMs, App Services, etc.), allowing secure and managed access to Azure resources without any credentials in the code.


#### Authentication using Access Keys (Resource -> Security + Networking -> Access Keys -> Connection String)
Below program uses connection string for authentication against a storage account
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

### Applications authentication via App objects

Step 1: Register your application. This will create an application object that application will use to connect to a Resource (storage account here)<br>

Go to MS Entra ID -> Manage -> App registrations -> New REgistration and fill in the details to create an app object<br>
You can note down its app/client id and directory/tenant id<br>
YOu also need a "secret". Your App Object -> Manage -> Goto Certificates & Secret, and generate a new secret key (basically a string). Copy the "Value" of the secret immediately (not teh SecretID). Once you navigae away from page you cant see the "Value" again and you will have to generate another secret.<br>

Make sure to give access to your app object by going back to resource (say Storage account here) and assign it to a blob reader role.  Storage Resource -> Access Control (IAM) -> Role Assignments. To give certain access click on "Add" -> "Add Role Assignment" and then in "Job function role" search for "Blob" and you will see blob related builtin role. Select "Blob reader role" and then add app object to that role.

```c#
using Azure.Identity;
using Azure.Storage.Blobs;

string containerName="data";
string fileName="script01.ps1";
string path=@"C:\tmp4\script01.ps1";
string tenantId="38dbefc3-d57f-4955-b62c-1406e16a4ea8"; 
string clientId="72ee1803-08cf-4c19-a334-8405e09a242c";
string secret="ywo8Q~iDGv1b1.Uk_CkXbLUw2G4YNLbWUo5s4b1T";
string storageAccountName="appstore55455344243";
string blobUri=$"https://{storageAccountName}.blob.core.windows.net/{containerName}/{fileName}";

ClientSecretCredential clientSecretCredential=new ClientSecretCredential(tenantId,clientId,secret);
BlobClient blobClient= new BlobClient(new Uri(blobUri),clientSecretCredential);

await blobClient.DownloadToAsync(path);
```

#### MS Graph APIs

To use MS Graph APIs to access certain outlook content, we can use the concept of app objects again. Create a new app object via MS Entra ID -> Manage -> App registrations. <br>
However, instead of giving RBAC to the app object for a resource we need to give it API Permissions via "Your App Object created above" -> Manage -> API Permissions.<br>
Click "Add a permission" -> Select "Microsoft Graph" and type of permission as "Application Permissions" and then add needed permissions for app object.<br>

Application permissions => Your application runs as a background service or daemon without a signed-in user.<br>

Now the app object is in place, it can be used to access the permissioned outlook content. The next step is now to get the access token. The entire flow is based on OAuth. We send the below request by first populating it with client_id/secret from the app object.

[https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow
](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow)
```
POST /{tenant}/oauth2/v2.0/token HTTP/1.1           //Line breaks for clarity
Host: login.microsoftonline.com:443
Content-Type: application/x-www-form-urlencoded

client_id=00001111-aaaa-2222-bbbb-3333cccc4444
&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default
&client_secret=qWgdYAmab0YSkuL1qKv5bPX
&grant_type=client_credentials
```
This request will return a token that will be used in API http headers to give access to Graph API. (https://learn.microsoft.com/en-us/graph/api/overview?view=graph-rest-1.0)

There are two steps involved in essence:<br><br>
 <strong>STEP 1: Get the OAuth 2.0 token - https://login.microsoftonline.com/1a738633-1476-4d96-828c-fa5727a31ca7/oauth2/v2.0/token</strong><br>
 <img src="./images/gettoken.PNG" alt="drawing" width="100%"/>
 <br><br>
 1a738633-1476-4d96-828c-fa5727a31ca7 in the URL is Entra ID as in SS below:<br><br>
<img src="./images/entraid.PNG" alt="drawing" width="100%"/><br><br>

client id comes from the app registered in Entra:<br><br>
<img src="./images/appreg.PNG" alt="drawing" width="100%"/><br><br>

 <strong>STEP 2: Make the API calls - https://graph.microsoft.com/v1.0/users</strong><br>
 <img src="./images/getusers.PNG" alt="drawing" width="100%"/>

## Key Vault

Can be used to store Encryption Keys, Certificates, and Secrets.<br>

Access: To access vault you can use RBAC or a vault access policy<br>

#### Encryption Keys
Before creating keys, we have to use RBAC and use the role of "Key Vault Administrator". Add members to it and those members can then add secrets or keys etc to the vault via Access Control (IAM).<br>

To use the keys from the vault, one can register an app object and use it to access the vault from the applications. Give the app object permissions by using the existing role of "Key Vault Crypto User" in the Key Vault as we are going to use an encryption key from the vault in the example code.

```c#
// We first need to add the package
// dotnet add package Azure.Security.KeyVault.Keys , dotnet add package Azure.Identity
// Then use an existing client application object access to the key vault via RBAC

// We will make use of the same Application Object concept

using System.Text;
using Azure.Identity;
using Azure.Security.KeyVault.Keys;
using Azure.Security.KeyVault.Keys.Cryptography;

//following 3 are from the registered app object with Entra
string clientId="da92e330-5968-4593-b3b5-f070fd6aaeee";
string tenantId="70c0f6d9-7f3b-4425-a6b6-09b47643ec58";
string clientSecret="9Hi8Q~t.0skdyQR-.aP9.4TSdpjDVWUZ4hDiecj-";

string keyVaultURI="https://appvault2939949.vault.azure.net/";
string keyName="appkey"; //<=== This is created in the Key Vault to store encryption key

// We need to use the KeyClient to get a key first
ClientSecretCredential clientSecretCredential=new ClientSecretCredential(tenantId, clientId,clientSecret);

KeyClient keyClient=new KeyClient(new Uri(keyVaultURI),clientSecretCredential);

var key=keyClient.GetKey(keyName);
Console.WriteLine($"Got a handle to the key {key.Value.Name}");

// Next we need to use the CryptographyClient class to perform cryptographic operations

string toEncrypt="This is a string we want to encrypt";
var cryptoClient = keyClient.GetCryptographyClient(key.Value.Name, key.Value.Properties.Version);

byte[] plainText = Encoding.UTF8.GetBytes(toEncrypt);

EncryptResult encryptResult = cryptoClient.Encrypt(EncryptionAlgorithm.RsaOaep, plainText);

Console.WriteLine($"Encrypted text {Convert.ToBase64String(encryptResult.Ciphertext)}");

// If you want to decrypt the text

byte[] ciperToBytes = encryptResult.Ciphertext;
var decryptedText=cryptoClient.Decrypt(EncryptionAlgorithm.RsaOaep,encryptResult.Ciphertext);

Console.WriteLine(Encoding.UTF8.GetString(decryptedText.Plaintext));
```

#### Secrets (say a DB pwd).
Create a secret with name say dbpassword and store the passowrd in vault. Search for permissions with word "secret" and use "Key Vault Secrets User" role and assign app object to it.
```c#
// We will make use of the same Application Object concept

using System.Text;
using Azure.Identity;
using Azure.Security.KeyVault.Keys;
using Azure.Security.KeyVault.Keys.Cryptography;

string clientId="da92e330-5968-4593-b3b5-f070fd6aaeee";
string tenantId="70c0f6d9-7f3b-4425-a6b6-09b47643ec58";
string clientSecret="9Hi8Q~t.0skdyQR-.aP9.4TSdpjDVWUZ4hDiecj-";
string keyVaultURI="https://appvault2939949.vault.azure.net/";
string keyName="appkey";

// We need to use the KeyClient to get a key first

ClientSecretCredential clientSecretCredential=new ClientSecretCredential(tenantId, clientId,clientSecret);

KeyClient keyClient=new KeyClient(new Uri(keyVaultURI),clientSecretCredential);

var key=keyClient.GetKey(keyName);
Console.WriteLine($"Got a handle to the key {key.Value.Name}");

// Next we need to use the CryptographyClient class to perform cryptographic operations

string toEncrypt="This is a string we want to encrypt";
var cryptoClient = keyClient.GetCryptographyClient(key.Value.Name, key.Value.Properties.Version);

byte[] plainText = Encoding.UTF8.GetBytes(toEncrypt);

EncryptResult encryptResult = cryptoClient.Encrypt(EncryptionAlgorithm.RsaOaep, plainText);

Console.WriteLine($"Encrypted text {Convert.ToBase64String(encryptResult.Ciphertext)}");

// If you want to decrypt the text

byte[] ciperToBytes = encryptResult.Ciphertext;
var decryptedText=cryptoClient.Decrypt(EncryptionAlgorithm.RsaOaep,encryptResult.Ciphertext);

Console.WriteLine(Encoding.UTF8.GetString(decryptedText.Plaintext));
```

## Managed Identities
<strong>NOTE: As you see we are still embedding some keys in code such as client id, tenant id and app object secret. Work around is Managed Identitied.</strong>

Gives a way for apps to authenticate without embedding credentials. Managed Ids are available for VMs, Web Apps, and Azure Functions. Managed Id for resource can be added to Entra Id and then used to assign permissions via RBAC.

#### Managed Identity (MI) on a VM
If you go to your VM and click on "Security" -> "Identity" on sidebar, there you can enable a System assigned Managed Identity (MD). This will register the VM in MS ENtra ID. From "Security" -> "Identity", you can now copy the Object (principal) ID of the VM and see role assignments.

Now go to Azure Storage Account -> Access Control (IAM) -> Role Assignment, select generic Blob Data Reader role and then click on "Managed Identity", under select members you will see your VM, select the VM and permission will be assigned to it.

Any hosted app on that VM will now be able to access the storage account without using credentials (use DefaultAzureCredential()).

```c#
using Azure.Identity;
using Azure.Storage.Blobs;

string containerName="data";
string fileName="script01.ps1";
string path=@"C:\tmp4\script01.ps1";

string storageAccountName="yourappstore12345";
string blobUri=$"https://{storageAccountName}.blob.core.windows.net/{containerName}/{fileName}";

BlobClient blobClient= new BlobClient(new Uri(blobUri),new DefaultAzureCredential());

await blobClient.DownloadToAsync(path);
```
Under the hood, the API gets the access token for VM identity from ENtra ID and use that for authorizations against stoarge account.

<strong>How to use managed identities for Azure resources on an Azure VM to acquire an access token</strong>
https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/how-to-use-vm-token
Managed identities for Azure resources provide Azure services with an automatically managed identity in Microsoft Entra ID. You can use this identity to authenticate to any service that supports Microsoft Entra authentication, without having credentials in your code.

This article provides various code and script examples for token acquisition. It also contains guidance about handling token expiration and HTTP errors.

<strong>Getting the Access Token (OAuth endpoint)</strong>
```c#
string tokenUri="http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://storage.azure.com/";
HttpClient httpClient=new HttpClient();
httpClient.DefaultRequestHeaders.Add("Metadata","true");

HttpResponseMessage responseMessage=await httpClient.GetAsync(tokenUri);

string content=await responseMessage.Content.ReadAsStringAsync();

Console.WriteLine(content);
```

<strong>Using the Access Token from previous step to access the Blob data</strong>
```c#
using System.Net.Http.Headers;
using System.Text.Json;

string tokenUri="http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://storage.azure.com/";
HttpClient httpClient=new HttpClient();
httpClient.DefaultRequestHeaders.Add("Metadata","true");

HttpResponseMessage responseMessage=await httpClient.GetAsync(tokenUri);

string content=await responseMessage.Content.ReadAsStringAsync();

Dictionary<string,string>? values=JsonSerializer.Deserialize<Dictionary<string,string>>(content);

foreach(KeyValuePair<string,string> pair in values)
{
    Console.WriteLine($"Key : {pair.Key}");
    Console.WriteLine($"Value : {pair.Value}");
}

string containerName="data";
string fileName="script01.ps1";

string storageAccountName="appstore55455344243";
string blobUri=$"https://{storageAccountName}.blob.core.windows.net/{containerName}/{fileName}";

HttpClient client= new HttpClient();
client.DefaultRequestHeaders.Add("x-ms-version","2024-05-04");
client.DefaultRequestHeaders.Authorization= new AuthenticationHeaderValue("Bearer",values["access_token"]);

HttpResponseMessage message=await client.GetAsync(blobUri);
string blobContent=await message.Content.ReadAsStringAsync();

Console.WriteLine(blobContent);
```

### System Assigned vs User Assigned.
When we enabled identity, we enabled "System" assigned MI. Under "Security" -> "Identity", there are two options "System" and "User". For User, we can create a "User Assigned Managed Identity" as a new resource and attach it to a VM or Azure Function, that way each VM or a Azure Function wont get a system id created. If a VM is deleted then that system id will also go away but in case of a user assigned id, it will persist and can be assigned to another VM/Azure Function. It can be attached to multiple resources.

### Azure Function
```c#
using Azure.Identity;
using Azure.Storage.Blobs;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

namespace Company.Function
{
    public class apptrigger
    {
        private readonly ILogger<apptrigger> _logger;

        public apptrigger(ILogger<apptrigger> logger)
        {
            _logger = logger;
        }

        [Function("apptrigger")]
        public IActionResult Run([HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequest req)
        {
            string containerName="data";
            string fileName="script01.ps1";            

            string storageAccountName="appstore55455344243";
            string blobUri=$"https://{storageAccountName}.blob.core.windows.net/{containerName}/{fileName}";


             BlobClient blobClient= new BlobClient(new Uri(blobUri),new DefaultAzureCredential());
             
             var response=blobClient.DownloadContent();
             string blobContent=response.Value.Content.ToString();

             return new OkObjectResult(blobContent);
        }
    }
}
```

### MI and Key Vault
Same way a user can go to Key Vault and assign MI of a particular function or VM key vault access permissions and then access the secret.

Go bacn to Azure Function app, and in the Settings -> Environment Variables you can define "APPSECRET" which is a secret name as Key and secret URI as value. Using this URI we will then access the secret value from the vault.

The code will look like this:
```c#
        [Function("apptrigger")]
        public IActionResult Run([HttpTrigger(AuthorizationLevel.Function, "get", "post")] HttpRequest req)
        {
            string secretReference="APPSECRET";
            string? value=System.Environment.GetEnvironmentVariable(secretReference, EnvironmentVariableTarget.Process);
            return new OkObjectResult(value);
        }
```

### Shared Access Signatures (SAS) Tokens
Under a resource (say a Storage Account), "Security + Networking", there is another item "Shared Access Signatures", using this you can generate a connection string that has more granular limits on access and use that connection string to access a resource say a storage account.

Description: SAS tokens provide temporary, scoped access to specific resources (e.g., blob, file, table) for a limited time. The token can be configured with fine-grained permissions, like read-only or write-only access.
Use Case: Useful when you need to grant limited, time-bound access to certain storage resources without exposing the full storage account.



# Section 13: Event and Message based solutions, API management

- Storage Queue - Simple message queue
- Azure Messaging Bus - Enterprise messaging platform
- Event Hubs - Here you ingest event data in real time
- Event Grid - build event driven applications

### Storage Queue
<img src="./images/stqueue.PNG" alt="drawing" width="50%"/>

Create a new queue "appqueue" as shown in SS:<br>
<img src="./images/appqueue.PNG" alt="drawing" width="50%"/>

#### Post Message:
The following script can be used to post messages in this Q:
```c#
// First add the package - dotnet add package Azure.Storage.Queues
using Azure.Storage.Queues;

string queueName="appqueue";
string connectionString="DefaultEndpointsProtocol=https;AccountName=appstore443434323;AccountKey=k+d/Y8Dd4ZVaEFyNoy7u0I86ieqB/yPk/lXMwiKjUH3HM8JsD4s0En3KvSjiXCxnXjdkknggD5/X+AStl9yTWA==;EndpointSuffix=core.windows.net";

for(int i=0;i<5;i++)
await AddMessage($"Test Message {i}");

async Task AddMessage(string message)
{
    QueueClient queueClient=new QueueClient(connectionString,queueName);
    await queueClient.SendMessageAsync(message);
    Console.WriteLine($"Message added to queue - {message}");
}
```

#### Peek Message - look at message but do NOT delete it
```c#
async Task PeekMessage(int messageCount)
{
    QueueClient queueClient=new QueueClient(connectionString,queueName);
    PeekedMessage[] messages=await queueClient.PeekMessagesAsync(maxMessages: messageCount);

    foreach(PeekedMessage message in messages)
    {
    Console.WriteLine($"Message ID #{message.MessageId}");
    Console.WriteLine($"Message Body #{message.Body}");
    }
}
```

#### Recieve Message - delete the message after processing it
```c#
async Task ReceiveMessage(int messageCount)
{
    QueueClient queueClient=new QueueClient(connectionString,queueName);
    QueueMessage[] messages=await queueClient.ReceiveMessagesAsync(maxMessages: messageCount);
    // Receiving the messages will make the messages invisible 
    // During this time you can process the messages
    // To delete the message , we need to delete them specifically

    // This value is required to delete the Message. If deletion fails using this popreceipt then the message has been dequeued by another client.
    foreach(QueueMessage message in messages)
    {
      Console.WriteLine($"Deleting message with Message ID #{message.MessageId}");
      await queueClient.DeleteMessageAsync(message.MessageId,message.PopReceipt);
    }
}
```

#### Get number of messages in the Q
```c#
int GetQueueLength()
{
    QueueClient queueClient=new QueueClient(connectionString,queueName);
    return  queueClient.GetProperties().Value.ApproximateMessagesCount;
}
```

#### Azure Queue Storage Trigger
We can create a Azure Function using template "Azure Queue Storage Trigger" in VS code. Note that if the code cant process the message and fails, it can create another queue say appqueue-poison and move the errorig messages to that Q. Q Trigger can also use your custom input classes to recieve the data.
```c#
using System;
using Azure.Storage.Queues.Models;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;

namespace Common.Services
{
    public class QueueTrigger
    {
        private readonly ILogger<QueueTrigger> _logger;

        public QueueTrigger(ILogger<QueueTrigger> logger)
        {
            _logger = logger;
        }

        [Function(nameof(QueueTrigger))]
        public void Run([QueueTrigger("appqueue", Connection = "62c5fc_STORAGE")] QueueMessage message)
        {
            _logger.LogInformation($"C# Queue trigger function processed: {message.MessageText}");
        }
    }
}
```

#### Output to Storage Table: output of azure q storage trigger can be stored in Q Table:
```c#
public class TableOrder
{
    public string? PartitionKey { get; set; }
    public string? RowKey { get; set; }
    public decimal Price { get; set;}
}

// We need to add the package
// dotnet add package Microsoft.Azure.Functions.Worker.Extensions.Tables
[Function(nameof(QueueTrigger))]
[TableOutput("Orders",Connection ="62c5fc_STORAGE")]
public object Run([QueueTrigger("appqueue", Connection = "62c5fc_STORAGE")] Order order)
{
    TableOrder tableOrder= new TableOrder()
    {
        PartitionKey=order.CourseName,
        RowKey=$"Order{order.Id}",
        Price=order.Price
    };

   _logger.LogInformation("Added to table");
   return tableOrder;
}
```

### Azure Service Bus - Fully managed Enterprise Service Broker; msg can be sent to service bus queues or to service bus topics
<img src="./images/azServiceBus.PNG" alt="drawing" width="50%"/>


