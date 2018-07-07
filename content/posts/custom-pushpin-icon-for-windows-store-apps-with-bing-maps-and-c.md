+++
author = "admin"
categories = [".NET", "C#", "Bing Maps", "Windows", "Windows Store", "XAML", "Windows 8"]
date = 2012-11-04T16:14:56Z
description = ""
draft = false
slug = "custom-pushpin-icon-for-windows-store-apps-with-bing-maps-and-c"
tags = [".NET", "C#", "Bing Maps", "Windows", "Windows Store", "XAML", "Windows 8"]
title = "Custom pushpin icon for Windows Store Apps with Bing Maps and C#"

+++


While the Bing Maps SDK is out of Release Preview and made final along with the release of Windows 8, i find that the documentation is still lacking quite a bit. Here is a couple of examples of how you can customize the pushpin control by changing the displayed image overlay, without having to implement a seperate Control.


## Custom icon for pushpin using XAML and C#

Add the following to the resources (e.g. in your XAML for the Page containing the Bing Maps Control): 

  
 Remember to set the width and height properties accordingly to match your own image.  
 Then add your pins to the Map control:
```csharp
Pushpin p = new Pushpin();
p.Name = "NewPin";
p.Style = this.Resources["PushPinStyle"] as Style;
Location locationOfPin = new Location(57.052023, 9.917122);
MapLayer.SetPositionAnchor(p, new Point(25 / 2, 39));
MapLayer.SetPosition(p, locationOfPin);
MyMap.Children.Add(p)
```
And achieve the following:
![C# bing maps pushpin custom icon windows store app](/images/2015/04/pushpin.png)

Remember to correctly anchor your pin if you do not want the actual location to be in the centre of your image (e.g. if you use a needle like pin as in the illustrated example). 


