+++
author = "admin"
categories = [".NET", "C#", "ASP.NET"]
date = 2015-02-12T19:47:56Z
description = ""
draft = false
slug = "subdomain-based-multitenancy-asp-net-mvc-5"
tags = [".NET", "C#", "ASP.NET"]
title = "Multitenancy with Subdomains in ASP.NET MVC 5"

+++


Developing a product that is multi-tenant is a pretty interesting challenge. Recently I was in the fortunate situation of having to work on such a thing. By giving organisations access to a tenant through a subdomain of choice is a pretty neat approach that provides users a sense of personal ownership of the product. In this post I will show a simple approach of how to use subdomains to identify the corresponding tenant in ASP.NET MVC 5.

First things first – what is the goal. Consider a CRM product, United CRM available on www.unitedcrm.com. ABC Inc. wishes to purchase access to United CRM for its employees. As part of signing up, ABC Inc. chooses the domain www.abc.unitedcrm.com for accessing it’s tenant. Let us see how this might be implemented.


## Extract Tenant in Requests

Whenever you navigate to a site, your browser will add the Host header to the request. Visiting www.news.google.com, will result in the Host header value “news.google.com”. `HttpContext` provides all information about an HTTP request in ASP.NET and can be used to read the headers. Your have a range of options, where in the request pipeline this should be extracted.

### Use a Route Constraint

Using a route constraint has the added benefit of being able to route all tenant specific request where appropriate. The following is a sample route constraint that extracts the subdomain and adds the tenant id to the route values dictionary:

```csharp
public class TenantRouteConstraint : IRouteConstraint 
{ 
    public bool Match(HttpContextBase httpContext, Route route, string parameterName, RouteValueDictionary values, RouteDirection routeDirection)
    { 
        var fullAddress = httpContext.Request.Headers["Host"].Split('.'); 
        if (fullAddress.Length < 2) 
        { 
            return false; 
        } 

        var tenantSubdomain = fullAddress[0]; 
        var tenantId = ... // Lookup tenant id (preferably use a cache) 

        if (!values.ContainsKey("tenant")) 
        { 
            values.Add("tenant", tenantId); 
        } 

        return true; 
    } 
}
```
Add the route constraint when creating routes:

```csharp
routes.MapRoute(
            name: "Default", url: "{controller}/{action}/{id}",
            defaults: new { controller = "Home", action = "Index", id = UrlParameter.Optional },
            constraints: new { TenantAccess = new TenantRouteConstraint() } 
);
```
The tenant id can then be extracted in controllers that end up handling the request:

```csharp
public ActionResult Index() 
{ 
    var tenantId = this.RouteData.Values["tenant"]; // Get some tenant specific data 
    return View(); 
}
```

Note that routing is used may be used several times for a single a request, specifically when Razor needs to resolve routes.

### Use an Action Filter

Using a custom action filter, we add logic to extract the tenant before controller actions execute:

```csharp
public class TenantActionFilter : ActionFilterAttribute, IActionFilter 
{ 
    public void OnActionExecuting(ActionExecutingContext filterContext) 
    { 
        var fullAddress = filterContext.HttpContext.Request.Headers["Host"].Split('.'); 
        if (fullAddress.Length < 2) 
        { 
            filterContext.Result = new HttpStatusCodeResult(404); //or redirect filterContext.Result = new RedirectToRouteResult(..);
        } 
        
        var tenantSubdomain = fullAddress[0]; 
        
        // Lookup tenant id (preferably use a cache) 
        var tenantId = ... filterContext.RouteData.Values.Add("tenant", tenantId);
        base.OnActionExecuting(filterContext); 
    } 
}
```
And extracting the tenant is done in the same way as with a route constraint:

```csharp
[TenantActionFilter]
public ActionResult Index() 
{ 
    var tenantId = this.RouteData.Values["tenant"]; // Get some tenant specific data 
    return View(); 
}
```
### Use a Base Controller

Finally, using a base controller is more or less exactly the same as the custom action filter, as MVC Controllers implements `IActionFilter`:

```csharp
public class TenantController : Controller 
{ 
    protected override void OnActionExecuted(ActionExecutedContext filterContext) 
    { 
        // Same as action filter 
    } 
}
```

## Enabling Subdomain Testing for Localhost

So far so good – we can identify the tenant a user is trying to access. In order to test it on a development machine, a couple of small additions needs to be done to allow subdomains with localhost (e.g. tenant1.localhost) together with IIS Express:

1. Go to project `Properties` , `Web` and hit `Create Virtual Directory` if the solution is completely new. This will add a site and bindings to the IIS Express application config file (or just run the solution once to make sure its added).
2. Open the IIS Express application config file located in folder `%USERPROFILE%\My Documents\IISExpress\config\applicationhost.config` – find the project and add a binding to a test subdomain: 
```markup
<bindings> 
    <binding protocol="http" bindingInformation="*:59920:localhost" /> 
    <binding protocol="http" bindingInformation="*:59920:test.localhost" /> 
</bindings>
```
3. Add the following entry to the hosts file: `127.0.0.1 test.localhost`

If you run Visual Studio as Administrator, test.localhost will work at this time. If you are not running Visual Studio as Administrator, you will need to add it to HTTP.sys to allow the traffic:

`netsh http add urlacl url=”http://test.localhost:59920/” user=everyone`


