+++
author = "admin"
date = 2013-05-25T10:17:09Z
description = ""
draft = false
slug = "azure-storage-services-windows-store-wp8"
title = "Azure Storage Services REST API Authentication from Windows Store and WP8 applications"

+++


The Azure Storage Client Library provides some good and easy access to Queue, Blob and Table services. The library requires [ODataLib](http://odata.codeplex.com/) that unfortunately is not available in a version for Store and WP8 yet. The good thing is, we have WCF Data Services based REST API to access the services. The [official documentation](http://msdn.microsoft.com/en-us/library/windowsazure/dd179355) provides details on the available operations, how to authenticate etc. How to authenticate can be a bit troublesome to derive from the documentation when you are not using the storage client library. I decided to put together this walk through that enables you to connect to the storage services using the REST API from Store and WP8 applications.

Note that I will focus on the Table service, but this the same approach for Blob and Queue services, by replacing relevant formats using the official documentation.


## Authentication Basics

All requests must be authenticated by adding two headers to the HTTP request:

* *x-ms-date* – Coordinated Universal Time (UTC) timestamp of the request. The service rejects all requests received that are older than 15 minutes for security reasons  
* *Authorization* – specifies an authentication scheme, account name and a signature constructed from a Hash-based Message Authentication Code (HMAC) computed using SHA256 and Base64 encoding on the request contents

The date header is fairly straightforward. The authorization header has the following format: `Authorization="<Scheme> <AccountName>:<Signature>"`

* *Scheme* can be *SharedKey* or *SharedKeyLite* – the shared key lite scheme is smaller in the contents that is hashed, and is the scheme required if you want to access the Table Service via. WCF Data Services, so we will go with this one here.</span>
* *AccountName* is your Azure storage name
* *Signature* is the constructed HMAC from a signature string in UTF8 encoding

Basically, the signature is constructed in the following way (from the official documentation): `Signature = Base64(HMAC-SHA256(UTF8(SignatureString)))`

The signature string used to construct the signature varies depending on the type of service. Blob and queue services signatures are constructed using the same format, and the table service requires its own signature format. You can see these formats in the [original documentation](http://msdn.microsoft.com/en-us/library/windowsazure/dd179355) for both Shared Key and Shared Key Lite schemes. If we look up the signature format for the Shared Key Lite scheme for Table Service, the documentation states the following format should be hashed:

`SignatureString = Date + "\n" CanonicalizedResource`

The *CanonicalizedResource* portion represents the resource targeted in the service. As with the general signature string format, this portion has the same format for blob and queue service requests, and Table service has its own format. Furthermore, for Blob and Queue service this portion is constructed differently for the Shared Key and Shared Key Lite schemes. For Table service the general rule is:

`CanonicalizedResource = "/<accountname>/<uripath>"`

Let's construct a signature string for an authorization header used to request some data.

[documentation](http://msdn.microsoft.com/en-us/library/windowsazure/dd179423.aspx) states that we can get an enumeration of all current Tables in the storage using a GET request using the request URI *https://myaccount.table.core.windows.net/Tables*<. The signature string will be the following:

`SignatureString = "Sat, 25 May 2013 15:50:20\n/myaccount/Tables"`

The (almost complete) headers that we want to add to the request is hereby:
```
x-ms-date = "Sat, 25 May 2013 15:50:20" 
Authorization = "SharedKeyLite myaccount:Base64(HMAC-SHA256(SignatureString))"
```

Lets see how we can construct the HMACSHA256 signature from the signature string and query the  REST API.


## Azure Storage REST API Authentication in Windows Store, WP8 and Portable Class Library

Calculating the HMAC256 is done differently depending whether it is a Store or a WP8 application or maybe a Portable Class Library that should support both. Let us assume we do a PCL, and wish to support Azure storage services in both platforms, that way we come across both. The goal is to show how to use the API, so I omit any logic that provides abstractions of the responses received. Thus we are just interested in getting the raw XML responses at this point. We will go and fetch a list of all tables in the Table storage at the given moment through the previously stated URI.

Create a new solution with a PCL project, a Windows Store project and a WP8 project. We will do requests using HttpClient  that is available on NuGet ([see this post](http://blogs.msdn.com/b/bclteam/archive/2013/02/18/portable-httpclient-for-net-framework-and-windows-phone.aspx)), that can be added to a PCL (or WP8 specific project.). As a result we may have a solution that looks like this:

* AzureStorageSolution 
 * AzureStorage.WP8
 * AzureStorage.Store
 * AzureStorage.PCL

Add the following interface to the PCL that allows the clients to get a raw response of the tables in the storage:

```csharp
public interface IAzureStorageService 
{
    Task<String> GetRawTablesEnumerationAsync(); 
}
```

In a real solution we would most likely return some constructed model based on the raw response, but for this purpose we just provide the clients with the response XML directly.

We will delegate the actual computation of the signature to the Phone and Store applications. Add the following interface to the PCL that will be implemented by each:

```csharp
public interface IAzureAuthSignatureComputeStrategy 
{ 
    string GetTableServiceSignature(string uriPath); 
}
```

The uriPath argument will take the *CanonicalizedResource *portion of the signature string as input (in a real solution you may want to create this automatically from the URI you are requesting).

Add an implementation of IAzureStorageService that will do the actual request to get the tables in an XML response format:

```csharp
public class AzureStorageService : IAzureStorageService 
{ 
    private readonly IAzureAuthSignatureComputeStrategy signatureComputeStrategy; 
    private const string TableServiceEndpoint = "http://myaccount.table.core.windows.net"; 

    public AzureStorageService(IAzureAuthSignatureComputeStrategy signatureComputeStrategy) 
    { 
        this.signatureComputeStrategy = signatureComputeStrategy; 
} 

    public async Task<string> GetRawTablesEnumerationAsync() 
    { 
        var client = new HttpClient(); 
        string signature = signatureComputeStrategy.GetTableServiceSignature("/myaccount/Tables()"); 
        string authorizationHeader = "SharedKeyLite myaccount:" + signature; 
        client.DefaultRequestHeaders.Add("x-ms-date", DateTime.UtcNow.ToString("R", System.Globalization.CultureInfo.InvariantCulture)); 
        client.DefaultRequestHeaders.Add("Authorization", authorizationHeader); 
        return await client.GetStringAsync(TableServiceEndpoint + "/Tables()"); 
    } 
}
```
If you are doing Store or WP8 specific app all the code until now will work as well (and for Store you will not need the HttpClient from NuGet). Now lets construct the signature in each of the applications and use the PCL to retrieve data.

#### Windows Store

Create the following class that implements the IAzureAuthSignatureComputeStrategy interface in your Store project:

```csharp
public class StoreSignatureComputeStrategy : IAzureAuthSignatureComputeStrategy 
{ 
    public string GetTableServiceSignature(string uriPath) 
    { 
        MacAlgorithmProvider macAlgorithmProvider = MacAlgorithmProvider.OpenAlgorithm(MacAlgorithmNames.HmacSha256); 
        var signatureString = DateTime.UtcNow.ToString("R", System.Globalization.CultureInfo.InvariantCulture) + "\n" + uriPath; 
        var bufferedSignatureString = CryptographicBuffer.ConvertStringToBinary(signatureString, BinaryStringEncoding.Utf8); 
        var key = "<YOUR ACCESS KEY>"; IBuffer bufferedKeyMaterial = CryptographicBuffer.DecodeFromBase64String(key); 
        var hmacKey = macAlgorithmProvider.CreateKey(bufferedKeyMaterial); 
        var bufferedHmac = CryptographicEngine.Sign(hmacKey, bufferedSignatureString); 
        var signature = CryptographicBuffer.EncodeToBase64String(bufferedHmac); 
        return signature; 
    } 
}
```

Note that Line 13 is your access key that you get from the Azure Management Portal. In a real solution, you may want to look for alternatives of how this should be stored.

Finally, we can connect and get the data from the REST API by injecting the compute strategy into the service class in our PCL:

```csharp
protected async override void OnNavigatedTo(NavigationEventArgs e) 
{ 
    var service = new AzureStorageService(new StoreSignatureComputeStrategy()); 
    var tablesResponse = await service.GetRawTablesEnumerationAsync(); 
}
```

####  Windows Phone

Calculating the signature is a bit more straightforward, here is the strategy implementation:

```csharp
public class WpSignatureComputeStrategy : IAzureAuthSignatureComputeStrategy 
{ 
    public string GetTableServiceSignature(string uriPath) 
    { 
        var key = "<YOUR ACCESS KEY>"; 
        var signatureString = DateTime.UtcNow.ToString("R", System.Globalization.CultureInfo.InvariantCulture) + "\n" + uriPath; 
        var generator = new HMACSHA256(Convert.FromBase64String(key)); 
        var signature = Convert.ToBase64String(generator.ComputeHash(System.Text.Encoding.UTF8.GetBytes(signatureString))); 
        return signature; 
    } 
}
```
And the use is exactly the same:

```csharp
protected override void OnNavigatedTo(NavigationEventArgs e) 
{ 
    var service = new AzureStorageService(new WpSignatureComputeStrategy()); 
    var tablesResponse = service.GetRawTablesEnumerationAsync(); 
}
```
Here is an example of a response that shows only the “WADLogsTable” table is created (used by Azure Diagnostics):
```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?> 
<feed 
  xml base="http://myaccount.table.core.windows.net/" 
  xmlns d="http://schemas.microsoft.com/ado/2007/08/dataservices" 
  xmlns:m="http://schemas.microsoft.com/ado/2007/08/dataservices/metadata" 
  xmlns="http://www.w3.org/2005/Atom"> 
    <title type="text">Tables</title> 
    <id>http://myaccount.table.core.windows.net/Tables</id> 
    <updated>2013-05-25T11:00:49Z</updated> 
    <link rel="self" title="Tables" href="Tables" /> 
    <entry> 
        <id>http://myaccount.table.core.windows.net/Tables('WADLogsTable')</id> 
        <title type="text"></title> 
        <updated>2013-05-25T11:00:49Z</updated> 
        <author> 
            <name /> 
        </author> 
        <link rel="edit" title="Tables" href="Tables('WADLogsTable')" /> 
        <category term="myaccount.Tables" scheme="http://schemas.microsoft.com/ado/2007/08/dataservices/scheme" /> 
        <content type="application/xml">
            <m:properties> 
                <d:TableName>WADLogsTable</d:TableName> 
            </m:properties> 
        </content>
    </entry> 
</feed>
```
And there you have it. With this you should be able to use your Azure Storage Services from your Windows Store and/or Windows Phone 8 applications. You should check out the [Azure documentation on Authentication](http://msdn.microsoft.com/en-us/library/windowsazure/dd179428.aspx) to see the signature format you need for Blob and Queue services. Furthermore all the operations available and corresponding URIs can also be found in the [Azure Storage Services REST API reference](http://msdn.microsoft.com/en-us/library/windowsazure/dd179355.aspx).

Finally, if you are up for it, you can get the [OData Client Tools](http://msdn.microsoft.com/en-us/jj658961) for Store and WP8 and use this to provide an abstraction over the REST API access and mapping to entity objects.


