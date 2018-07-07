+++
author = "admin"
date = 2014-12-03T19:52:02Z
description = ""
draft = false
slug = "capptain-or-app-telemetry-product-choice-xamarin-forms"
title = "Capptain (or your app telemetry service of choice) with Xamarin Forms"

+++


We are increasingly developing applications targeting both iOS and Windows Phone using Xamarin Forms. In this case, when it comes to telemetry, we are interested in putting as much as possible in the shared code. Here is a way to incorporate [Capptain](http://www.capptain.com/ "Capptain") into a Xamarin Forms app using DependencyService and a proxy on each platform. You can use the same approach with other telemetry services, as long as a native SDK is provided for the platforms you target.

*[Xamarin Insights](https://insights.xamarin.com/ "Xamarin Insights") is currently in preview, but looks very promising. You might want to check it out first and see if it suits your needs.*


## Define the Interface in the Shared Project / PCL

First, the interface through which the shared code will use, is defined in the shared project or PCL depending on which approach you are using in the Xamarin Forms solution. The interface can be as simple as a one-to-one mapping to the desired methods you wish to use on the original interface exposed in the Capptain SDK. Here is an example that supports registering activities and events:

```csharp
public interface ITelemetryAgent 
{ 
    void StartActivity(string name, Dictionary<object, object> extras = null); 
    void SendEvent(string name, Dictionary<object, object> extras = null); 
}
```


## Provide Platform Specific Implementations

Next we provide the platform specific implementations which basically just forwards a call to the SDK. Here it is for Windows Phone (install the Capptain NuGet package into the Windows Phone solution).:

```csharp
[assembly: Dependency(typeof(CapptainAgentProxy))] 
namespace XamarinApp.WinPhone 
{ 
    public class CapptainAgentWpProxy : ITelemetryAgent 
    { 
        private CapptainAgent agent = CapptainAgent.Instance;
        public void StartActivity(string name, Dictionary<object, object> extras = null) 
        { 
            this.agent.StartActivity(name, extras); 
        } 

        public void SendEvent(string name, Dictionary<object, object> extras = null) 
        { 
            this.agent.SendEvent(name, extras); 
        } 
    } 
}
```
For Android and iOS it is a little more tricky, since we will need to provide bindings to the Java and Objective-C libraries.

For Android, download the Capptain JAR file and use add it to a binding project with Build Action set to EmbeddedJar. Some errors will show up – I was able to handle this by removing the class CapptainNativePushToken (you will not be able to use the push features of Capptain) by adding the following to the Meta file:

```
<remove-node path="/api/package[@name='com.ubikod.capptain']
/class[@name='CapptainNativePushToken']" />
```
The Android proxy class follows the same approach as the Windows Phone one, except any additional data you send along is provided in a Bundle on the Android platform. You will have to make a convertion from the dictionary to a bundle (I wrote an extension method for this).
```csharp
[assembly: Xamarin.Forms.Dependency(typeof(CapptainAgentAndroidProxy))]
namespace XamarinSample.Droid 
{ 
    internal class CapptainAgentAndroidProxy : ITelemetryAgent 
    { 
        private CapptainAgent agent; 
        
        public CapptainAgentAndroidProxy() 
        { 
            this.agent = CapptainAgent.GetInstance(Forms.Context);
        } 

        public void StartActivity(string name, Dictionary<object, object> extras = null) 
        { 
            Bundle bundle = extras == null ? extras.AsBundle() : null; 
            this.agent.StartActivity((Activity)Forms.Context, name, bundle); 
        } 

        public void SendEvent(string name, Dictionary<object, object> extras = null) 
        { 
            Bundle bundle = extras == null ? extras.AsBundle() : null; 
            this.agent.SendEvent(name, bundle); 
        } 
}
```

## Wire Up Platform Specific Configuration

Telemetry services usually require some configuration initialization and event wiring in app life cycle events. For Windows Phone we add the following in App.xaml.cs to Application_Launching and Application_Activated:
```csharp
private void Application_Launching(object sender, LaunchingEventArgs e) 
{ 
    CapptainAgent.Instance.Init(); 
} 

private void Application_Activated(object sender, ActivatedEventArgs e) 
{ 
    CapptainAgent.Instance.OnActivated(e); 
}
```
For Android this should suffice in the MainActivity class (remember setting up the manifest file as described in the documentation):
```csharp
protected override void OnPause() 
{ 
    base.OnPause(); 
    CapptainAgent.GetInstance(this).EndActivity(); 
}
```

## Use DependencyService to Access Implementations

And now, finally in our Xamarin Forms page we can start shipping data about usage or errors:

```csharp
var agent = DependencyService.Get<ITelemetryAgent>(); agent.SendEvent("Purchase");
```
### Automatically Register Each Xamarin Forms Page as an Activity

If you want to track navigating between pages in general an easy way is to subclass e.g. ContentPage and wire up some activity tracking:
```csharp
public class CapptainContentPage : ContentPage 
{ 
    private ITelemetryAgent telemetryAgent = DependencyService.Get<ITelemetryAgent>(); 

    protected override void OnAppearing() 
    { 
        base.OnAppearing();
        this.telemetryAgent.StartActivity(this.GetType().Name); 
    } 

    protected virtual string GetPageActivityName() 
    { 
        return this.GetType().Name; 
    } 
}
```

And then let your page inherit from CapptainContentPage instead of ContentPage. By default this will register the activity using the type name, but you can override the GetPageActivityName method in your page to change this.


