+++
author = "admin"
categories = [".NET", "C#", "Dynamics CRM"]
date = 2014-06-26T08:29:39Z
description = ""
draft = false
slug = "dynamics-crm-20112013-re-register-plugins-key"
tags = [".NET", "C#", "Dynamics CRM"]
title = "Dynamics CRM 2011/2013 - Re-register Plugin Assembly With New Strong Name Key"

+++


All CRM plugins must be signed with a strong key in order to be deployed (both for letting Dynamics CRM be able to handle assembly naming conflicts and if you deploy it to the GAC on the server). I have come across a couple of projects where the password for the strong name key file was stuck in the head of another person – and later forgotten – making it troublesome set up a new development machine to build and deploy a solution.

For good reasons, the password cannot be retrieved, so here is a simple approach to re-registering your plugin/workflows with all existing steps and images as defined in the .crmregister file preserved.


## 1. Delete the Assembly From Server

This can be done either via. the CRM Explorer in Visual Studio or the Plugin Registration Tool. Note that this will leave your VS solution completeley intact and will not change anything in the .crmregister file.

Using the CRM Explorer in Visual Studio – connect to the organisation, choose “Plug-in Assemblies” in the menu, right click the assembly and choose “Delete Assembly”:

![deleteassembly](/images/2015/04/deleteassembly.png)

Using the Plugin Registration Tool – connect to to the organisation, right click the existing assembly and select “Unregister”:

![deleteassembly2](/images/2015/04/deleteassembly2.png)


## 2. Create the New Signed Name Key File

Right click the project and choose Properties. Go to the Signing tab and choose a new strong name key file.

![pluginkey](/images/2015/04/pluginkey-580x191.png)


## 3. Reset Id Attributes in the .crmregister File


Here comes the part where we tell the CRM server to consider this a new plugin with all the existing configuration defined in the .crmregister file.

Open the .crmregister file in the CRM Package Project:

![crmregister](/images/2015/04/crmregister.png)

Change all `Id=”{some-guid}”` attributes to `Id=”00000000-0000-0000-0000-000000000000″`. This must be done for all types of elements in the registration file (solution, plugin, step, image and so on).


## 4. Deploy Solution to CRM Server

Hit F5 and let the solution be deployed to a CRM server. This will cause everything to be registered as new. Re-open the .crmregister file and all Id attributes will contain a newly generated GUID.


