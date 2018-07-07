+++
author = "Jeppe Lund Andersen"
categories = [".NET", "C#", "Xamarin", "Xamarin Forms"]
date = 2015-06-18T15:34:00Z
description = ""
draft = false
slug = "async-camera-pictures-with-xamarin-forms"
tags = [".NET", "C#", "Xamarin", "Xamarin Forms"]
title = "Async Camera Pictures with Xamarin Forms"

+++

I have seen different approaches to using the device camera with Xamarin Forms. Here's an example of how this might be implemented, with an async interface for easy async-await consumption in the shared Xamarin Forms part of the application. I will show the implementation details of the shared code and iOS version. The full example is available on https://github.com/jlandersen/xamarin-forms-camera that includes Windows Phone and Android parts as well. If you are unfamiliar with accessing native features of the different platforms, check out the [official documentation](http://developer.xamarin.com/guides/cross-platform/xamarin-forms/dependency-service/).

## Use
In the shared code of the application, using the camera can be as simple as follows, if you mark the method as async:

```csharp
var cameraProvider = DependencyService.Get<ICameraProvider>();
var pictureResult = await cameraProvider.TakePictureAsync();

if (pictureResult != null)
{
    // Access picture or filename
}
```

## Interface for Shared Code
In the shared part of the application, define the following:

* A class that represents the result of taking a picture, containing e.g. the picture itself and additional information
* The interface, to be implemented on each platform, that produces an instance of the result.

The class holding a picture result may look as follows. The Image property is used to display the taken image on a page. The FileUri property provides the device-specific location of the image for later retrieval. 
```csharp
public class CameraResult
{
    // For displaying the picture on a page
    public ImageSource Image { get; set; }

    // Location on the device where the taken picture is stored
    public string FileUri { get; set; }
}
```

A suitable interface to work against may look as follows:
```csharp
public interface ICameraProvider
{
    Task<CameraResult> TakePictureAsync();
}
```

## iOS Implementation
In the iOS project, ICameraProvider add a class that implements the `ICameraProvider` interface, and register it with the Xamarin Dependency Service using the assembly attribute on the namespace. You should end up with a class similar to the following:
```csharp
[assembly: Dependency(typeof(CameraProvider))]
namespace App.iOS
{
    public class CameraProvider : ICameraProvider
    {
    }
}
```

The general flow when implementing the `TakePictureAsync` method, is to kick off the native camera functionality on iOS and return a task back to the shared code that can be awaited. `TaskCompletionSource` is used to represent the asynchronous operation, of a user interacting with the camera. Once the user is done interacting with the camera and returns to the app, the task will be updated with the result.

Let us look at the general implementation of`TakePictureAsync` in the iOS version:
```csharp

public Task<CameraResult> TakePictureAsync()
{
    var tcs = new TaskCompletionSource<CameraResult>();

    // Use the camera helper to launch the picker     
    Camera.TakePicture(
        UIApplication.SharedApplication.KeyWindow.RootViewController, 
        (imagePickerResult) => {
        // In the callback set the task result to the new picture using
        // tcs.TrySetResult(..)
    });

    return tcs.Task;
}
```

Check out the repository for the full details or the other platforms or simply pop it in your solution to access that camera easier :-)