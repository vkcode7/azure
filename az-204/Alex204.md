## Section 10: Implementing Azure Security and Auth

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
