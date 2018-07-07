+++
author = "admin"
date = 2013-05-21T12:58:56Z
description = ""
draft = false
slug = "csharp-unit-testing-classes-with-httpclient-dependence-using-autofixture"
title = "C# - Unit testing classes with HttpClient dependence + using Autofixture"

+++


The [HttpClient](http://msdn.microsoft.com/en-us/library/system.net.http.httpclient.aspx) added in the .NET framework 4.5 is simply put awesome in combination with the new async/await in C# 5.0. Also if you are doing Phone development or creating a PCL that targets a platform where HttpClient is not available, do yourself a favour and [get it from NuGet](http://blogs.msdn.com/b/bclteam/archive/2013/02/18/portable-httpclient-for-net-framework-and-windows-phone.aspx).

Im gonna go through a way of unit testing code that is dependent on HttpClient, e.g. for retrieving data from a service. It is a bit dirty, as we cannot use e.g. Moq because we are not working with interfaces and members are non-virtual, thus we cannot fully mock/stub the class. Thus, we cannot do something like this:
```csharp
var mock = new Mock<HttpClient>(); 
mock.Setup(x => x.GetStringAsync(It.IsAny<string>()))
    .Returns(() => SomeMockResponse);
```

Fortunately an HttpClient can be parameterised with a custom handler for sending requests. A typical approach is to wrap the HttpClient in some implementation of an interface that closely resembles or is a subset of HttpClient. Similarly the System.Web.Http. HttpServer can also be used to setup a host in-memory for testing.

In this case we use fake objects to enable the unit testing. In the end, we will do a version using Autofixture to automatically arrange the setup. It is based on [this approach by Pablo Cibraro](http://weblogs.asp.net/cibrax/archive/2012/09/12/unit-and-integration-testing-with-the-web-api-httpclient.aspx) but im gonna take it a bit further to allow code that e.g. reads from a server to be tested through GetStringAsync.

The HttpClient has a constructor that takes a single argument of the type HttpMessageHandler that governs how the client sends requests. The fake HttpMessageHandler can be done similarly to the one provided by Pablo Cibraro in his post.
```csharp
public class FakeHttpMessageHandler : HttpMessageHandler 
{ 
    private HttpResponseMessage response; 

    public FakeHttpMessageHandler(HttpResponseMessage response) 
    { 
        this.response = response; 
    } 

    protected override Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken cancellationToken) 
    { 
        var responseTask = new TaskCompletionSource<HttpResponseMessage>();
        responseTask.SetResult(response);

        return responseTask.Task;
    } 
}
```
Now we can specify a test like this where we inject the test-ready HttpClient into the SUT that is of some type HttpConsumer.
```csharp
[TestMethod]
public void TestGetContents()
{ 
    var responseMessage = new HttpResponseMessage();
    var messageHandler = new FakeHttpMessageHandler(responseMessage);
    var client = new HttpClient(messageHandler);
    var sut = new HttpConsumer(client); 

    sut.FetchContentsFromRemote(); 

    // Asserts
}
```

At this point we can test our SUT in a number of ways e.g. by setting the status code in the HttpResponseMessage. Now what if the unit being tested performs GetStringAsync using the HttpClient and spits out a list of some elements it parses from the response data. GetStringAsync uses HttpResponseMessage.Content to get the response data where the Content property is of the abstract type HttpContent. We make a fake derived type of HttpContent that can return test specific response data.

```csharp
public class FakeHttpContent : HttpContent 
{ 
    public string Content { get; set; } 

    public FakeHttpContent(string content) 
    { 
        Content = content; 
    } 

    protected async override Task SerializeToStreamAsync(Stream stream, TransportContext context)
    {
        byte[] byteArray = Encoding.ASCII.GetBytes(Content); 
        await stream.WriteAsync(byteArray, 0, Content.Length); 
    } 

    protected override bool TryComputeLength(out long length) 
    { 
        length = Content.Length;
        return true; 
    } 
}
```

Using this we can now test methods that reads data using GetStringAsync in the ways we want.
```csharp
[TestMethod]
public void FetchCarsFromRemoteShouldParseCorrectly()
{ 
    var responseMessage = new HttpResponseMessage(); 
    responseMessage.Content = new FakeHttpContent(GenerateJsonArrayOfCarModelsString()); 
    var messageHandler = new FakeHttpMessageHandler(responseMessage); 
    var client = new HttpClient(messageHandler); 
    var sut = new HttpConsumer(client); 

    IEnumerable<Cars> = sut.FetchCarsFromRemote(); 

    // Asserts 
}
```
Now lets change it to use [Autofixture](https://github.com/AutoFixture/AutoFixture). The benefit in this case is not significant as it requires some bit of configuration of the Fixture but the use of Autofixtures support for data theories with xUnit makes it a little better.

First we need to specify a strategy such that Autofixture selects the constructor of HttpClient that takes a single HttpMessageHandler argument. We create a strategy that always chooses the constructor that takes a single argument:

```csharp
public class SingleParameterConstructorQuery : IMethodQuery 
{ 
    public IEnumerable<IMethod> SelectMethods(Type type) 
    { 
        return from m in type.GetConstructors() 
        let parameters = m.GetParameters() 
        where parameters.Length == 1 
        select new ConstructorMethod(m) as IMethod; 
    } 
}
```

Now we create a customization that sets up a fixture for using the strategy when creating a new HttpClient instance:
```csharp
public class HttpClientCustomization : ICustomization 
{ 
    public void Customize(IFixture fixture) 
    { 
        fixture.Customize<HttpClient>(c => c.FromFactory(new MethodInvoker(new SingleParameterConstructorQuery()))); 
    } 
}
```
Before we head back to the testing, we create the custom attribute that initialises the test with the customization like this:
```csharp
public class AutoDomainDataAttribute : AutoDataAttribute 
{ 
    public AutoDomainDataAttribute() 
        : base(new Fixture().Customize(new HttpClientCustomization())) { } 
}
```
Now we can test something like this:
```csharp
[Theory, AutoDomainData] 
public void FetchCarsShouldParseCorrectly(
    [Frozen(As = typeof(HttpContent))] FakeHttpContent httpContent,
    [Frozen]HttpResponseMessage responseMessage, 
    [Frozen(As = typeof(HttpMessageHandler))] 
    FakeHttpMessageHandler messageHandler, 
    HttpConsumer sut) 
{ 
    responseMessage.StatusCode = HttpStatusCode.OK; 
    httpContent.Content = GenerateJsonArrayOfCarModelsString(); 
    IEnumerable<Cars> = sut.FetchCarsFromRemote(); 

    //Asserts 
}
```
Note that GetStringAync() will throw an exception if the status code is not OK (200) . This is set by default, but Autofixture initialises this to the first value “Continue” (100).

In the end – quite some (dirty) work to be able to test methods that are dependent on HttpClient. Personally, I prefer to take the other direction of wrapping HttpClient and let classes that gets or sends data communicate through this. However, some time it may be necessary to test methods that directly depend on HttpClient, and then its nice to be able to control the testing as presented. That being said, I encourage you to also look at [System.Web.Http.HttpServer](http://msdn.microsoft.com/en-us/library/system.web.http.httpserver(v=vs.108).aspx) before going down this route.

 


